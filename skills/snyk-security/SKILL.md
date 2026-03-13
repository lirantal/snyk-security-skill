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

Snyk is the primary security scanning tool across four domains: source code, dependencies,
containers, and infrastructure-as-code. The goal is never just to run a scan — it's to
leave the project with its security issues actually resolved.

## Prerequisites

Before scanning, verify Snyk is installed and authenticated:

```bash
snyk --version          # should return a version number
snyk auth               # authenticate if not already done (opens browser)
# or: SNYK_TOKEN=<token> snyk <command>
```

If Snyk is not installed:
- npm: `npm install -g snyk`
- Homebrew: `brew tap snyk/tap && brew install snyk`
- Direct download: https://github.com/snyk/cli/releases

---

## Use Cases

### Use Case 1: Secure Project Source Code (SAST)

**Goal:** Find and fix security vulnerabilities in the code the developer wrote —
SQL injection, XSS, hardcoded secrets, insecure crypto, path traversal, etc.

**Reference file:** `references/code-security.md` — read this for command flags,
interpreting findings, fix patterns, and secure coding rules to apply during fixes.

**Steps:**

1. **Scan** — Run Snyk Code against the project:
   ```bash
   snyk code test
   ```

2. **Analyze** — For each finding, understand:
   - What type of vulnerability it is (e.g., SQL Injection, XSS)
   - Where the vulnerable code is (file + line number)
   - The data flow Snyk shows — how untrusted input reaches the vulnerable sink
   - The severity and whether it's practically exploitable in context

3. **Fix** — Modify the source code to remediate the vulnerability. Prioritize
   Critical and High severity first. Apply the secure coding patterns from
   `references/code-security.md` — don't just suppress the finding, fix the root cause.

4. **Verify** — Re-run `snyk code test` and confirm the finding is gone. If new
   issues surface from the fix, address those too.

5. **Repeat** steps 2–4 until no Critical or High severity issues remain.

**Done when:** `snyk code test` returns no Critical or High findings, or the
remaining findings have been reviewed and accepted with documented justification.

---

### Use Case 2: Secure Project Dependencies (SCA)

**Goal:** Find and fix known vulnerabilities (CVEs) in open-source libraries and
packages the project depends on — both direct and transitive dependencies.

**Reference file:** `references/dependency-security.md` — read this for package
manager specifics, fix strategies, and CI integration patterns.

**Steps:**

1. **Ensure dependencies are installed** — Snyk reads the actual dependency tree,
   so install first:
   ```bash
   npm install        # or pip install, mvn install, go mod download, etc.
   ```

2. **Scan** — Run Snyk's dependency scan:
   ```bash
   snyk test
   # For monorepos or multiple manifests:
   snyk test --all-projects
   ```

3. **Analyze** — For each vulnerability, understand:
   - Which package is affected and what the CVE is
   - Whether Snyk shows a fix path (upgradeable / patchable / no fix)
   - Whether the vulnerability is reachable given how the package is used
   - Whether it's in a production or dev-only dependency

4. **Fix** — Apply the fix that Snyk recommends:
   - **Upgrade:** bump the direct dependency to the version that resolves it
   - **Patch:** apply Snyk's patch if available (`snyk fix`)
   - **Replace:** swap out a package with no fix for a maintained alternative
   - **Ignore (last resort):** `snyk ignore --id=<ID> --reason="..." --expiry=<date>`

5. **Verify** — Re-run `snyk test` and confirm the fixed vulnerabilities are gone.
   Check that the upgrade didn't introduce new issues.

6. **Repeat** steps 3–5 until no Critical or High severity findings remain.

**Done when:** `snyk test` returns no Critical or High findings, or remaining
findings are documented ignores with justification and expiry dates.

---

### Use Case 3: Secure Container Images

**Goal:** Find and fix vulnerabilities in Docker images — OS package CVEs, bundled
application dependency CVEs, and Dockerfile security misconfigurations.

**Reference file:** `references/container-security.md` — read this for base image
selection, Dockerfile best practices, and rebuild/re-scan workflow.

**Steps:**

1. **Scan** — Test the image, including the Dockerfile if available:
   ```bash
   snyk container test <image>:<tag>
   # With Dockerfile for enhanced analysis:
   snyk container test <image>:<tag> --file=Dockerfile
   ```

2. **Analyze** — Group findings by type:
   - **OS package vulnerabilities** — what base image introduced them, and whether
     a base image upgrade would fix them (Snyk suggests specific alternatives)
   - **Application dependency vulnerabilities** — packages installed via npm/pip/etc.
     inside the image
   - **Dockerfile misconfigurations** — running as root, exposed ports, no USER
     directive, secrets in ENV, etc.

3. **Fix** — Address each category:
   - **Base image upgrade:** update the `FROM` line in the Dockerfile to the version
     Snyk recommends, or switch to a slimmer/distroless alternative
   - **App dependency CVEs:** fix in the source manifest (same as Use Case 2) and
     rebuild
   - **Dockerfile issues:** apply best practices from `references/container-security.md`

4. **Rebuild and verify:**
   ```bash
   docker build -t <image>:<new-tag> .
   snyk container test <image>:<new-tag> --file=Dockerfile
   ```

5. **Repeat** steps 2–4 until no Critical or High findings remain.

**Done when:** The rebuilt image passes `snyk container test` with no Critical or
High severity findings.

---

### Use Case 4: Secure Infrastructure-as-Code

**Goal:** Find and fix security misconfigurations in IaC files before they reach
production — publicly exposed resources, missing encryption, over-permissive IAM,
containers running as root, missing network policies, etc.

**Reference file:** `references/iac-security.md` — read this for Terraform,
Kubernetes, and CloudFormation fix patterns with concrete before/after examples.

**Steps:**

1. **Scan** — Test all IaC files in the project:
   ```bash
   snyk iac test ./infrastructure/
   # Or a specific file:
   snyk iac test main.tf
   snyk iac test k8s/deployment.yaml
   ```

2. **Analyze** — For each finding, understand:
   - The affected resource and the specific misconfigured attribute (Snyk shows
     the exact path, e.g., `aws_s3_bucket.data > acl`)
   - What security risk the misconfiguration creates
   - Whether it's a real exposure or already mitigated by other controls

3. **Fix** — Modify the IaC file to correct the misconfiguration. Use the examples
   in `references/iac-security.md` as templates. The principle is always least
   privilege and defense in depth: restrict access, enable encryption, require
   authentication.

4. **Verify** — Re-run `snyk iac test` on the modified file to confirm the finding
   is resolved and no new issues were introduced.

5. **Repeat** steps 2–4 until no Critical or High severity findings remain.

**Done when:** `snyk iac test` returns no Critical or High findings. Review
remaining Medium/Low findings and document accepted risks where applicable.

---

## Choosing the Right Use Case

When the user's request is ambiguous (e.g., "make my project secure"), assess the
project and run all applicable use cases. A typical full-project security pass:

```bash
snyk code test          # source code (if applicable language)
snyk test               # dependencies
snyk container test ... # if Docker is in use
snyk iac test ./infra   # if IaC files exist
```

Address findings in order of severity across all scan types.

## Severity Reference

- **Critical** — Actively exploitable, fix before anything else
- **High** — Significant risk, fix before next release
- **Medium** — Address in normal sprint cycles
- **Low** — Track and fix when convenient

## After All Issues Are Resolved

```bash
snyk monitor            # upload snapshot for ongoing monitoring
```

Snyk will alert when new vulnerabilities are published that affect the project.
