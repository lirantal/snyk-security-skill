# Container Security — snyk container test

Snyk Container scans Docker images and container registries for vulnerabilities in OS
packages (Alpine, Debian, Ubuntu, RHEL, etc.) and application dependencies bundled into
the image. It also analyzes Dockerfiles for security misconfigurations.

## Running the Scan

```bash
# Scan a local image by name:tag
snyk container test nginx:latest
snyk container test myapp:1.0.0

# Scan a specific registry image
snyk container test gcr.io/myproject/myimage:latest
snyk container test 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# Scan a distroless image
snyk container test gcr.io/distroless/base

# Scan a local archive (saved with docker save)
snyk container test docker-archive:container.tar

# Include the Dockerfile for enhanced analysis
snyk container test myapp:latest --file=Dockerfile

# Output as JSON
snyk container test myapp:latest --json
snyk container test myapp:latest --json-file-output=container-results.json

# Fail CI only on high/critical
snyk container test myapp:latest --severity-threshold=high

# Associate with Snyk org
snyk container test myapp:latest --org=<org-slug>
```

## What Gets Scanned

1. **OS packages** — vulnerabilities in the base OS packages (apt, apk, rpm, etc.)
2. **Application packages** — npm, pip, Maven, etc. packages installed in the image
3. **Dockerfile misconfigurations** — when `--file=Dockerfile` is passed

## Dockerfile Security Best Practices

When Snyk flags Dockerfile issues, or when reviewing a Dockerfile, apply these practices:

### Use minimal base images
```dockerfile
# Prefer slim or distroless over full OS images
FROM node:20-alpine        # smaller attack surface than node:20
FROM gcr.io/distroless/nodejs20  # no shell, minimal packages
```

### Don't run as root
```dockerfile
# Create and use a non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### Avoid secrets in image layers
```dockerfile
# Wrong: secret baked into image
RUN export API_KEY=abc123 && ./configure

# Right: pass at runtime
# docker run -e API_KEY=... myapp
```

### Pin base image versions
```dockerfile
# Wrong: floating tag
FROM node:latest

# Right: pinned to a specific digest or version
FROM node:20.11.0-alpine3.19
```

### Minimize installed packages
```dockerfile
# Only install what you need, clean up in the same layer
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*
```

### Use multi-stage builds to reduce the final image
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Runtime stage — only copy what's needed
FROM gcr.io/distroless/nodejs20
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["dist/index.js"]
```

## Interpreting Results

Container scan results include:
- **OS vulnerabilities** — package name, installed version, fixed version, CVE ID
- **Application vulnerabilities** — same as `snyk test` results for app deps
- **Dockerfile issues** — misconfiguration rule ID, description, recommended fix

For each vulnerability, Snyk shows:
- The **base image** that introduced the vulnerability
- Whether a **base image upgrade** would fix it (with specific recommendations)
- The **minimal base image** Snyk recommends to reduce total vulnerability count

## Fix Workflow

### Fix OS vulnerabilities
The most effective fix is usually upgrading the base image:
```dockerfile
# Snyk output will suggest: "Upgrade to node:20.11-alpine3.19" (for example)
FROM node:20.11-alpine3.19   # upgrade from whatever you had
```

Then rebuild and re-scan:
```bash
docker build -t myapp:1.0.1 .
snyk container test myapp:1.0.1 --file=Dockerfile
```

### Fix application vulnerabilities
Same as SCA — upgrade the vulnerable package in your package manifest,
then rebuild the image.

### Monitor for new vulnerabilities
```bash
snyk container monitor myapp:latest --org=<org-slug>
```
Snyk will alert you when new vulnerabilities are published that affect your image.

## CI/CD Integration

```bash
# In your build pipeline, after building the image:
docker build -t myapp:$TAG .

# Test and fail on high/critical
snyk container test myapp:$TAG --file=Dockerfile --severity-threshold=high

# Monitor the image in production
snyk container monitor myapp:$TAG --org=my-org --project-name=myapp-production
```

## Tips

- Always pass `--file=Dockerfile` when the Dockerfile is available — it gives Snyk
  additional context about the build and enables Dockerfile-specific checks
- Regularly rebuild images to pick up OS package patches, not just when your code changes
- Use `snyk container test` in both build pipeline (fail fast) and deployment pipeline
  (gate on production)
- Distroless images have the smallest attack surface but require multi-stage builds
