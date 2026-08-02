# Encryption & Decryption — Mobile API Security Notes

> Context: securing sensitive fields in API responses for a mobile app (Android + iOS). Flow: OTP login → JWT issued → a session key is exchanged → RSA-2048 is used to encrypt/decrypt sensitive payload fields, decrypted client-side inside the Axios response interceptor.

---

## 1. A common mobile secure-session architecture

**Flow:**
1. User completes **OTP verification** at login.
2. Backend issues a **JWT** for the session.
3. Alongside the JWT, a **session key** is generated and "locked" (encrypted) for exchange.
4. **Key placement:**
   - One key half lives in the **device's secure storage** — Android **Keystore** / iOS **Keychain** (ideally backed by hardware — StrongBox / Secure Enclave).
   - The other half lives in the **backend**, ideally behind an HSM/KMS-backed secret store rather than a plain DB column/env var.
5. On every subsequent API call, sensitive fields come back **encrypted**; the mobile app decrypts them **in the Axios response interceptor** before the data reaches the UI/state layer.

**Algorithm:** RSA-2048, typically with **OAEP padding** and **SHA-256** as the hash inside OAEP (written as `RSA-2048/OAEP-SHA256`) — the modern, safe default for RSA. Older systems sometimes use `PKCS#1 v1.5` padding instead, which is weaker (see padding-oracle note below).

**Why this is (usually) a hybrid scheme under the hood:** RSA can only encrypt data smaller than its key size (roughly key-size-minus-padding, e.g. ~190 bytes for RSA-2048 with OAEP). If the sensitive payload is more than a few bytes, pure RSA can't carry it directly — so what typically happens is:
- A random **symmetric key** (AES) is generated per session/request.
- The **payload** is encrypted with **AES** (fast, no size limit).
- The **AES key itself** is encrypted with **RSA** (slow, but the key is tiny) using the public key.
- The receiver decrypts the RSA-wrapped AES key with their private key, then uses that AES key to decrypt the actual payload.

This pattern is called **hybrid encryption** — RSA/ECC for *key exchange*, AES for *bulk data*. It's how TLS, PGP, and basically every real-world "asymmetric encryption" system works, because pure asymmetric encryption of bulk data is both slow and size-limited.

---

## 2. Symmetric vs Asymmetric — the core concept

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | One shared key, used for both encrypt & decrypt | Key **pair** — public key encrypts, private key decrypts |
| Speed | Fast (good for bulk data) | Slow (100-1000x slower), size-limited |
| Key distribution problem | Hard — both sides need the *same secret* without ever transmitting it in the open | Easy — public key can be shared openly, only the private key must stay secret |
| Examples | AES, ChaCha20, 3DES (legacy) | RSA, ECC (ECDH/ECDSA), Diffie-Hellman |
| Typical use | Encrypting the actual data | Encrypting/exchanging the symmetric key, or signing |

**Rule of thumb:** asymmetric crypto solves the *key distribution* problem; symmetric crypto does the *heavy lifting* on data. Real systems (TLS, our API scheme above) use both together — that combo is the hybrid model.

---

## 3. Algorithms present in the market (reference)

### Symmetric (data encryption)
- **AES (Advanced Encryption Standard)** — the industry standard. Key sizes: 128 / 192 / **256**-bit.
  - **Modes matter more than key size:**
    - `AES-GCM` — **authenticated encryption** (gives you confidentiality + integrity in one step, detects tampering). **Preferred mode today.**
    - `AES-CBC` — needs a separate MAC (HMAC) for integrity, or it's vulnerable to padding-oracle/bit-flipping attacks. Common in older mobile SDKs.
    - `AES-CTR` — stream-like, fast, also needs a separate MAC.
  - If you see raw `AES-CBC` with no HMAC anywhere in the mobile code, that's a red flag worth raising.
- **ChaCha20-Poly1305** — modern alternative to AES-GCM, faster in pure software (no hardware AES instructions), used heavily on mobile/low-power devices. Google prefers it in some Android TLS stacks for this reason.
- **3DES / DES** — legacy, deprecated. Shouldn't appear in new systems.

### Asymmetric (key exchange / signing)
- **RSA** — key sizes 1024 (broken/deprecated), 2048 (current minimum), 3072/4096 (higher security, slower). Needs a **padding scheme**:
  - `PKCS#1 v1.5` — older, vulnerable to padding-oracle attacks (Bleichenbacher). Avoid for new designs.
  - `OAEP` (Optimal Asymmetric Encryption Padding) — modern, safe default. Usually paired with a hash like SHA-256 → written as "RSA-OAEP-SHA256".
