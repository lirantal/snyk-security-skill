# Dependency Security (SCA) — snyk test

Snyk's Software Composition Analysis (SCA) scans open-source dependencies and libraries
for known vulnerabilities (CVEs). It reads your package manifest (package.json, requirements.txt,
pom.xml, go.mod, Gemfile, etc.) and checks each dependency — including transitive ones —
against Snyk's vulnerability database.

## Running the Scan

```bash
# Scan the current project (auto-detects manifest)
snyk test

# Scan a specific manifest file
snyk test --file=package.json
snyk test --file=requirements.txt
snyk test --file=pom.xml

# Scan all manifests in the repo recursively
snyk test --all-projects

# Scan all manifests with custom depth (default is 4)
snyk test --all-projects --detection-depth=6

# Include dev dependencies (excluded by default)
snyk test --dev

# Specify package manager explicitly
snyk test --package-manager=npm

# Output results as JSON
snyk test --json
snyk test --json-file-output=deps-results.json

# Only fail on high/critical (useful for CI)
snyk test --severity-threshold=high

# Associate with a Snyk org for policy enforcement
snyk test --org=<your-org-slug>
```

## Supported Package Managers

| Ecosystem | Manifest file | Notes |
|---|---|---|
| npm / Yarn | package.json, yarn.lock | Run `npm install` first |
| Python | requirements.txt, Pipfile, pyproject.toml | Virtual env recommended |
| Maven | pom.xml | Run `mvn install` first |
| Gradle | build.gradle | Run `./gradlew dependencies` first |
| Go | go.mod, go.sum | Run `go mod download` first |
| Ruby | Gemfile, Gemfile.lock | Run `bundle install` first |
| .NET | *.csproj, packages.config | Restore packages first |
| PHP | composer.json | Run `composer install` first |
| Rust | Cargo.toml | Run `cargo build` first |

**Important:** Snyk reads the dependency tree from lock files and installed node_modules/vendor
directories. Always build/install the project before scanning so the full tree is visible.
If results seem incomplete, run the appropriate install command first.

## Interpreting Results

Each vulnerability finding includes:
- **CVE/CWE identifier** and Snyk vulnerability ID
- **Affected package** and version range
- **Fixed in version** (if a fix is available)
- **Severity** (Critical/High/Medium/Low) with CVSS score
- **Exploit maturity** — is there a known exploit in the wild?
- **Upgrade path** — direct or transitive dependency

Snyk groups findings into three categories:
1. **Upgradeable** — the fix is a direct dependency upgrade you control
2. **Patchable** — Snyk has a patch available (less common)
3. **Not fixable** — no fix yet; consider alternatives or accept the risk

## Fix Workflow

### Option 1: Upgrade (preferred)
When Snyk shows an upgrade path, update the dependency:
```bash
# npm
npm install <package>@<fixed-version>

# pip
pip install <package>==<fixed-version>
# update requirements.txt accordingly

# Go
go get <module>@<fixed-version>
go mod tidy
```

Then re-run `snyk test` to confirm the vulnerability is resolved.

### Option 2: Auto-fix
```bash
snyk fix     # applies available upgrades automatically (where supported)
```

### Option 3: Ignore (use sparingly)
When a vulnerability is not exploitable in your context:
```bash
snyk ignore --id=<SNYK-ID> --reason="Not exploitable: X" --expiry=2025-12-31
```
This creates a `.snyk` policy file. Always include a reason and expiry date.

### Option 4: Monitor for future fixes
```bash
snyk monitor    # uploads snapshot to Snyk dashboard for ongoing monitoring
```
Snyk will alert you when a fix becomes available.

## Transitive Dependencies

Most vulnerabilities are in transitive (indirect) dependencies — packages your direct
dependencies depend on. These are harder to fix. Options:
- Upgrade the direct dependency to a version that pulls in the fixed transitive version
- Use `overrides` (npm) or `resolutions` (Yarn) to force a specific version
- Replace the vulnerable library with an alternative

## CI Integration Recommendations

```bash
# Fail the build on high/critical vulnerabilities
snyk test --severity-threshold=high

# Continue build but save results for review
snyk test --json-file-output=snyk-results.json || true
snyk monitor    # always monitor even if test passes
```

## Common Questions

**"I have hundreds of vulnerabilities, where do I start?"**
Start with Critical and High severity, and within those, prioritize vulnerabilities with
known exploits (Exploit Maturity: "Proof of Concept" or higher). Filter with:
`snyk test --severity-threshold=high`

**"This vulnerability is in a test dependency, do I need to fix it?"**
Test dependencies don't run in production, but if they're included in your build output
or CI environment, they can still be exploited. Use judgment — generally fix High/Critical
even in dev deps.

**"The vulnerable package is deeply nested, I can't upgrade it."**
Check if your direct dependency has released a version that resolves it. If not, evaluate
whether the vulnerability is actually reachable in your use of the package.
