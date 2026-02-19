# CHIP-8 Emulator - Writeup

| Field | Value |
|-------|--------|
| **CTF** | 0xfun CTF 2026 |
| **Category** | Reverse Engineering |

---

## Challenge Description

> Ever wondered how emulators tick under the hood? I built one — the simplest of all, a CHIP-8 emulator. Alongside it, I’ve dropped 100+ games and programs for you to play… or are they really only for playing?
> Somewhere deep in this virtual silicon, a flaw hides. Uncover it, and in just quad cycles, the flag is yours. Miss it, and you’ll be stuck endlessly.

We are given a Linux ELF binary `chip8Emulator`.

---

## Initial Analysis

```
$ file chip8Emulator
ELF 64-bit LSB pie executable, dynamically linked, not stripped
```

Running help:

```
./chip8Emulator --help
```

Shows:

```
-r, --romFileName   Set the ROM file path to be used
```

So the emulator loads and executes CHIP-8 ROM files.

This means:

> Our only input is a ROM → this is a VM / emulator challenge.

---

## Looking for the Flag

While reversing, a suspicious global string appears:

```
SMr85LT/QH8WBgB7FAHDJ+RDYEOzmc+8Hq+2HKyaEbwR0DN9BaUFpMgRyi3p9HBH...
```

Inside a function named:

```
void __static_initialization_and_destruction_0(void)
```
##### the the string was stored in a global array called `_3nc` so after checking function calls for `_3nc` we discovered one new function `Cpu::superChipRendrer()`

The function does:

1. Base64 decode the string
2. AES-256-CBC decrypt
3. Write result to file

```cpp
ofstream("flag.txt") << decrypted_data;
```

So the flag already exists inside the binary — encrypted.

Therefore the goal becomes:

> Find how `superChipRendrer()` gets executed.

---

## Fake Crypto Dependency

The key appears to depend on CPU state:

```cpp
chat_toStr(local_268, this, local_248);
```

But the function actually does nothing:

```cpp
string * chat_toStr(string *out, ..., string *in)
{
    std::string(out, in);
    return out;
}
```

So:

```
AES key = constant hardcoded string
```

Meaning:

> We don’t need to reverse crypto at all
> We just need to make the emulator call the function

---

## Finding the Backdoor Opcode

Searching cross-references to `superChipRendrer` reveals:

```cpp
void Cpu::decode_F_instruction()
{
    bVar1 = opcode_low_byte;

    if (bVar1 == 0xff) {
        superChipRendrer(this);
    }
}
```

This means the emulator added a **hidden CHIP-8 instruction**:

```
FXFF
```

This opcode does not exist in real CHIP-8.

So:

> Executing any `FXFF` instruction instantly decrypts and writes the flag.

---

## Building the Exploit ROM

We just need a CHIP-8 ROM containing the opcode.

Minimal safe ROM:

| Instruction | Meaning          |
| ----------- | ---------------- |
| 00E0        | Clear screen     |
| F0FF        | Trigger backdoor |
| 1200        | Infinite loop    |

Hex:

```
00 E0 F0 FF 12 00
```

Create ROM:

```
printf '\x00\xE0\xF0\xFF\x12\x00' > exploit.ch8
```

Run emulator:

```
./chip8Emulator -r exploit.ch8
```

The emulator executes the hidden instruction and creates:

```
flag.txt
```

Read the flag:

```
cat flag.txt
0xfunCTF2025{N0w_y0u_h4v3_clear_1dea_H0w_3mulators_WoRK}
```

---

## Why This Works

The challenge is not about breaking AES.

Instead it demonstrates a common reversing lesson:

> If software decrypts the flag for you — trigger the code path, don’t break the crypto.

The emulator contained a developer backdoor instruction:

```
FXFF → privileged behavior
```

So this is effectively a **VM escape via undocumented opcode**.

---

## Final Solution

**Vulnerability:** Hidden CHIP-8 instruction
**Backdoor Opcode:** `FXFF`
**Exploit ROM:** `00E0F0FF1200`

## Flag
`0xfunCTF2025{N0w_y0u_h4v3_clear_1dea_H0w_3mulators_WoRK}`

PWN by **W4RR1OR**
