# Snyk Security Skills

A collection of agent skills for Snyk security scanning — designed for Claude Code
and other coding agents that support the skills specification.

## What are skills?

Skills are packaged instructions that give AI coding agents domain-specific knowledge
and repeatable workflows. When a skill is installed, the agent knows when to use it
and how to carry out multi-step tasks that would otherwise require the user to guide
every step.

This repository hosts skills focused on application security using the
[Snyk CLI](https://docs.snyk.io/snyk-cli).

## Skills

### `snyk-security`

Runs Snyk CLI security scans across all four Snyk domains and drives findings to
resolution — not just reporting, but actually fixing the issues found.

| Domain | What it scans | CLI command |
|---|---|---|
| Code Security (SAST) | Source code for injection, XSS, hardcoded secrets, etc. | `snyk code test` |
| Dependency Security (SCA) | Open-source libraries for known CVEs | `snyk test` |
| Container Security | Docker images for OS and app vulnerabilities | `snyk container test` |
| IaC Security | Terraform, Kubernetes, CloudFormation for misconfigurations | `snyk iac test` |

**Location:** [`skills/snyk-security/`](skills/snyk-security/)

## Repository structure

```
skills/
└── snyk-security/
    ├── SKILL.md                     Main skill — instructions, use cases, examples, troubleshooting
    └── references/
        ├── code-security.md         SAST workflow + secure code writing rules
        ├── dependency-security.md   SCA workflow, package managers, fix strategies
        ├── container-security.md    Container scanning, Dockerfile best practices
        └── iac-security.md          Terraform, Kubernetes, CloudFormation fix patterns
```

Skills use progressive disclosure: the agent loads `SKILL.md` when the skill triggers,
then reads only the relevant reference file for the domain it's working in.

## Prerequisites

- [Snyk CLI](https://docs.snyk.io/snyk-cli/install-or-update-the-snyk-cli) installed
- A Snyk account (free tier works for open-source projects)
- Authenticated: `snyk auth`

## Contributing

Contributions are welcome — new skills, improvements to existing ones, or additional
reference material. Open an issue or pull request.

## License

MIT
