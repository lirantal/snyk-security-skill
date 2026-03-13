---
name: snyk-security
description: >
  Runs Snyk CLI security scans and fixes the vulnerabilities found. Covers four domains:
  source code (SAST via snyk code test), open-source dependencies (SCA via snyk test),
  container images (snyk container test), and infrastructure-as-code files such as
  Terraform, Kubernetes YAML, and CloudFormation (snyk iac test). Use this skill whenever
  the user wants to scan for vulnerabilities, audit dependencies, secure a Dockerfile or
  container image, check IaC for misconfigurations, or asks "is my code/app/image/infra
  secure?". Also trigger for requests like "fix security issues", "run a security scan",
  "check for CVEs", "find hardcoded secrets", or "review my Terraform for security problems"
  even if Snyk is not mentioned by name.
license: MIT
compatibility: >
  Requires the Snyk CLI to be installed and authenticated (snyk auth or SNYK_TOKEN env var).
  Compatible with Claude Code on macOS, Linux, and Windows. Requires internet access for
  Snyk vulnerability database lookups. Supported languages and ecosystems vary by scan type;
  see reference files for per-domain details.
metadata:
  author: lirantal
  version: 1.0.0
---

# Snyk Security Scanning

# Instructions

The goal is never just to run a scan — it's to leave the project with its security
issues actually resolved. Each use case below follows the same core loop:
scan → analyze → fix → verify → repeat until clean.

Before starting any scan, verify Snyk is ready:

```bash
snyk --version   # confirms CLI is installed
snyk auth        # opens browser to authenticate (one-time setup)
```

If Snyk is not installed, help the user install it first:
- npm: `npm install -g snyk`
- Homebrew: `brew tap snyk/tap && brew install snyk`

---

### Step 1: Identify the scan scope

Determine which domains apply to this project. When the user's request is specific
("scan my Dockerfile"), go directly to that use case. When it's ambiguous ("make my
project secure"), run all applicable scans.

| What to scan | Use case | CLI command |
|---|---|---|
| Source code (JS, Python, Java, Go, etc.) | Use Case A | `snyk code test` |
| npm, pip, Maven, Go modules, etc. | Use Case B | `snyk test` |
| Docker images | Use Case C | `snyk container test` |
| Terraform, Kubernetes YAML, CloudFormation | Use Case D | `snyk iac test` |

---

### Step 2: Run the scan (choose the use case)

#### Use Case A — Secure Project Source Code (SAST)

Finds vulnerabilities in code the developer wrote: SQL injection, XSS, hardcoded
secrets, insecure crypto, path traversal, etc.

Reference: `references/code-security.md` — command flags, fix patterns, and secure
coding rules to apply during remediation.

```bash
snyk code test
```

Expected output: a list of findings grouped by severity, each with file path, line
number, vulnerability type, and data flow showing how untrusted input reaches the sink.

For each finding, prioritize Critical and High first. Read the data flow to understand
the attack vector, then fix the root cause in the source code. Re-run to confirm
the finding is gone before moving to the next one.

**Done when:** `snyk code test` returns no Critical or High findings.

---

#### Use Case B — Secure Project Dependencies (SCA)

Finds known CVEs in open-source libraries — both direct and transitive dependencies.

Reference: `references/dependency-security.md` — package manager specifics, upgrade
paths, ignore workflow, and CI integration patterns.

```bash
# Install dependencies first so Snyk can read the full tree
npm install   # or: pip install -r requirements.txt, mvn install, go mod download, etc.

snyk test
# Monorepos or multiple manifests:
snyk test --all-projects
```

Expected output: vulnerabilities grouped by package, each with CVE ID, severity,
affected version range, and fix path (upgrade / patch / no fix available).

For each finding: upgrade direct dependencies to the fixed version, run `snyk fix`
for auto-applicable patches, or document an ignore with justification and expiry
as a last resort. Re-run after each fix to confirm and check for newly exposed issues.

**Done when:** `snyk test` returns no Critical or High findings.

---

#### Use Case C — Secure Container Images

Finds OS package CVEs, application dependency CVEs, and Dockerfile misconfigurations
in Docker images.

Reference: `references/container-security.md` — base image selection, Dockerfile
best practices, multi-stage builds, rebuild/re-scan workflow.

```bash
snyk container test IMAGE:TAG
# With Dockerfile for enhanced misconfiguration analysis:
snyk container test IMAGE:TAG --file=Dockerfile
```

Expected output: OS vulnerabilities grouped by base image layer, application
dependency vulnerabilities, and Dockerfile issues (running as root, missing USER,
secrets in ENV, etc.). Snyk also suggests specific base image upgrades that would
reduce the total vulnerability count.

Fix by upgrading the base image (`FROM` line) to the version Snyk recommends,
fixing any app dependency CVEs in the source manifest, and correcting Dockerfile
issues. Rebuild and re-scan to verify.

```bash
docker build -t IMAGE:NEW-TAG .
snyk container test IMAGE:NEW-TAG --file=Dockerfile
```

**Done when:** The rebuilt image passes `snyk container test` with no Critical or
High severity findings.

