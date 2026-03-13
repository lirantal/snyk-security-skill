---
name: snyk-security
description: >
  Security scanning skill using the Snyk CLI. Use this skill whenever the user wants to
  scan for security vulnerabilities, find security issues in code, check dependencies for
  known CVEs, scan container images, or test infrastructure-as-code files for misconfigurations.
  Trigger this skill for any request involving: security scanning, vulnerability detection,
  SAST (static analysis), SCA (software composition analysis), dependency auditing, container
  security, Dockerfile analysis, Terraform/Kubernetes security checks, snyk commands, or
  "is my code/app/image/IaC secure?". Even if the user just asks to "check for issues" or
  "review security" without mentioning Snyk specifically — use this skill.
---

# Snyk Security Scanning

Snyk is the primary tool for security scanning across four domains. Each domain has its own
CLI command and workflow. Read the relevant reference file for domain-specific guidance.

## Prerequisites

Before scanning, verify Snyk is installed and authenticated:

```bash
snyk --version          # should return a version number
snyk auth               # authenticate if not already done (opens browser)
# or: SNYK_TOKEN=<token> snyk <command>
```

If Snyk is not installed, guide the user to install it:
- npm: `npm install -g snyk`
- Homebrew: `brew tap snyk/tap && brew install snyk`
- Direct: download from https://github.com/snyk/cli/releases

## Scanning Domains

Choose the right scan based on what the user wants to check:

| User intent | Domain | Reference file | CLI command |
|---|---|---|---|
| Scan source code for bugs/vulnerabilities | Code Security (SAST) | `references/code-security.md` | `snyk code test` |
| Scan npm/pip/Maven/etc. dependencies | Dependency Security (SCA) | `references/dependency-security.md` | `snyk test` |
| Scan Docker images / containers | Container Security | `references/container-security.md` | `snyk container test` |
| Scan Terraform, Kubernetes, CloudFormation | IaC Security | `references/iac-security.md` | `snyk iac test` |

When the user's request is clear, go straight to the relevant reference file and follow it.
When the context is ambiguous (e.g., "scan my project"), run all applicable scans.

## General Workflow

1. **Read the relevant reference file** for the domain being scanned
2. **Run the scan** with appropriate flags
3. **Interpret the results** — severity levels, affected files/packages, fix guidance
4. **Prioritize findings** — Critical and High severity issues first
5. **Suggest or apply fixes** — upgrades, patches, code changes, or config corrections
6. **Re-run to verify** fixes actually resolved the issues

## Severity Levels

Snyk uses four severity levels. Help the user understand what needs immediate attention:

- **Critical** — Actively exploitable, fix immediately
- **High** — Significant risk, fix before next release
- **Medium** — Worth addressing in normal sprint cycles
- **Low** — Track and fix when convenient

Use `--severity-threshold=high` to fail builds only on High/Critical issues in CI contexts.

## Useful Global Flags

```bash
--json                          # machine-readable output
--json-file-output=results.json # save JSON to file
--org=<org-slug>               # associate with a Snyk organization
--severity-threshold=<level>   # low|medium|high|critical
--all-projects                 # scan all manifests in a directory tree
-d                             # debug output for troubleshooting
```

## After Scanning

- `snyk monitor` — upload snapshot for continuous monitoring in the Snyk dashboard
- `snyk ignore --id=<vuln-id>` — suppress a known/accepted finding with a reason
- `snyk fix` — auto-apply available upgrades (where supported)

## Reference Files

- `references/code-security.md` — SAST scanning + secure code writing rules
- `references/dependency-security.md` — SCA/dependency scanning workflow
- `references/container-security.md` — Container image scanning workflow
- `references/iac-security.md` — Terraform, Kubernetes, and IaC scanning workflow
