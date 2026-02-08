# Epstein Files – Forensics Writeup

| Field | Value |
|-------|--------|
| **CTF** | Pragyan CTF 2026 |
| **Category** | Forensics |


### Description
>You are provided with a PDF file related to an ongoing investigation. The document appears complete, but not everything is as it seems. Analyze the file carefully and recover the hidden flag.

**Flag format:** `pctf{...}`

---

## Solution Overview
This challenge hides the flag behind multiple layers:
1. A hidden PDF comment containing hex data
2. XOR decoding using a hidden key
3. GPG decryption of data appended after EOF
4. A final mixed ROT decoding (ROT13 for letters + ROT5 for digits)

---

## Step 1: Initial Enumeration
Basic inspection of the PDF using common forensic commands (`file`, `pdfinfo`, `strings`) did not reveal anything suspicious at first. The document appeared to be a normal multi-page PDF containing Epstein’s contact list.

---

## Step 2: Finding Suspicious Hidden Data 
Running `strings` with keyword searches revealed a suspicious hidden PDF comment:

```bash
strings contacts.pdf | grep -i "%"
........
n3/ACc%
% /Hidden (3e373f283d312d25222332362c3d2e292322)
%%EOF
```
PDF comments (lines starting with %) are ignored by renderers, making this an ideal hiding spot. The value inside parentheses appears to be hex-encoded data.

## Step 3: Discovering the XOR Key

Further inspection of the PDF revealed hidden text on one of the pages. Two strings were drawn in black text and then covered by a near-black rectangle, making them invisible in normal viewing.

**The hidden text revealed:**
`XOR_KEY
JEFFREY`
This confirms that the XOR key is JEFFREY.

## Step 4: XOR Decoding the Hidden Hex

The hex string from the PDF comment is XOR-decoded using the key JEFFREY.
```
hex_str = "3e373f283d312d25222332362c3d2e292322"
data = bytes.fromhex(hex_str)
key = b"jeffrey"
decoded = bytes([data[i] ^ key[i % len(key)] for i in range(len(data))])
print(decoded.decode())
```

- Output:  `trynottogetdiddled`

- You thought this is the flag hahahaha i tried submiting but incorrect .So this is clearly not the flag, so it should be an password for another hidden file or something...

## Step 5: Extracting Hidden Files

- tried using normal tool like `pdfdetach` but nothing was embeded inside so i was confused what need to be done further  
- Inspecting the PDF with xxd reveals extra data appended after the %%EOF marker.

- grep -abo "%%EOF" contacts.pdf

- Using the offset and skipping the EOF marker, the appended data is extracted:

- `dd if=contacts.pdf of=after_eof_raw.bin bs=1 skip=13984076 status=none`

- Checking the extracted file:
```
 $file after_eof_raw.bin
Output: PGP symmetric key encrypted data - AES256
```
## Step 6: PGP Decryption

- The appended data is decrypted using the previously recovered passphrase.

gpg --decrypt after_eof_raw.bin

Output:`cpgs{96a2_a5_j9l_u8_0h6p6q8}`

This resembles a flag, but the format is incorrect. we have to struggle more :(( 

## Step 7: Final ROT Decoding

The decrypted output requires a combined ROT transformation:
- ROT13 for letters
- ROT5 for digits

Symbols and underscores remain unchanged
```
decrypted = "cpgs{96a2_a5_j9l_u8_0h6p6q8}"
result = []
for c in decrypted:
    if c.isalpha():
        base = ord('a')
        result.append(chr((ord(c) - base + 13) % 26 + base))
    elif c.isdigit():
        result.append(str((int(c) + 5) % 10))
    else:
        result.append(c)
print("".join(result))
```

## Flag 
`pctf{41n7_n0_w4y_h3_5u1c1d3}`

PWN by **W4RR1OR**