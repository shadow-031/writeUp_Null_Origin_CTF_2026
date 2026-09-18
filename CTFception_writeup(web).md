# Every Archive Hides Another Archive: Solving CTFception's Tilak Vault Challenge

## 1. Challenge Overview

| Field | Detail |
|---|---|
| **Challenge Name** | Kesari Vault 1908 |
| **Platform** | [CTFception](https://ctfception.onrender.com/) |
| **Theme** | "Every archive hides another archive." |
| **Category** | Reconnaissance / Crypto / Forensics |
| **Historical Anchor** | Lokmanya Bal Gangadhar Tilak, Mandalay Jail imprisonment (1908–1914) |

The challenge presents itself as an "encrypted historical vault" transmitting on a fictional carrier frequency of **25.09 MHz**, guarded by an archivist node named `PRX02`. The flavor text repeats a recurring motif throughout the challenge:

> "Ideas that refuse to disappear cannot be silenced by time or stone."
>
> "Some stories survive because someone keeps telling them."

These aren't just atmosphere — as this writeup shows, the historical framing directly encodes the cryptographic material needed to solve the puzzle.

[Screenshot: CTFception landing page showing the PRX02 terminal banner]

## 2. Objective

The stated goal is to "investigate the platform, crack the vault, and reconstruct the lost transmission." Breaking that down into concrete technical objectives:

1. Enumerate the challenge's virtual filesystem via the web terminal.
2. Identify and retrieve the password for an encrypted ZIP vault.
3. Download and extract that vault locally.
4. Reverse-engineer a custom decoder found inside the vault.
5. Recover a "beacon key" using that decoder.
6. Feed the beacon key back into the web terminal to trigger the final `reconstruct` stage.

A key structural feature of this challenge — and the reason for this writeup's two-part structure — is that it spans **two separate environments**: the CTFception web terminal (used purely for reconnaissance and clue discovery) and a local Kali Linux machine (used for the actual archive extraction and reverse engineering). Conflating these two stages would misrepresent how the challenge is actually solved, so they're kept clearly separate below.

## 3. Initial Reconnaissance — CTFception Web Terminal

On loading the challenge, the web terminal identifies itself as follows:

```text
CTFCEPTION [STATIC KERNEL 4.2.0]

SYSTEM: ARCHIVE PROTOCOL // CTFCEPTION [STATIC KERNEL 4.2.0]
STATION: PRX02 CLANDESTINE NODE // CARRIER: 25.09 MHz // STATUS: ONLINE
```

with an interactive prompt of:

```text
prx02@tilak-archive:~$
```

[Screenshot: initial terminal banner and prompt]

Before touching any files, the sensible first move in any unfamiliar CTF terminal is to check what commands are actually supported, rather than guessing:

```text
prx02@tilak-archive:~$ help
```

This confirmed the terminal exposes a constrained but functional shell-like command set (`ls`, `cat`, `inspect`, and eventually `reconstruct`), rather than a full POSIX shell. That distinction matters — it tells us this is a purpose-built puzzle environment, not a real compromised host, so the "recon" here is about *reading challenge-provided evidence*, not exploiting a real filesystem.

## 4. Enumerating the Virtual Filesystem

Running `ls` revealed the full layout of the virtual archive:

```text
prx02@tilak-archive:~$ ls
/archive
├── records/
│   ├── kesari_1881_manifesto.txt
│   └── swaraj_declaration_1916.txt
├── transmissions/
│   ├── kesari_wire_1908.log
│   └── signal_intercept.raw
├── evidence/
│   ├── telemetry.json
│   └── surveillance_report_1897.txt
└── old/
    └── corrupted_sector_09.bak [DECOY]

/downloads
├── kesari_vault_1908.zip [ENCRYPTED VAULT]
└── extract_vault.py [HELPER TOOL]
```

A few things stand out immediately:

- `/archive/old/corrupted_sector_09.bak` is explicitly tagged `[DECOY]` — a direct signal from the challenge author not to waste time here. Historical/wire-log files under `records/` and `transmissions/` are almost certainly flavor text supporting the theme rather than functional clues, since the challenge points elsewhere for the actual mechanics.
- `/downloads/kesari_vault_1908.zip` is tagged `[ENCRYPTED VAULT]` — this is clearly the primary artifact to acquire.
- `/archive/evidence/telemetry.json` doesn't carry an explicit tag yet at this stage, but its name ("evidence," "telemetry") makes it worth inspecting early.

This directory listing is what shapes the entire rest of the solve path: identify the encrypted vault, find whatever unlocks it, then pivot to a local environment for the actual cryptographic work.

## 5. Inspecting the Encrypted Vault

Rather than blindly downloading the ZIP, I used the terminal's `inspect` command to pull metadata first — a reasonable habit before interacting with any unknown encrypted artifact, even in a sandboxed CTF:

```text
prx02@tilak-archive:~$ inspect /downloads/kesari_vault_1908.zip
```

Output:

```text
[+] FILENAME        : kesari_vault_1908.zip
[+] VIRTUAL PATH    : /downloads/kesari_vault_1908.zip
[+] CLASSIFICATION  : Downloadable Artifacts
[+] MIME TYPE       : application/zip
[+] RECORD SIZE     : 3.4 KB
[+] HISTORICAL DATE : 1908-07-22
[+] SUMMARY         : Encrypted archival package from Mandalay. Requires philosophical work passphrase.
[+] DIRECT DOWNLOAD : downloads/kesari_vault_1908.zip
[+] ARCHIVE ACCESS  : ENCRYPTED ZIP. Passphrase required.
[+] HINT            : Check /archive/evidence/telemetry.json for the philosophical work passphrase hint!
```

This is the pivotal moment of the reconnaissance phase. Two facts fall out of this output:

1. The ZIP is password-protected, and the password is tied to a **"philosophical work"** — a specific, checkable historical detail rather than something to brute-force.
2. The terminal explicitly names the next file to inspect: `telemetry.json`.

This is a good example of a CTF author using narrative metadata to *direct* the solver, rather than leaving the path ambiguous. It also confirms that brute-forcing the ZIP password is not the intended approach — the challenge wants the password *derived*, not guessed.

## 6. Reading Critical Telemetry

Following the pointer from the `inspect` output:

```text
prx02@tilak-archive:~$ cat /archive/evidence/telemetry.json
```

The relevant portions of the returned JSON:

```json
{
  "system": "CTFCEPTION ARCHIVE ENGINE",
  "version": "4.2.0-STATIC",
  "mission": "Lokmanya Tilak - Ideas That Refuse to Disappear",
  "active_nodes": [
    {
      "node_id": "PRX02-ALPHA-1908",
      "status": "ONLINE",
      "carrier_frequency": "25.09 MHz",
      "integrity_score": 87.4,
      "archivist": "PRX02",
      "quote": "Some stories survive because someone keeps telling them."
    }
  ],
  "access_key_hint": "The vault passphrase is the romanized title of Tilak's monumental philosophical work composed secretly by pencil inside Mandalay Jail between 1908 and 1914 (lowercase, single word, 11 letters).",
  "telegraph_cipher_hint": "Once inside the vault, the telegraph beacon payload requires the chronicle launch coordinate key (DDMM format: 2509).",
  "terminal_command": "reconstruct <DECODED_BEACON_KEY>"
}
```

This single file is the backbone of the entire challenge. It gives us, in order:

- **The ZIP password derivation rule** (`access_key_hint`).
- **The eventual decryption key needed inside the vault** (`telegraph_cipher_hint` — `2509`), well before we even open the ZIP.
- **The exact syntax of the final command** we'll need at the very end (`reconstruct <DECODED_BEACON_KEY>`).

In other words, `telemetry.json` is effectively a roadmap for the rest of the challenge, marked `[CRITICAL]` for good reason.

## 7. Deriving the ZIP Password

The `access_key_hint` gives a precise, checkable historical clue:

- Tilak's monumental philosophical work
- Composed secretly by pencil, in Mandalay Jail
- Written 1908–1914
- Romanized title
- Lowercase
- Single word
- Exactly 11 letters

Tilak's philosophical treatise written during his Mandalay imprisonment is well documented as *Gita Rahasya* (also transliterated *Shrimadbhagvadgita Rahasya*), his commentary on the Bhagavad Gita's philosophy of Karma Yoga.

Applying the hint's constraints:

```text
Gita Rahasya  →  remove space, lowercase  →  gitarahasya
```

Verifying the letter count:

```text
g  i  t  a  r  a  h  a  s  y  a
1  2  3  4  5  6  7  8  9  10 11
```

Exactly 11 letters — matching the hint precisely. This confirms:

```text
ZIP PASSWORD = gitarahasya
```

It's worth pausing on the design of this clue: it deliberately gives *just enough* constraints (word count, case, exact length) to let a solver confirm they've found the correct answer without needing to brute-force the archive at all. This is intentional puzzle design, not a shortcut around the challenge.

## 8. Downloading the Artifact

With the password derived, the last step inside the web terminal was retrieving the actual file. The `inspect` output earlier had already surfaced the direct download path:

```text
downloads/kesari_vault_1908.zip
```

I used the web terminal's direct download link to pull the encrypted archive down to my Kali Linux machine. This is the boundary between the two environments referenced throughout this writeup: **everything before this point happened inside the CTFception web terminal**, and **everything from this point forward happens locally**.

[Screenshot: browser download dialog for kesari_vault_1908.zip]

## 9. Local Kali Analysis

With `kesari_vault_1908.zip` now sitting locally, the first step — as with any unfamiliar file — was to confirm what it actually is rather than trusting the extension:

```bash
file kesari_vault_1908.zip
```

Output:

```text
kesari_vault_1908.zip: Zip archive data, made by v2.0, extract using at least v2.0, last modified Sep 17 2026 23:23:44, uncompressed size 2265, method=AES Encrypted
```

This confirms two things: it is genuinely a ZIP container, and it's protected with **AES encryption** rather than the weaker legacy ZipCrypto scheme. That's consistent with the terminal's earlier claim that a passphrase is required — a standard `unzip` without the right tooling likely wouldn't handle this cleanly, so `7z` (which handles AES-encrypted ZIPs natively) was the right tool for extraction.

## 10. Extracting the First Layer

Using the password derived in Section 7:

```bash
7z x -pgitarahasya kesari_vault_1908.zip -ovault1
```

The password worked immediately, confirming the historical-clue derivation was correct — no brute-forcing was needed.

To see what had actually been extracted:

```bash
find vault1 -type f -exec file {} \;
```

Output:

```text
vault1/telegraph_decoder.py: Python script, ASCII text executable
vault1/MANDALAY_DISPATCH_1908.txt: ASCII text
vault1/beacon_payload.enc: ASCII text, with no line terminators
```

## 11. Discovering the Nested Evidence

This is where the challenge's title theme — "Every archive hides another archive" — becomes literal rather than metaphorical. The vault didn't contain a flag directly; it contained:

- A **dispatch file** (more narrative/clue text)
- A **custom decoder script**
- An **encrypted payload** that the decoder is meant to process

In other words, cracking the ZIP didn't finish the challenge — it just unlocked the next puzzle layer, consistent with the challenge's framing throughout.

## 12. Reading the Mandalay Dispatch

```bash
cat vault1/MANDALAY_DISPATCH_1908.txt
```

The dispatch is written as an in-universe intelligence report describing Tilak's imprisonment and an encrypted telegraph beacon. The technically relevant lines were:

```text
Day and Month: 25th Sept -> Key: '2509'
```

and explicit instructions:

```text
1. Decode 'beacon_payload.enc' using telegraph_decoder.py with the chronicle key '2509'.
2. Enter the resulting beacon phrase into the CTFception Web Terminal using:
   reconstruct <DECODED_BEACON_KEY>
```

Two things worth noting here:

- The dispatch's `2509` key **matches** what `telemetry.json` already told us back in Section 6 (`telegraph_cipher_hint`). The challenge reinforces its own clues across both environments rather than hiding the key in only one place — a nice bit of redundant design that also makes the puzzle self-verifying.
- The `25.09 MHz` carrier frequency mentioned throughout the challenge's flavor text is thematically the same digits as `2509`, but it's the **dispatch's explicit instruction**, not the frequency by itself, that confirms `2509` is the actual cryptographic key to use. The frequency is thematic reinforcement; the dispatch is the operative instruction.

## 13. Reverse Engineering `telegraph_decoder.py`

Before running unfamiliar code against the payload, I read through the script to understand exactly what it does:

```python
#!/usr/bin/env python3

import sys
import base64

def decode_payload(payload_path, key):
    try:
        with open(payload_path, "r") as f:
            encoded_str = f.read().strip()

        data = base64.b64decode(encoded_str)

        key_bytes = key.encode("utf-8")

        decoded = bytes([
            b ^ key_bytes[i % len(key_bytes)]
            for i, b in enumerate(data)
        ])

        return decoded.decode("utf-8")

    except Exception as e:
        return f"[!] Error during decoding: {e}"

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("[!] Usage: python telegraph_decoder.py <chronicle_key>")
        print("[*] Note: The courier dispatch notes the chronicle launch date (DDMM format: 2509).")
        sys.exit(1)

    key = sys.argv[1]

    result = decode_payload("beacon_payload.enc", key)

    print("=" * 60)
    print(f"[+] Decoded Telegraph Beacon: {result}")
    print(f"[+] Terminal Command: reconstruct {result}")
    print("=" * 60)
```

Walking through the logic:

1. It opens `beacon_payload.enc` (hardcoded — not passed as a CLI argument).
2. Strips leading/trailing whitespace from the file contents.
3. Base64-decodes the string into raw bytes.
4. Converts the supplied key argument into bytes.
5. XORs every byte of the decoded data against the key bytes, cycling the key with modulo indexing (`i % len(key_bytes)`) whenever the data is longer than the key.
6. Decodes the resulting byte sequence back into a UTF-8 string.
7. Prints both the decoded beacon phrase and the exact `reconstruct` command to run next.

The script also expects **exactly one command-line argument** — the key — via `sys.argv[1]`.

## 14. Understanding Base64 + Repeating-Key XOR

Looking at the contents of the encrypted payload:

```bash
cat vault1/beacon_payload.enc
```

```text
fnp7dHN7aXhtcnFtenRvenpnf3d7dnx8bQQJCQo=
```

The character set (alphanumerics plus `+`, `/`) and the trailing `=` padding are classic Base64 hallmarks, which the decoder script's own `base64.b64decode()` call confirms.

The core cryptographic operation is a **repeating-key XOR cipher**, one of the oldest and most common lightweight obfuscation techniques used in CTFs:

```text
cipher_byte[i] = plaintext_byte[i] XOR key_byte[i mod key_length]
```

Because XOR is its own inverse (`a XOR b XOR b = a`), decryption is identical to encryption — applying the same key via XOR again recovers the original plaintext:

```text
plaintext_byte[i] = cipher_byte[i] XOR key_byte[i mod key_length]
```

This is precisely why the *same* `telegraph_decoder.py` script can be used to reverse the encoding: it doesn't need a separate "decrypt mode," since XOR-with-key is symmetric. It's a fine teaching example of why repeating-key XOR provides no real security against a known-key or known-plaintext scenario — it's included here purely as a puzzle mechanic, not as a real-world cryptographic recommendation.

## 15. Debugging the Incorrect Decoder Invocation

My first attempt at running the decoder was based on a natural but incorrect assumption — that the script takes both a filename and a key as arguments:

```bash
python3 telegraph_decoder.py beacon_payload.enc 2509
```

This produced an error rather than a valid beacon phrase.

Looking back at the source code explains exactly why:

```python
key = sys.argv[1]
```

`sys.argv[1]` is the **first** argument after the script name — so in the command above, the script interpreted:

```text
sys.argv[1] = "beacon_payload.enc"
```

as the *key*, not as a filename. Meanwhile, `2509` (intended to be the key) was simply ignored, since the script never reads `sys.argv[2]`. Additionally, the payload filename is hardcoded inside `decode_payload("beacon_payload.enc", key)`, so it was never meant to be supplied on the command line at all.

The fix was to drop the filename argument entirely and pass only the key:

```bash
python3 telegraph_decoder.py 2509
```

This is a useful debugging lesson in its own right: when a script errors out or produces garbage, checking `sys.argv` usage against the actual invocation is often faster than guessing at input formats.

## 16. Recovering the Beacon

Running the corrected command from inside the `vault1` directory:

```bash
cd vault1
python3 telegraph_decoder.py 2509
```

produced:

```text
============================================================
[+] Decoded Telegraph Beacon: LOKMANYA_GATHA_CHRONICLE_1908
[+] Terminal Command: reconstruct LOKMANYA_GATHA_CHRONICLE_1908
============================================================
```

The recovered beacon key is:

```text
LOKMANYA_GATHA_CHRONICLE_1908
```

This is the confirmed output of the second cryptographic layer, obtained by base64-decoding `beacon_payload.enc` and reversing the repeating-key XOR with the `2509` key extracted from the Mandalay dispatch.

## 17. Final Web Terminal Reconstruction

With the beacon key in hand, the last step (per both `telemetry.json` in Section 6 and the dispatch in Section 12) is to return to the CTFception web terminal and run:

```text
reconstruct LOKMANYA_GATHA_CHRONICLE_1908
```

**Note on scope:** the output of this final `reconstruct` command was not captured during this solve session, so it isn't included here. `LOKMANYA_GATHA_CHRONICLE_1908` is confirmed as the correctly recovered beacon key based on all evidence gathered above, and `reconstruct <key>` is confirmed (from `telemetry.json`) as the intended next command — but the resulting transmission/flag text itself is not claimed or fabricated in this writeup. Anyone reproducing this solve should expect this command to reveal the final stage of the challenge.

[Screenshot: web terminal after running the `reconstruct` command]

## 18. Complete Solve Chain

```text
CTFception Web Terminal
        │
        ├── ls
        │
        ├── inspect encrypted ZIP
        │
        └── cat telemetry.json
                  │
                  ▼
          Password clue found
                  │
                  ▼
             gitarahasya
                  │
                  ▼
      Download kesari_vault_1908.zip
                  │
                  ▼
             Kali Linux
                  │
                  ▼
       AES-encrypted ZIP archive
                  │
                  ▼
       7z extraction with password
                  │
                  ▼
              vault1/
                  │
        ┌─────────┼──────────────┐
        ▼         ▼              ▼
     dispatch   decoder.py   beacon_payload.enc
        │
        ▼
       Key = 2509
                  │
                  ▼
       Base64 decode + XOR
                  │
                  ▼
LOKMANYA_GATHA_CHRONICLE_1908
                  │
                  ▼
Return to CTFception Web Terminal
                  │
                  ▼
reconstruct LOKMANYA_GATHA_CHRONICLE_1908
                  │
                  ▼
             Final stage
```

## 19. Technical Lessons

A few generalizable takeaways from this challenge:

- **Read metadata before acting on a file.** The `inspect` command and `file` utility both provided critical context (encryption type, hints, MIME type) before any destructive or blind action was taken.
- **Trust explicit clues over thematic ones.** The `25.09 MHz` frequency and the `2509` key look related, but it was the dispatch's *explicit instruction* that confirmed the correct cryptographic key — thematic consistency is a design flourish, not a substitute for a stated instruction.
- **Read source before executing.** Reverse engineering `telegraph_decoder.py` before running it avoided wasted attempts and explained the eventual `sys.argv` bug immediately once it occurred.
- **Repeating-key XOR is trivially reversible** given the key, which is precisely why it's suitable for a puzzle layer but never appropriate as real-world encryption.
- **Argument-parsing bugs are common and diagnosable.** A quick look at how a script indexes `sys.argv` resolves most "why is this failing" moments faster than trial and error.

## 20. Artifacts and Indicators

| Artifact | Value |
|---|---|
| Encrypted vault | `kesari_vault_1908.zip` |
| ZIP password | `gitarahasya` |
| Encryption method | AES (per `file` output) |
| Nested decoder | `telegraph_decoder.py` |
| Nested dispatch | `MANDALAY_DISPATCH_1908.txt` |
| Encrypted beacon payload | `beacon_payload.enc` |
| Cipher key | `2509` |
| Cipher scheme | Base64 → repeating-key XOR |
| Recovered beacon key | `LOKMANYA_GATHA_CHRONICLE_1908` |
| Final command | `reconstruct LOKMANYA_GATHA_CHRONICLE_1908` (output not captured) |

## 21. TL;DR

```bash
# Local extraction and decoding
7z x -pgitarahasya kesari_vault_1908.zip -ovault1
cd vault1
python3 telegraph_decoder.py 2509
```

```text
# Final step, back in the CTFception web terminal
reconstruct LOKMANYA_GATHA_CHRONICLE_1908
```

*(The output of the final `reconstruct` command was not captured during this solve and is not claimed here.)*
FLAG: NullOrigin{L0km4ny4_G4th4_25S3pt_Sw4r4jy4_PRX02_}
## 22. Conclusion

This challenge is a well-constructed example of layered puzzle design: a web-based reconnaissance stage that discloses just enough information to derive credentials without brute-forcing, paired with a local reverse-engineering stage that requires actually reading and understanding a small custom script rather than treating it as a black box. The historical framing around Lokmanya Tilak's Mandalay imprisonment isn't incidental flavor — it's load-bearing, since the ZIP password itself is derived directly from that history. The "every archive hides another archive" theme plays out literally at every layer: an encrypted ZIP hides a decoder and a dispatch, which hide an encrypted payload, which hides a beacon key, which unlocks a final stage in a completely different environment.

The solve documented here is confirmed through `LOKMANYA_GATHA_CHRONICLE_1908`; the final transmission behind the `reconstruct` command remains the next — and, as far as the available evidence shows, final — step of the challenge.
