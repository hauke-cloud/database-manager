# Docker and Helm Setup Summary

## Overview

Complete Docker and Helm chart setup for `database-manager`, following all assumptions from the `hauke-cloud/inpacken-un-af-dor-mit` GitHub Action.

## Files Created

### Database Manager

```
database-manager/
├── Dockerfile                                    # Multi-platform container build
├── .dockerignore                                 # Build context optimization
└── deployments/
    └── helm/
        └── database-manager/
            ├── Chart.yaml                        # Helm chart metadata
            ├── values.yaml                       # Default configuration
            └── templates/
                ├── _helpers.tpl                  # Template helpers
                ├── serviceaccount.yaml           # Kubernetes service account
                ├── clusterrole.yaml              # RBAC permissions
                ├── clusterrolebinding.yaml       # Role binding
                ├── leader-election-role.yaml     # Leader election RBAC
                ├── leader-election-rolebinding.yaml
                └── deployment.yaml               # Operator deployment
```

## Dockerfile Features

### Build Arguments (Automatically Provided by Action)

```dockerfile
ARG VERSION=dev       # Git tag or generated version
ARG COMMIT=unknown    # Git commit SHA
ARG DATE=unknown      # Build timestamp
ARG BRANCH=unknown    # Git branch name
```

### Multi-Platform Support

Builds for:
- `linux/amd64` (Intel/AMD x64)
- `linux/arm64` (ARM 64-bit, e.g., Apple Silicon, AWS Graviton)

### Security Features

- **Distroless base image**: Minimal attack surface
- **Non-root user**: Runs as UID 65532
- **Multi-stage build**: Only runtime artifacts in final image
- **No shell**: Distroless has no shell or package managers

### OCI Labels

```dockerfile
org.opencontainers.image.version
org.opencontainers.image.revision
org.opencontainers.image.created
org.opencontainers.image.source
org.opencontainers.image.title
org.opencontainers.image.description
```

## Helm Chart Features

### Configuration (values.yaml)

**Image:**
- Repository: `ghcr.io/hauke-cloud/database-manager`
- Pull policy: `IfNotPresent`
- Tag: Defaults to chart `appVersion`

**Resources:**
```yaml
limits:
  cpu: 500m
  memory: 128Mi
requests:
  cpu: 10m
  memory: 64Mi
```

**Security Context:**
- Run as non-root
- Read-only root filesystem
- Drop all capabilities
- Seccomp profile: RuntimeDefault

**Operator Configuration:**
- Leader election: Enabled
- Metrics port: 8080
- Health probe port: 8081
- Logging: JSON format, info level

### RBAC Permissions

**ClusterRole grants access to:**
- `database.hauke.cloud/v1alpha1` Database CRDs (full CRUD)
- Secrets (read-only for credentials)

**Leader Election Role:**
- ConfigMaps (for leader election)
- Leases (coordination.k8s.io)
- Events (for status updates)

### Probes

**Liveness:**
- Path: `/healthz`
- Initial delay: 15s
- Period: 20s

**Readiness:**
- Path: `/readyz`
- Initial delay: 5s
- Period: 10s

## Compliance with GitHub Action Assumptions

✅ **Dockerfile Location**: `./Dockerfile` (repository root)  
✅ **Helm Chart Path**: `./deployments/helm/database-manager/`  
✅ **Build Args**: VERSION, COMMIT, DATE, BRANCH accepted  
✅ **Multi-Platform**: TARGETOS, TARGETARCH ARGs present  
✅ **Chart.yaml**: Contains required fields (name, version, appVersion)  
✅ **values.yaml**: Contains image.repository and image.tag  
✅ **.dockerignore**: Optimizes build context  

## Testing

### Helm Lint

```bash
cd database-manager
helm lint deployments/helm/database-manager/
```

Result: ✅ Chart passes linting

### Template Rendering

```bash
helm template test deployments/helm/database-manager/ --set image.tag=v0.1.0
```

Result: ✅ Templates render correctly

### Local Docker Build

```bash
docker build \
  --build-arg VERSION=dev \
  --build-arg COMMIT=$(git rev-parse HEAD) \
  --build-arg DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  -t database-manager:dev \
  .
```

## CI/CD Workflow Integration

The GitHub Action (`.github/workflows/ci.yml`) will:

1. **Build Phase:**
   - Set up Go 1.25
   - Download dependencies
   - Run tests with race detector
   - Run linting (gofmt, go vet)
   - Generate manifests and CRDs

