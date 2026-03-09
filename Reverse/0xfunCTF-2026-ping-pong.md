# pingpong - Writeup

| Field | Value |
|-------|--------|
| **CTF** | 0xfun CTF 2026 |
| **Category** | Reverse Engineering |

> *“ONLY attempt if you love table tennis...”*

This is a small reversing challenge written in Rust.
The binary pretends to implement a network ping-pong protocol, but the goal is actually to recover a flag hidden inside an encrypted hex buffer.

---

## 1. Finding the encrypted data

In `main` the program calls:

```
FUN_00118950(&local_430,0x10acea,0x4a);
```

If decoding fails it prints:

```
"Invalid hex string."
```

So the data at address `0x10acea` is a **hex-encoded byte array**.

From the disassembly:

```
30 31 34 39 35 34 35 62 35 66 34 62 35 64 31 65
35 63 35 34 35 64 31 61 35 35 30 33 36 63 35 37
30 30 34 30 34 62 34 36 35 30 35 64 34 32 36 65
30 32 30 30 31 62 34 39 30 39 30 33 30 39 35 37
34 31 34 61 37 62 37 61 34 38
```

Hex string:

```
0149545b5f4b5d1e5c545d1a55036c5700404b46505d426e02001b4909030957414a7b7a48
```

---

## 2. Decode the hex

Python:

```python
hex_blob = "0149545b5f4b5d1e5c545d1a55036c5700404b46505d426e02001b4909030957414a7b7a48"
data = bytes.fromhex(hex_blob)
print(data)
```

Output is unreadable → encrypted.

So the buffer must be decrypted later.

---

## 3. Understanding the program logic

The binary contains interesting strings:

```
"112.105.110.103"
"112.111.110.103"
"Now you're pinging the pong!"
"IP Blocked!!!"
```

Convert those numbers:

| Numbers         | ASCII |
| --------------- | ----- |
| 112.105.110.103 | ping  |
| 112.111.110.103 | pong  |

So the program simulates a **ping ↔ pong conversation**.

---

## 4. The important decryption loop

In the decompiled code:

```
uVar10 = uVar16 % uVar4;
local_430[i] ^= local_528[uVar10];
```

This means:

> Each byte of the encrypted buffer is XORed with a repeating keystream.

Where does `local_528` come from?

Earlier the program builds it from the text responses of:

```
"112.105.110.103"
"112.111.110.103"
```

So the keystream is the ASCII text:

```
112.105.110.103112.111.110.103
(repeating)
```

NOT `"pingpong"` — but the decimal representation of it.

---

## 5. Recreate the keystream and decrypt

Python solver:

```python
hex_blob = "0149545b5f4b5d1e5c545d1a55036c5700404b46505d426e02001b4909030957414a7b7a48"
data = bytes.fromhex(hex_blob)

# reconstructed keystream
key = (b"112.105.110.103" + b"112.111.110.103")

out = bytearray()

for i in range(len(data)):
    out.append(data[i] ^ key[i % len(key)])

print(out.decode())
```

---

## 6. Flag

```
0xfun{h0mem4d3_f1rewall_305x908fsdJJ}
```

---

## Why this works

The program pretends to do networking:

* ping → send
* pong → response
* IP blocked → wrong packets

But actually:

| Theme        | Real meaning        |
| ------------ | ------------------- |
| ping/pong    | keystream generator |
| socket reads | fake complexity     |
| XOR loop     | actual crypto       |
| hex blob     | encrypted flag      |

So the challenge is simply:

> Recover XOR keystream from the “ping pong” text protocol.

---

## Takeaway

This is a classic beginner reversing trick:

1. Hide data as hex
2. Obfuscate XOR key using fake networking logic
3. Player reconstructs keystream statically

No need to emulate sockets  just read the strings carefully.

---

## Flag
`0xfun{h0mem4d3_f1rewall_305x908fsdJJ}`

PWN by **W4RR1OR**