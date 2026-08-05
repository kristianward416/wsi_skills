# Cryptography Implementation Guide

This reference backs the Information Disclosure mitigations in the senior-security skill's STRIDE workflow and the encryption-at-rest/in-transit controls in `security-architecture-patterns.md`. It covers the practical decisions engineers actually get wrong: which primitive to reach for, the implementation pitfalls that turn correct-primitive-choice into a vulnerability anyway, key management, and — the most important section — when not to build any of this yourself.

## 1. Symmetric vs. asymmetric: the decision tree

**Start here:**

```
Do both parties already share a secret (or can establish one out-of-band)?
│
├── YES → Use symmetric encryption (AES-GCM / ChaCha20-Poly1305)
│         Fast, small ciphertext overhead. Use for:
│           - Encrypting data at rest (files, DB columns, backups)
│           - Bulk data encryption in any protocol, even ones that
│             use asymmetric crypto for key exchange (TLS does both)
│
└── NO, parties are strangers with no shared secret →
    │
    Do you need confidentiality, or authenticity/non-repudiation, or both?
    │
    ├── Confidentiality only (encrypt to someone whose public key you have)
    │     → Asymmetric encryption (RSA-OAEP, or ECIES) —
    │       but in practice: use asymmetric crypto to establish a
    │       shared symmetric key (ECDH), then switch to symmetric
    │       (AES-GCM) for the actual payload. This is "hybrid
    │       encryption" and is how essentially every real protocol
    │       (TLS, Signal, age, PGP) works — almost nothing encrypts
    │       bulk data directly with RSA.
    │
    └── Authenticity / non-repudiation / signatures
          → Asymmetric signatures (Ed25519 preferred; ECDSA/RSA-PSS
            if constrained by platform support). Use for:
              - JWT/token signing where the verifier shouldn't be
                able to forge new tokens (asymmetric: verifier only
                has the public key)
              - Software/artifact signing
              - Any scenario where "prove I said this and I can't
                later deny it" matters (non-repudiation ties directly
                to the STRIDE "R" category)
```

**Key sizing (as of current practice):**

| Primitive | Minimum acceptable | Recommended |
|---|---|---|
| AES | 128-bit | 256-bit (AES-256-GCM) |
| RSA | 2048-bit | 3072–4096-bit, or avoid RSA and use ECC |
| ECDSA/ECDH | P-256 | P-256 or Curve25519 |
| Ed25519 | fixed 256-bit | (no smaller option — this is the point) |
| Password hashing | n/a — see §4 | Argon2id |

**Why hybrid, not pure asymmetric, for bulk data:** RSA/ECIES have message-size limits and are orders of magnitude slower than AES per byte. Every practical system — TLS's handshake, age encryption, PGP/GPG, Signal's protocol — uses asymmetric operations only to *establish or wrap* a symmetric key, then does the actual bulk encryption symmetrically. If you find yourself RSA-encrypting a large payload directly, that's a design smell, not a valid shortcut.

## 2. Common implementation pitfalls

Choosing the right primitive is necessary but not sufficient — most real-world crypto breaks happen in *how* a correct primitive is used, not in the primitive itself.

### ECB mode

AES in ECB (Electronic Codebook) mode encrypts each block independently with no chaining — identical plaintext blocks produce identical ciphertext blocks. This leaks structure: the canonical demonstration is encrypting an image in ECB mode and still being able to see the outline of the original image in the ciphertext, because repeated regions of pixels produce repeated ciphertext blocks.

**Never use ECB.** Use AES-GCM (provides both confidentiality and integrity/authentication in one primitive — an AEAD, Authenticated Encryption with Associated Data) for basically all new symmetric encryption needs. If GCM's hardware acceleration isn't available on your target platform, ChaCha20-Poly1305 is the standard alternative AEAD.

```
# WRONG — ECB, no IV, no authentication
cipher = AES.new(key, AES.MODE_ECB)
ciphertext = cipher.encrypt(pad(plaintext))

# RIGHT — AES-GCM, fresh random nonce, authenticated
nonce = os.urandom(12)          # 96-bit nonce is the GCM standard size
cipher = AES.new(key, AES.MODE_GCM, nonce=nonce)
ciphertext, tag = cipher.encrypt_and_digest(plaintext)
# store: nonce || ciphertext || tag  (all three needed to decrypt/verify)
```

### IV/nonce reuse

For GCM specifically, reusing a (key, nonce) pair is catastrophic — it doesn't just weaken confidentiality, it can let an attacker recover the authentication key and forge arbitrary authenticated ciphertexts under that key. This is a strictly worse failure mode than IV reuse under CBC (which "only" leaks XOR of two plaintext blocks).

