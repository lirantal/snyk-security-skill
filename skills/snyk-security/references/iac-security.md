# IaC Security — snyk iac test

Snyk IaC scans infrastructure-as-code files for security misconfigurations — things like
publicly exposed storage buckets, missing encryption, overly permissive IAM policies,
unencrypted secrets, and containers running as root. It catches problems before they
reach production.

## Supported Formats

| Format | File patterns |
|---|---|
| Terraform | `*.tf`, `*.tf.json` |
| Kubernetes | `*.yaml`, `*.yml` (deployment, service, pod, etc.) |
| CloudFormation | `*.yaml`, `*.yml`, `*.json` (CF templates) |
| ARM Templates | `*.json` (Azure Resource Manager) |
| Helm Charts | `Chart.yaml` + templates directory |
| Serverless Framework | `serverless.yml` |
| Docker Compose | `docker-compose.yml` |

## Running the Scan

```bash
# Scan a specific file
snyk iac test main.tf
snyk iac test k8s/deployment.yaml
snyk iac test cloudformation/template.yaml

# Scan all IaC files in a directory recursively
snyk iac test ./infrastructure/

# Scan the current directory
snyk iac test

# Output as JSON
snyk iac test --json
snyk iac test ./infra --json-file-output=iac-results.json

# Only fail on high/critical
snyk iac test --severity-threshold=high

# Associate with Snyk org
snyk iac test --org=<org-slug>
```

## Interpreting Results

Each finding includes:
- **Issue title** — e.g., "S3 bucket is publicly readable"
- **Severity** — Critical/High/Medium/Low
- **Path** — the exact resource and attribute in the file causing the issue
- **Rule ID** — Snyk's rule identifier (e.g., SNYK-CC-TF-1)
- **Remediation** — what to change and why

The path format is: `[resource_type].[resource_name] > [attribute_path]`
Example: `aws_s3_bucket.my_bucket > acl`

## Common IaC Findings and Fixes

### Terraform

**Publicly accessible S3 bucket:**
```hcl
# Vulnerable
resource "aws_s3_bucket" "data" {
  acl = "public-read"  # anyone can read
}

# Fixed
resource "aws_s3_bucket" "data" {
  # no public ACL; use explicit bucket policies for specific access
}
resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

**Unencrypted storage:**
```hcl
# Fixed: enable server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}
```

**Over-permissive IAM policy:**
```hcl
# Vulnerable
resource "aws_iam_policy" "bad" {
  policy = jsonencode({
    Statement = [{
      Action   = "*"     # wildcard actions
      Resource = "*"     # wildcard resources
      Effect   = "Allow"
    }]
  })
}

# Fixed: use least privilege
resource "aws_iam_policy" "good" {
  policy = jsonencode({
    Statement = [{
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "arn:aws:s3:::my-bucket/*"
      Effect   = "Allow"
    }]
  })
}
```

**Security group allowing all inbound traffic:**
```hcl
# Vulnerable
resource "aws_security_group_rule" "bad" {
  cidr_blocks = ["0.0.0.0/0"]
  from_port   = 0
  to_port     = 0
  protocol    = "-1"   # all traffic
}

# Fixed: restrict to specific ports and sources
resource "aws_security_group_rule" "good" {
  cidr_blocks = ["10.0.0.0/8"]   # internal only
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
}
```

### Kubernetes

**Container running as root:**
```yaml
# Vulnerable: no security context
spec:
  containers:
  - name: app
    image: myapp:1.0

# Fixed: explicit non-root user
spec:
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
```

**No resource limits:**
```yaml
# Vulnerable: no limits
resources: {}

# Fixed: set both requests and limits
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

**Privileged container:**
```yaml
# Vulnerable
securityContext:
  privileged: true

# Fixed
securityContext:
  privileged: false
  capabilities:
    drop:
    - ALL
```

**Missing network policy:**
Add a NetworkPolicy to restrict pod-to-pod communication:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

**Secret in environment variable (plaintext):**
```yaml
# Vulnerable
env:
- name: DB_PASSWORD
  value: "supersecret"   # plaintext in YAML

# Fixed: reference a Kubernetes Secret
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: password
```

## Drift Detection

Snyk IaC can also detect configuration drift — differences between your IaC definition
and what's actually deployed in the cloud:

```bash
snyk iac describe --only-unmanaged   # resources in cloud not in IaC
snyk iac describe --only-managed     # managed resources with drift
```

This requires cloud provider credentials (AWS, GCP, Azure).

## Fix Workflow

1. Run `snyk iac test ./infra` to get a full list of findings
2. Sort by severity — address Critical/High first
3. Make the fix in the IaC file (see examples above)
4. Re-run `snyk iac test` to confirm the issue is resolved
5. Review the change in a PR before applying to production
6. Apply with `terraform plan` + `terraform apply` or equivalent

## CI Integration

```bash
# Gate PRs on IaC security
snyk iac test ./terraform --severity-threshold=high

# Save results for audit trail
snyk iac test ./terraform --json-file-output=iac-scan.json || true
```

## Tips

- Run IaC scans in your pull request pipeline to catch misconfigurations before they
  land in the main branch — much cheaper than fixing in production
- Use `--severity-threshold=high` in CI and review Medium/Low in the Snyk dashboard
- Snyk IaC rules cover 100+ checks for AWS, GCP, Azure, and Kubernetes
- For Terraform: scan the `.tf` source files, not the generated plan (though `snyk iac test tfplan.json` also works for plan files)
