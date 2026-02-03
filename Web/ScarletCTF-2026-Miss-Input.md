# Miss-Input - CTF Writeup

## Challenge Information

| Field | Value |
|-------|--------|
| **CTF** | ScarLet |
| **Category** | Web |
| **Difficulty** | Medium |
| **Points** | 477|
| **Challenge Link** | https://missinput.ctf.rusec.club/ |

## Description

>"IT'S A MISINPUT! YOU CALM DOWN! YOU CALM THE F DOWN!"


**Files provided:**
- `challenge.wasm` – A WebAssembly module exporting XOR-based functions (red herring)
- Client-side JavaScript (embedded in page) – Implements the real logic

**Flag format:** `^RUSEC\{.*\}$`

---

## TL;DR

The challenge performs a **client-side repeating-key XOR** on a hardcoded ciphertext and only checks if the decrypted output starts with `RUSEC{`.  
By extracting the ciphertext and XOR logic from JavaScript, then recovering the repeating key using known plaintext, we can XOR-decrypt the full flag offline.

**Vulnerability:** Client-side cryptography with weak validation  
**Approach:** Known-plaintext attack on repeating-key XOR

---

## Initial Analysis

### Web Reconnaissance

I opened the challenge website and inspected the source and DevTools.

Tools used:
- Browser DevTools (Sources, Network, Console)
- Manual code inspection

Key observations:
- No request containing the input is sent to the server.
- The input field is intentionally corrupted by JavaScript timers.
- A WASM file is loaded, but its exports are never used in validation.
- A global JS function is exposed for decryption.

The important function is exposed as:

```js
window.__tactical_support_v2 = rw;
```

## Main Analysis – JavaScript Crypto Logic
### Identifying the Real Logic

The core logic is entirely client-side:
```
rw = t => {
  try {
    let e = (hex => {
      let out = "";
      for (let i = 0; i < hex.length; i += 2)
        out += String.fromCharCode(parseInt(hex.substr(i, 2), 16));
      return out;
    })("1f6466740d2b0c070a187370017c6a757e071b686e70051b0c6e78007b611b670a704d");

    let i = ((data, key) => {
      let res = "";
      for (let n = 0; n < data.length; n++)
        res += String.fromCharCode(
          data.charCodeAt(n) ^ key.charCodeAt(n % key.length)
        );
      return res;
    })(e, t);

    if (i.startsWith("RUSEC{")) return i;
    return "ACCESS DENIED";
  } catch {
    return "SYSTEM ERROR";
  }
};
```
### Vulnerability

- The ciphertext is hardcoded.
- The encryption is repeating-key XOR.
- Only startsWith("RUSEC{") is validated.
- No server-side verification exists.
- This makes the challenge vulnerable to a known-plaintext XOR attack.

### XOR Analysis (Key Recovery)
Ciphertext (35 bytes)
``` 
0:1f  1:64  2:66  3:74  4:0d  5:2b
6:0c  7:07  8:0a  9:18 10:73 11:70
12:01 13:7c 14:6a 15:75 16:7e 17:07
18:1b 19:68 20:6e 21:70 22:05 23:1b
24:0c 25:6e 26:78 27:00 28:7b 29:61
30:1b 31:67 32:0a 33:70 34:4d
```
**Known Plaintext Structure**

We know the flag format starts with: `RUSEC{` \
Byte-by-byte alignment: 
```
| Index | Cipher | Plain  | Key    |
| ----: | ------ | ------ | ------ |
|     0 | 1f     | R (52) | 4d → M |
|     1 | 64     | U (55) | 31 → 1 |
|     2 | 66     | S (53) | 35 → 5 |
|     3 | 74     | E (45) | 31 → 1 |
|     4 | 0d     | C (43) | 4e → N |
|     5 | 2b     | { (7b) | 50 → P |

```
This reveals the repeating key prefix: `M151NP`

## Flag Reconstruction via Informed Guessing

After extracting the XOR logic, an important observation was that the decryption
function could be called **directly from the browser console**.

By testing a short candidate key, we immediately obtained a **full-length,
printable output**, even though it looked partially garbled.

### Initial Console Decryption

Using the browser console:

```js

window.__tactical_support_v2("M151NP")
DEBUG: Decrypted attempt: RUSEC{A6?)= LM_D0WVY[AKKA_M151VV?A
'RUSEC{A6?)= LM_D0WVY[AKKA_M151VV?A\x03'
```

At this point, two crucial facts were established:

The output length exactly matched the ciphertext length (35 bytes)
- The prefix RUSEC{ was valid, meaning the XOR key prefix was correct
- Human-Assisted Plaintext Normalization

### Human-Assisted Plaintext Normalization

The decrypted string contained almost-readable English, suggesting the flag
was intentionally obfuscated but meant to be human-recognizable.

By visually inspecting the output, the following substitutions were obvious:

```
RUSEC{A6?)= LM_D0WVY[AKKA_M151VV?A\x03

RUSEC{A6?_C4LM_D0WN_[AKKA_M151VV?A}
```

### Key Recovery from Guessed Plaintext

With parts of the plaintext now confidently known, we XORed them against their
corresponding ciphertext bytes to recover additional key material.

Known guessed segments:
`_C4LM_D0WN_`
`_M151`
For each known position: \
key_byte = ciphertext_byte XOR plaintext_byte



| Index | Cipher | Plain | XOR Result | Key Char |
|------:|--------|-------|------------|----------|
| 9  | 18 | _ | 47 | G |
| 10 | 73 | C | 30 | 0 |
| 11 | 70 | 4 | 44 | D |
| 12 | 01 | L | 4d | M |
| 13 | 7c | M | 31 | 1 |
| 14 | 6a | _ | 35 | 5 |
| 15 | 75 | D | 31 | 1 |
| 16 | 7e | 0 | 4e | N |
| 17 | 07 | W | 50 | P |
| 18 | 1b | N | 55 | U |
| 19 | 68 | _ | 37 | 7 |

---

#### Recovered key fragment
**G0DM151NPU7**


This fragment overlaps cleanly with the previously recovered key prefix
`M151NP`, allowing us to align and extend the repeating XOR key.

---

#### Final reconstructed repeating key
`M151NPU7_G0D`

- Just guess the `_` while XOR Same as previous steps

### Final Flag Construction 

```
data_hex = "1f6466740d2b0c070a187370017c6a757e071b686e70051b0c6e78007b611b670a704d"
data = bytes.fromhex(data_hex)

key = b"M151NPU7_G0D"

flag = bytearray()

for i in range(len(data)):
    flag.append(data[i] ^ key[i % len(key)])

print(f"Recovered flag bytes: {flag}")
print(f"Recovered flag string: {flag.decode()}")

```
## Flag 
`RUSEC{Y0U_C4LM_D0WN_175_A_M151NPU7}`

PWN by **W4RR1OR**