Rules that prevent this in practice:
- **Never hardcode or reuse a nonce.** Generate it fresh, randomly, per encryption operation — `os.urandom(12)` per call, not once at startup.
- **Never derive a nonce from something that can repeat** (e.g., a counter that resets on restart without persisted state, or a timestamp with insufficient resolution under high throughput).
- **If you must use a fixed key for a very high volume of messages,** track nonce usage explicitly (a persisted counter that survives restarts) rather than relying on `os.urandom` collision probability alone — at sufficient volume even 96-bit random nonces have a non-negligible birthday-bound collision risk (~2^32 messages per key is the commonly cited caution threshold for random 96-bit GCM nonces). If volume is that high, rotate keys more frequently (see §3) rather than trying to out-engineer nonce collision math.
- **CBC mode**, if you're stuck supporting it for legacy compatibility, requires a random, unpredictable IV per message — a static or sequential IV lets an attacker manipulate plaintext via bit-flipping in adjacent blocks (and CBC without a separate MAC is also vulnerable to padding-oracle attacks — this is why AEAD modes like GCM superseded "encrypt-then-separately-MAC" CBC constructions for new designs).

### Timing attacks

Any comparison or cryptographic operation whose execution time depends on secret data leaks that data through the side channel of "how long did this take."

Classic example — comparing a computed HMAC/signature to an expected value with a naive `==`:

```
# WRONG — early-exit string comparison leaks how many leading
# bytes matched, via timing, letting an attacker forge a valid
# tag byte-by-byte over many requests
if computed_tag == expected_tag:
    ...

# RIGHT — constant-time comparison
import hmac
if hmac.compare_digest(computed_tag, expected_tag):
    ...
```

This applies anywhere a secret-derived value is compared: MAC verification, password-reset-token comparison, API-key comparison, session-token comparison. The fix is always the same shape: use your language's constant-time comparison primitive (`hmac.compare_digest` in Python, `crypto.timingSafeEqual` in Node, `subtle.ConstantTimeCompare` in Go) — never `==`, `.equals()`, or `strcmp` for secret comparisons.

Timing side channels also show up in less obvious places: RSA decryption timing varying with the private key's bit pattern in naive implementations (mitigated by RSA blinding, which mature libraries implement automatically — another reason not to hand-roll RSA), and password-verification code that does expensive work (e.g., a DB lookup) only when the username exists, letting an attacker enumerate valid usernames by timing the response — the login endpoint should do constant-effort work regardless of whether the account exists (this is the same account-enumeration Information-Disclosure threat scored in `threat-modeling-guide.md` §4, and here it is again as a timing-channel implementation detail rather than an error-message detail).

### Other frequently-seen pitfalls worth naming

- **Using a weak or predictable source of randomness** (`random.random()`, `Math.random()`, or any non-cryptographic PRNG) for keys, nonces, tokens, or password-reset codes. Always use a CSPRNG (`os.urandom`, `crypto.randomBytes`, `secrets` module in Python) for anything security-sensitive.
- **Rolling your own "encryption" via XOR with a repeating key, or a custom obfuscation scheme** and calling it encryption. This is not encryption; frequency analysis or known-plaintext attacks break it trivially.
- **Storing passwords with a fast general-purpose hash** (MD5, SHA-256 alone) instead of a purpose-built slow password hash — see §4.
- **Encrypting without integrity protection** — plain CBC/CTR with no MAC and no AEAD means an attacker can flip ciphertext bits and produce plaintext that decrypts to attacker-chosen values with no error, since there's nothing checking authenticity. Always use an AEAD mode, or explicitly add HMAC-then-MAC (encrypt-then-MAC order specifically, not MAC-then-encrypt) if you're stuck with a non-AEAD cipher.

## 3. Key management and rotation

Correct primitives and correct usage still fail if key lifecycle is unmanaged.

**Generation:** always from a CSPRNG, sized per §1's table. Never derive keys from low-entropy sources (a password directly used as an AES key) — if you must derive a key from a password, use a proper KDF (Argon2id, or PBKDF2/scrypt if constrained) with a high work factor, which is a different problem from password *storage* hashing but uses the same family of primitives.

**Storage:** keys live in a dedicated secrets manager (AWS KMS/Secrets Manager, HashiCorp Vault, GCP KMS, Azure Key Vault) — never in source code, environment variables checked into config repos, or application config files committed to version control. This is exactly the failure mode `secret_scanner.py` in the SKILL.md workflow is built to catch — a hardcoded key or credential found by that scan is a key-management failure, and per SKILL.md's gate, any high/critical finding blocks merge until rotated and moved to a secret manager, not just deleted from the current commit (git history retains it).

**Rotation:** rotate on a schedule (commonly 90 days for high-sensitivity keys, longer for lower-sensitivity ones, per your compliance regime) *and* immediately on suspected compromise. Rotation requires a design that supports key versioning — don't just swap the key and break every already-encrypted record:

```
# Encrypted record carries its key's identifier so old data
# stays decryptable after the active key rotates
{
  "key_id": "kms-key-2026-q2",
  "nonce": "...",
  "ciphertext": "...",
  "tag": "..."
}
```