---

#### Use Case D — Secure Infrastructure-as-Code

Finds misconfigurations in Terraform, Kubernetes, CloudFormation, ARM templates,
Helm charts, and Serverless files before they reach production.

Reference: `references/iac-security.md` — fix patterns with before/after examples
for AWS, GCP, Azure, and Kubernetes.

```bash
snyk iac test ./infrastructure/
# Or target specific files:
snyk iac test main.tf
snyk iac test k8s/deployment.yaml
```

Expected output: findings grouped by severity, each with the affected resource name,
the exact misconfigured attribute path (e.g., `aws_s3_bucket.data > acl`), the
security risk it creates, and remediation guidance.

Fix by modifying the IaC file to correct the misconfiguration — apply least privilege,
enable encryption, restrict public access. Re-run to verify. The principle across all
IaC fixes is: default to deny, encrypt at rest and in transit, minimize blast radius.

**Done when:** `snyk iac test` returns no Critical or High findings.

---

### Step 3: Monitor ongoing security

After resolving all findings, register the project for continuous monitoring:

```bash
snyk monitor
```

Snyk will alert when new vulnerabilities are published that affect the project,
so issues are caught before they become incidents.

---

## Examples

**Example 1: Full project security audit**

User says: "Can you do a full security scan of this project?"

Actions:
1. Check Snyk is installed and authenticated
2. Run `snyk code test` — review and fix any High/Critical source code findings
3. Run `npm install` then `snyk test` — upgrade vulnerable dependencies
4. If a Dockerfile exists, run `snyk container test IMAGE:TAG --file=Dockerfile`
5. If IaC files exist, run `snyk iac test ./infra`
6. Run `snyk monitor` to register for ongoing alerts

Result: All Critical and High findings resolved across all applicable domains.
Project is registered for continuous monitoring.

---

**Example 2: Fix a specific CVE in a dependency**

User says: "Snyk says lodash 4.17.15 has a prototype pollution vulnerability, can you fix it?"

Actions:
1. Run `snyk test` to confirm the finding and see the upgrade path
2. Check if a direct upgrade resolves it: `npm install lodash@latest`
3. Re-run `snyk test` to confirm the vulnerability is gone
4. If lodash is a transitive dep, identify which direct dependency pulls it in and
   upgrade that, or use an `overrides` entry in package.json

Result: `snyk test` no longer reports the prototype pollution CVE. package.json
and package-lock.json are updated with the safe version.

---

**Example 3: Secure a Dockerfile before deployment**

User says: "We're about to push a new container to production, scan it first"

Actions:
1. Run `snyk container test myapp:latest --file=Dockerfile`
2. Review OS vulnerabilities — check if Snyk suggests a base image upgrade
3. Review Dockerfile issues — look for missing USER directive, secrets in ENV, etc.
4. Update the `FROM` line to the recommended base image version
5. Fix any Dockerfile misconfigurations (add USER, move secrets to runtime env)
6. Run `docker build -t myapp:latest-secure .` and re-scan to verify

Result: Container image rebuilt with updated base image and secure Dockerfile.
`snyk container test` passes with no Critical or High findings.

---

**Example 4: Review Terraform for security issues**

User says: "Review my Terraform files for any security problems before we apply"

Actions:
1. Run `snyk iac test ./terraform/`
2. Identify Critical/High findings — commonly: public S3 buckets, missing encryption,
   over-permissive security groups, no CloudTrail logging
3. Fix each misconfiguration using patterns from `references/iac-security.md`
4. Re-run `snyk iac test` to confirm all issues are resolved
5. Confirm the Terraform plan looks correct after changes

Result: Terraform files updated with security fixes. `snyk iac test` returns no
Critical or High findings. Safe to apply.

---

## Troubleshooting

**Error:** `snyk: command not found`
Cause: Snyk CLI is not installed.
Solution: `npm install -g snyk` or `brew tap snyk/tap && brew install snyk`

---

**Error:** `Authentication failed. Please run snyk auth.`
Cause: Not authenticated with Snyk.
Solution: Run `snyk auth` (opens a browser to log in) or set the `SNYK_TOKEN`
environment variable with a token from https://app.snyk.io/account.

---

**Error:** `Could not detect supported target files in [path]`
Cause: Snyk couldn't find a package manifest, or dependencies haven't been installed.
Solution: Run the appropriate install command first (`npm install`, `pip install -r requirements.txt`,
etc.), then re-run. Use `--file=<manifest>` to point Snyk at a specific file.

---

**Error:** `snyk code test` exits with no output or "SAST is not enabled"
Cause: Snyk Code (SAST) may not be enabled for the organization.
Solution: Log into https://app.snyk.io, go to Settings, and enable Snyk Code. If
using a free account, confirm Snyk Code is included in the plan.

---

**Exit code 1 vs exit code 2**
- Exit code `1` means vulnerabilities were found — this is expected and working correctly.
- Exit code `2` means an error occurred (bad auth, missing files, network issue).
Only exit code `2` indicates something is wrong with the scan itself.
