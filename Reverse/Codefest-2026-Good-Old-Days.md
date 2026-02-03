# Good Old Days –  CTF Writeup

| Field | Value |
|-------|--------|
| **CTF** | Codefest 2026 |
| **Category** | Reverse Engineering |


## Challenge Overview
The challenge provides a Flash (`.swf`) game with the description:

> *No one can beat me at this game.*

Instead of playing the game, the goal is to reverse-engineer the SWF and recover the hidden flag.

---

## Step 1: Decompiling the SWF

- you can try strings on the file  no flag in strings :(

Since Flash games contain client-side logic, the first step is to decompile the SWF.

Tool used:
- **JPEXS Free Flash Decompiler (FFDec)**
```
https://github.com/jindrapetrik/jpexs-decompiler
```

After opening There are many folders but the file and exporting scripts, a class named **`UserData.as`** 

> Wasted like 1 hour finding this file lol

 stands out. This file is large (~1600 lines), but most of it handles normal gameplay and save data.

So instead of reading everything, we search for **string building logic**.

---

## Step 2: Finding the Suspicious Code
Inside `UserData.as`, the following code appears in the `loadData()` function:

```actionscript
var studioTag:String = "";
var i:int = 0;

while (i < levelDifficulty.length)
{
    studioTag += String.fromCharCode(
        levelDifficulty[i] ^ bestTimes[i % 12]
    );
    i++;
}

trace(studioTag);
```
- This is the important part.

### Observations:

`String.fromCharCode() is used → text is being constructed ^ is the XOR operator`

`bestTimes[i % 12]` means a repeating key of length 12

This clearly indicates a repeating-key XOR cipher.

## Step 3: Understanding the Data

Two arrays are involved:

**Encrypted data**
```
 levelDifficulty = [
  104,14,96,89,43,74,38,72,75,25,21,51,
  18,81,96,120,41,75,17,120,76,18,99,4,
  71,45,72,80,41,75,10,120,76,41,103,124,
  82,84,49,9,120,26,96,65
];
```
 **XOR Key**

`bestTimes = [ ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ? ]`
- The key is not directly visible, but its length (12) is known.

## Step 4: Known-Plaintext Attack ( Hardest to think )

CTF flags usually follow a known format.
Here, the flag is known to start with:`CodefestCTF{`

this part is like very guessy but its one of the way to try as flag inital length and key array length matches so why not to try 

XOR has a useful property:
`cipher ^ plaintext = key`

So for the first 12 characters:

`bestTimes[i] = levelDifficulty[i] ^ ord("CodefestCTF{"[i])`

- This allows us to fully recover the XOR key

## Step 5 : Python Solver Script
```python
levelDifficulty = [
    104,14,96,89,43,74,38,72,75,25,21,51,
    18,81,96,120,41,75,17,120,76,18,99,4,
    71,45,72,80,41,75,10,120,76,41,103,124,
    82,84,49,9,120,26,96,65
]

# known plaintext prefix
known = "CodefestCTF{"

# recover key
bestTimes = [
    levelDifficulty[i] ^ ord(known[i])
    for i in range(len(known))
]

# decode full flag
flag = ""
for i in range(len(levelDifficulty)):
    flag += chr(levelDifficulty[i] ^ bestTimes[i % 12])

print("Key:", bestTimes)
print("Flag:", flag)

```
### OUTPUT

```
Key: [43, 97, 4, 60, 77, 47, 85, 60, 8, 77, 83, 72]
Flag: CodefestCTF{90dDddDDD_0LlLLldd_DDd44y555555}
```

## Flag
`CodefestCTF{90dDddDDD_0LlLLldd_DDd44y555555}`

PWN by **W4RR1OR**