# Snyk Security Skills

Stop manually chasing security vulnerabilities across your codebase, dependencies,
containers, and infrastructure. The `snyk-security` skill gives Claude Code the
knowledge to run Snyk scans, understand what's actually exploitable, and fix the
issues — all in one conversation, without you having to direct every step.

## What you get

Instead of running `snyk test`, reading a wall of CVEs, and figuring out which ones
matter and how to fix them yourself, you can say:

> "Scan this project for security issues and fix anything critical."

Claude will run the right Snyk commands for your stack, prioritize findings by
exploitability, apply fixes (dependency upgrades, code changes, Dockerfile corrections,
IaC patches), verify each fix resolves the issue, and loop until the project is clean.

The skill covers all four Snyk domains:

| Domain | What gets fixed |
|---|---|
| **Code Security (SAST)** | Injection flaws, XSS, hardcoded secrets, insecure crypto, path traversal in your source code |
| **Dependency Security (SCA)** | Known CVEs in npm, pip, Maven, Go, Ruby, and other open-source packages |
| **Container Security** | Vulnerable OS packages and app dependencies in Docker images; Dockerfile misconfigurations |
| **IaC Security** | Misconfigured Terraform, Kubernetes YAML, CloudFormation, and Helm charts before they reach production |

## Installing the snyk-security skill

### 1. Get the skill

Clone this repository:

```bash
git clone https://github.com/lirantal/snyk-security-skills.git
```

Or download the latest release as a ZIP from the [Releases](../../releases) page
and unzip it locally.

### 2. Install in Claude Code

```bash
# From inside the cloned repository
claude skill install skills/snyk-security
```

Or install manually: open Claude Code settings, navigate to **Skills**, click
**Add skill**, and select the `skills/snyk-security` folder.

### 3. Verify it's active

Ask Claude:

> "What security scanning skills do you have available?"

Claude should confirm the `snyk-security` skill is loaded and describe what it can do.

### 4. Prerequisites

Before using the skill, make sure Snyk is installed and authenticated in your environment:

```bash
# Install the Snyk CLI
npm install -g snyk

# Authenticate (opens a browser window)
snyk auth
```

A free Snyk account works for open-source projects. Sign up at [snyk.io](https://snyk.io).

## Usage examples

```
"Do a full security scan of this project and fix whatever you find."

"Snyk flagged a critical vulnerability in express 4.18.1 — can you resolve it?"

"Scan my Dockerfile before we ship this release."

"Check our Terraform files for any security misconfigurations."

"Are there any hardcoded secrets or injection vulnerabilities in the src/ folder?"
```

## Repository structure

```
skills/
└── snyk-security/
    ├── SKILL.md                     Main skill — use cases, workflow steps, examples, troubleshooting
    └── references/
        ├── code-security.md         SAST scanning + secure code writing rules
        ├── dependency-security.md   SCA workflow, package managers, fix strategies
        ├── container-security.md    Container scanning, Dockerfile best practices
        └── iac-security.md          Terraform, Kubernetes, CloudFormation fix patterns
```

The skill uses progressive disclosure: Claude loads only `SKILL.md` when the skill
triggers, then reads the relevant reference file for whichever domain it's working in.

## Contributing

Contributions are welcome — new skills, improvements to existing ones, or additional
reference material for languages and ecosystems not yet covered. Open an issue or
pull request.

## License

MIT