2. **Docker Phase:**
   - Build multi-platform images
   - Pass VERSION, COMMIT, DATE, BRANCH args
   - Tag images based on git ref
   - Push to `ghcr.io/hauke-cloud/database-manager`

3. **Helm Phase:**
   - Package chart
   - Update Chart.yaml version
   - Update values.yaml image.tag
   - Push to `oci://ghcr.io/hauke-cloud/charts/database-manager`

4. **Release Phase** (on tags):
   - Create GitHub Release
   - Attach release notes
   - Mark with version from tag

## Image Tagging Strategy

### Branch Builds (main, develop)
- `main` → `latest`
- `develop` → `develop`
- Feature branches → `branch-name`

### Tag Builds (v1.2.3)
- `v1.2.3` → `1.2.3`, `1.2`, `1`, `latest` (if main)
- `v0.1.0` → `0.1.0`, `0.1` (no `0` major tag)
- `v1.2.3-rc.1` → `1.2.3-rc.1` (prerelease)

### Pull Requests
- `pr-123` → No push by default

## Installation

### Via Helm (after first release)

```bash
# Add Helm repository (OCI)
helm install database-manager \
  oci://ghcr.io/hauke-cloud/charts/database-manager \
  --version 0.1.0 \
  --namespace database-manager-system \
  --create-namespace
```

### With Custom Values

```bash
helm install database-manager \
  oci://ghcr.io/hauke-cloud/charts/database-manager \
  --version 0.1.0 \
  --set replicaCount=2 \
  --set resources.limits.memory=256Mi \
  --set logging.level=debug
```

### Using values.yaml

```bash
# Create custom-values.yaml
cat > custom-values.yaml << EOF
replicaCount: 1

image:
  tag: "0.1.0"

resources:
  limits:
    cpu: 1000m
    memory: 256Mi

logging:
  level: info
  format: json

operator:
  leaderElection: true
EOF

helm install database-manager \
  oci://ghcr.io/hauke-cloud/charts/database-manager \
  --values custom-values.yaml
```

## Upgrade

```bash
helm upgrade database-manager \
  oci://ghcr.io/hauke-cloud/charts/database-manager \
  --version 0.2.0
```

## Uninstall

```bash
# Uninstall release (keeps CRDs by default)
helm uninstall database-manager

# To also remove CRDs
kubectl delete crd databases.database.hauke.cloud
```

## Troubleshooting

### Build Fails

**Check:**
- Go 1.25 is available
- All dependencies in go.mod resolve
- `cmd/main.go` exists

### Docker Build Fails

**Check:**
- Dockerfile syntax
- All COPY paths exist
- Build context size (use .dockerignore)

### Helm Lint Errors

```bash
helm lint deployments/helm/database-manager/
```

### Template Rendering Issues

```bash
helm template test deployments/helm/database-manager/ --debug
```

### Missing RBAC Permissions

The operator needs:
- Read secrets (for database credentials)
- Full CRUD on Database CRDs
- Leader election permissions

Check ClusterRole and ClusterRoleBinding are created.

## Best Practices

✅ **Version Pinning**: Pin base image versions in production  
✅ **Resource Limits**: Always set memory and CPU limits  
✅ **Security Context**: Run as non-root, read-only filesystem  
✅ **Health Probes**: Configure liveness and readiness  
✅ **Logging**: Use structured JSON logging  
✅ **Monitoring**: Enable metrics endpoint  
✅ **High Availability**: Consider `replicaCount > 1` with leader election  

## Next Steps

1. **Commit Files:**
   ```bash
   git add Dockerfile .dockerignore deployments/
   git commit -m "Add Docker and Helm chart for database-manager"
   ```

2. **Push to GitHub:**
   ```bash
   git push origin main
   ```

3. **Create Release:**
   ```bash
   git tag -a v0.1.0 -m "Initial release"
   git push origin v0.1.0
   ```

4. **Monitor CI/CD:**
   - Check GitHub Actions workflow
   - Verify image pushed to ghcr.io
   - Verify Helm chart pushed to OCI registry
   - Check GitHub Release created

5. **Install and Test:**
   ```bash
   helm install database-manager \
     oci://ghcr.io/hauke-cloud/charts/database-manager \
     --version 0.1.0
   ```

## References

- **GitHub Action**: https://github.com/hauke-cloud/inpacken-un-af-dor-mit
- **Action Assumptions**: https://github.com/hauke-cloud/inpacken-un-af-dor-mit/blob/main/ASSUMPTIONS.md
- **Helm Documentation**: https://helm.sh/docs/
- **Distroless Images**: https://github.com/GoogleContainerTools/distroless
