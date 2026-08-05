---
name: a11y-audit
description: WCAG 2.2 accessibility audit and fix skill for React, Next.js, Vue, Angular, Svelte, and HTML codebases. Use when the user asks for an accessibility audit, a11y review, WCAG compliance check, ADA compliance check, color/contrast checking, screen-reader or keyboard-navigation fixes, ARIA fixes, or wants to add an accessibility gate to CI/CD. Runs a static scanner across 10 violation categories (images, forms, headings, landmarks, keyboard, ARIA, color/contrast, links, tables, media), checks color pairs against WCAG 2.2 contrast thresholds with a suggest mode, applies framework-specific fixes, and produces CI-friendly exit codes and markdown reports.
license: MIT
metadata:
  version: 2.9.0
  author: Alireza Rezvani (https://alirezarezvani.com)
  category: engineering-team
---

# a11y-audit

## Overview

`a11y-audit` is a WCAG 2.2 accessibility audit and remediation skill for frontend
codebases (React, Next.js, Vue, Angular, Svelte, and plain HTML). It combines a
static violation scanner, a color-contrast checker, framework-aware fix patterns,
and CI wiring guidance so that accessibility issues can be found, triaged, fixed,
verified, and gated in a repeatable, scriptable way.

The skill ships two standalone scripts (no build step required — plain Python) and
nine reference documents covering ARIA patterns, contrast math, framework-specific
fixes, WCAG 2.2's new success criteria, CI/CD integration, and report templates.

## When to use this skill

Use `a11y-audit` when the user asks to:
- Run a WCAG 2.2 / accessibility / a11y audit on a project or component
- Find or fix accessibility violations (missing alt text, unlabeled form fields,
  heading order, missing landmarks, keyboard traps, ARIA misuse, broken link text,
  malformed tables, inaccessible media)
- Check whether a foreground/background color pair meets contrast requirements, or
  get a suggested compliant color
- Add an accessibility check/gate to a CI pipeline (pre-commit, GitHub Actions,
  GitLab CI, etc.) with a pass/fail exit code
- Generate an accessibility audit report for a codebase or PR

## Workflow

Follow this sequence for a complete audit-and-fix pass. Steps 4-6 can be skipped if
the user only wants a scan/report, and steps 1-2 can be skipped if the user only
wants a one-off contrast check.

### 1. Run the scanner

Scan the target project for violations across all 10 categories:

```bash
python scripts/a11y_scanner.py /path/to/project
```

Common variants:

```bash
# Machine-readable JSON output (for parsing or CI)
python scripts/a11y_scanner.py /path/to/project --json

# Only surface the most important severities
python scripts/a11y_scanner.py /path/to/project --severity critical,serious

# Explicit output format flag (equivalent to --json when set to json)
python scripts/a11y_scanner.py /path/to/project --format json

# Generate a full markdown audit report file
python scripts/a11y_scanner.py /path/to/project --report --output audit-report.md
```

### 2. Triage findings by severity

Group the scanner's findings into `critical`, `serious`, `moderate`, and `minor`
buckets (as emitted by the scanner). Address in this order:
1. `critical` — blocks assistive technology users entirely (e.g., no accessible
   name on an interactive control, keyboard traps)
2. `serious` — major barriers (e.g., missing form labels, broken landmark
   structure, heading order violations)
3. `moderate` / `minor` — polish issues (e.g., redundant ARIA, non-descriptive
   link text)

Cross-reference unfamiliar rule IDs against `references/wcag-quick-ref.md` to
confirm the underlying success criterion and its level (A/AA/AAA).

### 3. Apply framework-specific fixes

For each finding, load `references/framework-a11y-patterns.md` and, if working in
JSX/TSX, also `references/examples-by-framework.md` for copy-pasteable
before/after snippets scoped to React, Next.js, Vue, Angular, or Svelte. For
findings involving custom widgets (tabs, menus, dialogs, comboboxes, etc.), load
`references/aria-patterns.md` for the correct WAI-ARIA Authoring Practices pattern
before changing markup. Use `assets/sample-component.tsx` as a worked reference
for what a fixed component looks like end to end.

### 4. Verify color/contrast fixes

For every color pair touched or flagged by the scanner, verify (or find a
compliant replacement) with the contrast checker:

```bash
# Check a single foreground/background pair
python scripts/contrast_checker.py "#ffffff" "#000000"

# Get a suggested compliant color close to the one in use
python scripts/contrast_checker.py "#336699" --suggest "#336699"

# Check every color pair declared in a stylesheet
python scripts/contrast_checker.py --batch styles.css

# See example thresholds and passing/failing pairs
python scripts/contrast_checker.py --demo
```

Consult `references/color-contrast-guide.md` for the underlying thresholds
(4.5:1 normal text, 3:1 large text/UI components, AAA variants) and how they map
to WCAG 2.2 criteria.

### 5. Generate the report

Re-run the scanner with `--report --output audit-report.md` to produce a final
markdown report. Use `references/audit-report-template.md` as the structural
template (sections, severity summary table, per-finding detail block) and
`expected_outputs/sample-scan-report.md` as a reference for expected formatting.

### 6. Wire into CI

If the user wants an automated gate, follow `references/ci-cd-integration.md` to
add the scanner (with `--severity critical,serious` and JSON output for parsing)
as a pipeline step that fails the build on nonzero exit code / any critical
finding. Use `references/testing-checklist.md` to also cover manual/assistive-tech
checks (screen reader pass, keyboard-only pass) that the static scanner cannot
catch.

## Definition of done / verification

Before declaring the audit or fix pass complete:

1. Re-run the scanner against the same target:
   ```bash
   python scripts/a11y_scanner.py /path/to/project --severity critical,serious --json
   ```
2. Confirm the process exit code is `0` (non-zero indicates remaining
   critical/serious findings — treat this the same way CI would).
3. Confirm the JSON/report output shows **zero `critical` findings**, and that
   any remaining `serious` findings are explicitly acknowledged/accepted by the
   user (e.g., tracked as follow-up, not silently dropped).
4. For any color changes made, confirm `contrast_checker.py` reports a pass
   (`AA` at minimum, `AAA` if the user asked for AAA) for every changed pair.
5. Compare current output against `expected_outputs/sample-scan-output.json` and
   `expected_outputs/sample-contrast-output.txt` only as formatting references —
   not as literal expected values, since results are project-specific.

## References

Load these on demand — don't load all of them up front.

- `references/wcag-quick-ref.md` — Condensed WCAG 2.2 success-criteria lookup table (level A/AA/AAA, short description); use to quickly identify what a scanner rule ID maps to.
- `references/wcag-22-new-criteria.md` — Details on the success criteria new in WCAG 2.2 (e.g., focus appearance, dragging movements, target size, accessible authentication); use when a finding relates to a 2.2-specific rule or the user asks what's new versus 2.1.
- `references/aria-patterns.md` — WAI-ARIA Authoring Practices patterns for custom interactive widgets (menus, tabs, dialogs, comboboxes, etc.); use before fixing/authoring any custom ARIA widget.
- `references/framework-a11y-patterns.md` — Framework-specific accessibility fix patterns (React/Next.js/Vue/Angular/Svelte idioms: focus management, semantic elements, form binding); use as the primary fix-authoring reference in step 3.
- `references/examples-by-framework.md` — Concrete before/after code examples of common fixes broken out per framework; use alongside framework-a11y-patterns.md for copy-paste snippets.
- `references/color-contrast-guide.md` — Explanation of WCAG contrast ratio math and thresholds for normal text, large text, and UI components; use when interpreting or explaining contrast_checker.py output.
- `references/testing-checklist.md` — Manual verification checklist (screen reader, keyboard-only, zoom/reflow) covering what static scanning cannot catch; use as a supplement after automated fixes, before sign-off.
- `references/audit-report-template.md` — Structural template for the markdown audit report (summary table, per-finding sections); use when generating a report with `--report`.
- `references/ci-cd-integration.md` — How to wire the scanner into pre-commit hooks and CI systems (GitHub Actions, GitLab CI, generic exit-code gating); use when the user wants an automated accessibility gate.
