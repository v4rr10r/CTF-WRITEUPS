# Plumbing – Forensics Writeup

| Field | Value |
|-------|--------|
| **CTF** | Pragyan CTF 2026 |
| **Category** | Forensics |

## Challenge Description
> We found a Docker image that was already built and shipped.  
> Something sensitive might have slipped through during build time, but the final container looks clean??  
> Analyze the image and recover what was lost.  
> **Flag format:** `p_ctf{...}`

---

## Initial Analysis

The provided files follow the **OCI image layout**:
- `index.json`
- `manifest.json`
- `blobs/sha256/*`

At first glance, the extracted filesystem layers appear clean:
- Debug files are removed
- Output is overwritten
- Application code is replaced with a stub

This suggests the secret **does not exist in the final filesystem**.

---

## Key Insight – Docker Plumbing

The challenge title **“Plumbing”** hints at:
- stdin / stdout
- pipes
- Docker build internals
- metadata rather than runtime artifacts

In Docker/OCI images, **build-time commands are permanently stored** in the image **config history**, even if files are later deleted.

Therefore, the real target is the **image config blob**, not the filesystem layers.

---

## Inspecting the Image Config

From `manifest.json`:

```json
"Config": "blobs/sha256/b3f4caf17486575f3b37d7e701075fe537fe7c9473f38ce1d19d769ea393913d"
```
- Inspecting it:

jq . blobs/sha256/b3f4caf17486575f3b37d7e701075fe537fe7c9473f38ce1d19d769ea393913d

### Build History Leak

Inside the history field:
```
{
  "created_by": "RUN /bin/sh -c echo \"p_ctf{d0ck3r_l34k5_p1p3l1n35}X|O\" | python3 process.py # buildkit"
}
```
## What happened:

-  During build time, a sensitive value was piped via stdin

-    It was never written to disk

     -  Later layers attempted cleanup:

     - /tmp/state_round7.bin removed

     - output.bin overwritten

     - script replaced

- However, Docker history is immutable

- The leaked value survives only in the build metadata.

## Flag
`p_ctf{d0ck3r_l34k5_p1p3l1n35}`

## Takeaway
- Docker images may look clean at runtime

- Build-time secrets can leak via image metadata

- Deleting files does not remove history

- Always inspect the OCI config and history when doing container forensics
