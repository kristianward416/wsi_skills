# Compliance Requirements

Reference for `senior-secops`: control mappings for SOC 2, PCI-DSS, HIPAA, and GDPR, to use alongside `compliance_checker.py`. This is an engineering-facing summary, not legal advice — for actual audit scoping or breach-notification obligations, involve legal/compliance counsel; the frameworks and thresholds below are commonly-cited industry summaries, not a substitute for the current text of each standard.

## SOC 2 (Trust Services Criteria)

Relevant Common Criteria (CC) for engineering:

| Criterion | Engineering control |
|---|---|
| CC6.1 (Logical Access) | Least-privilege IAM roles; access reviewed on a schedule; offboarding revokes access same-day. |
| CC6.6 (Boundary Protection) | Network segmentation; security groups/firewalls default-deny; public-facing surface minimized. |
| CC6.7 (Data Transmission) | TLS 1.2+ for all data in transit; no plaintext internal service-to-service traffic for sensitive data. |
| CC7.2 (Monitoring) | Centralized logging with alerting on auth failures, privilege changes, and anomalous access. |
| CC8.1 (Change Management) | Code changes go through review + CI checks before production; infra changes are code-reviewed (IaC), not manual console edits. |

`compliance_checker.py --framework soc2` maps these against detected infra config (IAM policies, security group rules, TLS config) and flags gaps.

## PCI-DSS (v4.0) — for anything touching cardholder data

- **Req 3**: Cardholder data at rest is encrypted or tokenized; PANs are never stored in logs.
- **Req 4**: Strong cryptography for cardholder data in transit (TLS 1.2+, no deprecated cipher suites).
- **Req 6**: Secure development practices — this overlaps directly with `references/security_standards.md`'s secure-coding section.
- **Req 8**: Unique IDs for every user with system access to cardholder data environments; MFA required for administrative access.
- **Req 10**: Logging and monitoring of all access to cardholder data and system components.
- **Req 11**: Regular vulnerability scanning and penetration testing (route to `security-pen-testing` for the engagement itself; this skill covers the ongoing scanning cadence).

Most engineering orgs should minimize PCI scope by using a tokenizing payment processor rather than touching raw PANs at all — check with `senior-architect` / `senior-backend` whether the current design can reduce scope before treating full PCI compliance as a given requirement.

## HIPAA — for anything touching Protected Health Information (PHI)

- **Access controls**: role-based access to PHI, unique user identification, automatic logoff.
- **Audit controls**: hardware/software/procedural mechanisms that record and examine activity on systems containing PHI.
- **Encryption**: PHI encrypted at rest and in transit (addressable but treated as required in practice by most auditors).
- **Business Associate Agreements (BAAs)**: any third-party vendor/subprocessor that touches PHI needs a signed BAA before integration — a compliance/legal item, but engineering should flag new subprocessors during design review, not after launch.
- **Breach notification**: HIPAA has strict, short breach-notification timelines — engineering's job is ensuring detection/logging is good enough that a breach is discoverable at all within the notification window; the notification itself is a legal/compliance action.

## GDPR — for anything touching EU residents' personal data

- **Data minimization**: collect only what's needed for the stated purpose; don't retain personal data past its useful life without a lawful basis.
- **Right to erasure / access / portability**: the system needs a real mechanism to locate and delete or export a specific person's data, not just a promise — verify this is achievable before it's a support ticket.
- **Encryption and pseudonymization**: recommended technical measures; strengthens the compliance posture and can reduce breach-notification severity.
- **Data Processing Agreements (DPAs)**: required with any processor handling EU personal data on the org's behalf — a legal item, but engineering should track which vendors qualify as processors.
- **Breach notification**: 72-hour notification to the relevant supervisory authority for qualifying breaches — same engineering obligation as HIPAA: make breaches detectable fast enough to hit the window.

## Running the Checker

```
python3 scripts/compliance_checker.py --framework soc2|pci-dss|hipaa|gdpr
```

Exit code 0 means no gaps detected against the mapped technical controls; exit code 1 lists gaps with the specific control ID. A clean exit code covers only the technical controls above — it is not a substitute for an actual third-party audit or legal compliance review.