- **ECC (Elliptic Curve Cryptography)** — same security as RSA with much smaller keys (256-bit ECC ≈ 3072-bit RSA). Faster, less battery/CPU on mobile. This is where the industry has been moving.
  - **ECDH** — key *exchange* (agree on a shared secret without sending it).
  - **ECDSA** — *signing* (verify authenticity/integrity, doesn't encrypt).
  - **Ed25519 / X25519** — modern curve, fast, side-channel resistant, used by Signal, WhatsApp, WireGuard.
- **Diffie-Hellman (DH)** — the original key-exchange algorithm; ECDH is its elliptic-curve evolution.

### Hashing (one-way — not encryption, no decryption exists)
- **SHA-256 / SHA-512** — general-purpose hashing, integrity checks, part of signatures.
- **MD5 / SHA-1** — broken/deprecated, avoid.
- **bcrypt / scrypt / Argon2** — purpose-built for **password hashing** (slow by design, salted, resistant to GPU brute-force). Never use plain SHA-256 for passwords.
- **HMAC** — hash + secret key, used to verify a message wasn't tampered with (message authentication code).

### Digital signatures (authenticity, not confidentiality)
- **RSA-PSS**, **ECDSA**, **EdDSA (Ed25519)** — prove *who* sent something and that it wasn't altered. Different goal from encryption: encryption = "no one else can read it," signing = "you can prove I sent it and it's unmodified."

---

## 4. Where the keys physically live (mobile secure storage)

- **Android Keystore** — private keys can be generated *inside* hardware (StrongBox/TEE) so the raw key material never enters app memory or gets extracted, even on a rooted device (best effort — root can still sometimes intercept usage, not extraction).
- **iOS Keychain / Secure Enclave** — same idea; Secure Enclave-backed keys are non-exportable.
- **Backend** — should sit behind an **HSM or cloud KMS** (AWS KMS, GCP Cloud KMS, HashiCorp Vault) rather than a raw private key file/env var/DB row.

**Why this matters:** the entire security model collapses if a private key is extractable from either side. Hardware-backed storage is *why* RSA/ECC private keys go in Keychain/Keystore instead of, say, SharedPreferences or NSUserDefaults.

---

## 5. Key management concepts worth knowing

- **Key rotation** — session keys should have a lifespan tied to (or shorter than) the JWT's expiry. If the JWT refreshes, does the session key rotate too? Worth verifying — a session key that outlives many JWT refreshes widens the exposure window.
- **Perfect Forward Secrecy (PFS)** — using a fresh key per session (rather than one static long-lived key-pair for everyone) means a compromise of one session's key doesn't expose past/future sessions. This is presumably the point of generating a *per-login* session key rather than reusing one global key pair.
- **Replay protection** — encryption alone doesn't stop an attacker from **replaying** a captured encrypted request. Need a nonce, timestamp, or request-signing layer alongside encryption if that's a threat in scope.
- **Padding-oracle attacks** — if RSA is using PKCS#1 v1.5, or AES is using CBC without an HMAC, subtly different error responses for "bad padding" vs "bad data" can leak the plaintext byte-by-byte over many requests. Use OAEP + GCM to make this class of bug structurally impossible.

---

## 6. Transport (TLS) vs payload encryption — why do both?

HTTPS/TLS already encrypts everything in transit, so why *also* encrypt specific fields in the JSON body? Legitimate reasons:
- **Defense in depth** — if TLS is terminated early (load balancer, API gateway, CDN, corporate proxy, or a misconfigured cert on a rooted device bypassing pinning), the payload is still protected.
- **Data at rest in logs** — request/response logging (Sentry, CloudWatch, access logs) captures the JSON body. If sensitive fields are pre-encrypted, they don't leak into logs in plaintext.
- **Compliance** — some data classes (PII, financial, health) require encryption *at the application layer*, not just in transit, for audit purposes.
- **MITM on compromised devices** — a rooted/jailbroken device with a user-installed root CA can sometimes intercept TLS; app-layer encryption is one more layer that CA trick alone doesn't break.

This is why "we already have HTTPS, why encrypt the payload too?" is a fair question — the answer is defense-in-depth + logs + compliance, not "TLS is insufficient by itself."

---

## 7. General checklist for this kind of scheme

- [ ] RSA key size 2048 minimum, with **OAEP** padding (not PKCS#1 v1.5) and SHA-256 as the OAEP hash.
- [ ] The "session key" should be a symmetric (AES) key wrapped by RSA — not RSA encrypting the bulk payload directly (size + performance problem otherwise).
- [ ] AES mode should be **GCM** (authenticated) not bare CBC/ECB.
- [ ] Never **ECB mode** — it leaks patterns in the plaintext (the classic "ECB penguin" image example). Easy mistake if a library defaults to it.
- [ ] Backend private key should live in an HSM/KMS, not a plain env var / DB column.
- [ ] Session key lifecycle: rotate on JWT refresh, invalidate on logout.
- [ ] Replay protection: nonce/timestamp on encrypted requests, so a captured encrypted blob can't just be replayed.
- [ ] Certificate/public-key pinning on the mobile app, so the public key used to encrypt/verify can't be swapped by a MITM.
- [ ] Decrypted plaintext should never get logged (Axios interceptor logs, crash reporters like Sentry/Crashlytics can accidentally capture decrypted response bodies).
- [ ] Root/jailbreak detection — Keystore/Keychain guarantees weaken on compromised devices.

---

## 8. Session isolation — every user gets their own key pair

This scheme is **per-user, per-session**, not one shared key for the whole system:
- Each login generates a **fresh key pair (or session key) unique to that user's session** — user A's device holds a private key that only pairs with the encryption done for user A's session, and the backend holds the matching counterpart for that same session.
- One user's private key **cannot decrypt another user's data** — there's no single master key that unlocks everyone's payloads. A leak of one device's or one session's key only exposes that one session, not the whole user base.
- This is the same idea as **Perfect Forward Secrecy** above: because keys are minted per session instead of reused globally, compromise is contained to that one session instead of cascading to every user.

---

## Interview one-liner

> "We use a hybrid encryption scheme: after OTP + JWT login, a per-session symmetric key is issued, wrapped with RSA-OAEP so only the holder of the matching private key (one half in the device's Keystore/Keychain, one half in the backend's KMS) can unwrap it. Sensitive response fields are then encrypted with that session key and decrypted client-side in the Axios response interceptor. It's defense-in-depth on top of TLS — protecting the data in logs, at rest, and against MITM even if transport security is somehow bypassed."