On rotation: generate/register the new key version, switch *new* encryption operations to it, but keep old key versions available (in the KMS, not in application code) for decrypting existing data until it's been re-encrypted under the new key or has aged out per retention policy. Envelope encryption (a data-encryption-key per record/file, itself encrypted by a master key held in the KMS) makes this tractable at scale — you rotate the master key without re-encrypting every payload, only re-wrapping the much smaller data-encryption-keys.

**Access control on keys themselves:** apply least privilege (per `security-architecture-patterns.md` §3) to the KMS/Vault policies — the service that encrypts customer PII shouldn't necessarily have decrypt permission if it never needs to read that data back (write-only encryption use cases exist, e.g., encrypting audit logs a service only appends to). Log every key-use operation (KMS access logs) — this is the audit/detection layer of defense-in-depth, and it's what lets you answer "was this key used to decrypt anything after the breach was detected" during incident response.

**Revocation:** for asymmetric keys used for signing/authentication (TLS certs, signing keys), have a working revocation path (CRL/OCSP for certs, an explicit revocation list for application-level signing keys) and actually test it before you need it under pressure.

## 4. Password hashing specifically

Passwords are not "data to encrypt" — they're secrets you need to *verify* without ever storing in a reversible form. Use a purpose-built password hashing function, not a general-purpose cryptographic hash and not reversible encryption:

- **Argon2id** — current recommended default. Memory-hard (resists GPU/ASIC cracking better than purely CPU-hard functions), tunable time/memory/parallelism cost parameters.
- **bcrypt** — acceptable fallback where Argon2 isn't available in the platform's crypto library; has a max input length quirk (commonly cited as 72 bytes) to be aware of.
- **scrypt** — also memory-hard, acceptable alternative.
- **Never**: raw SHA-256/SHA-512/MD5 of the password, even with a salt — these are fast hashes, designed for speed, which is the opposite of what you want for password storage since it lets an attacker with a stolen hash dump brute-force at billions of guesses/second on commodity GPUs.

Each password hash should embed a unique random salt (Argon2id and bcrypt do this automatically as part of their standard encoded output format) — never a shared/global salt, which lets attackers precompute one rainbow table for the entire user base instead of needing a separate one per user.

## 5. When to use vs. when not to build your own crypto

**Use existing, audited libraries — always, for the primitives themselves:**
- Python: `cryptography` (hazmat layer for primitives, recipes layer for common patterns), `PyNaCl` (libsodium bindings)
- Node: built-in `crypto` module, or `libsodium-wrappers`
- Go: standard library `crypto/*` packages
- General: libsodium is a good default across languages specifically because its API is designed to make misuse (ECB, nonce reuse, missing authentication) hard to reach by default.

**Never build your own:**
- A novel encryption algorithm or cipher mode. Every homegrown scheme that hasn't survived years of public cryptanalysis should be assumed breakable — "nobody will bother attacking my custom scheme" is not a security property, and the entire history of applied cryptography is professionals' custom schemes getting broken once anyone looked.
- A novel authentication/signature scheme, or a "simplified" reimplementation of an existing one for "performance" or "to avoid a dependency."
- Your own random number generator ("we seed it with the server's boot time and process ID for extra entropy") instead of the OS/language CSPRNG.
- A "lightweight" reimplementation of TLS, JWT validation, or OAuth token verification instead of a maintained library — these have had years of edge-case vulnerabilities (algorithm confusion, padding oracles, signature-wrapping attacks) found and fixed in mainstream libraries; a from-scratch reimplementation starts from zero on all of that hard-won knowledge.

**Where composing existing primitives into your own protocol *is* reasonable — with care:**
- Designing an application-level encryption scheme that *composes* audited primitives (e.g., "we use AES-256-GCM with envelope encryption and Ed25519-signed metadata for our backup format") is normal system design, not "building your own crypto" in the dangerous sense — the danger zone is inventing new low-level primitives (ciphers, hash functions, signature math), not combining existing ones.
- Even so, novel *compositions* of correct primitives can still introduce protocol-level flaws (padding oracles, cross-protocol attacks, missing domain separation between different uses of the same key). For anything beyond "encrypt this blob, verify that signature" — multi-party protocols, novel authentication flows, anything going into a widely-deployed product — get a third-party cryptographic review before shipping. This is also the point at which this skill routes out: implementation-level crypto review composing primitives is in scope here, but a from-scratch protocol design for a high-stakes system belongs in front of a dedicated cryptography specialist or external audit, not decided solely from this reference document.

**The practical rule of thumb:** if you're choosing which well-known primitive to use and how to wire it up correctly (this document's §1–§4), you're doing this skill's job. If you're inventing a new primitive or a new protocol from scratch, stop and get a specialist review — the cost of a subtle cryptographic design flaw shipped to production vastly exceeds the cost of that review.
