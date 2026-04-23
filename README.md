

<a href="https://hauke.cloud" target="_blank"><img src="https://img.shields.io/badge/home-hauke.cloud-brightgreen" alt="hauke.cloud" style="display: block;" /></a>
<a href="https://github.com/hauke-cloud" target="_blank"><img src="https://img.shields.io/badge/github-hauke.cloud-blue" alt="hauke.cloud Github Organisation" style="display: block;" /></a>

# Database Manager

<img src="https://raw.githubusercontent.com/hauke-cloud/.github/main/resources/img/organisation-logo-small.png" alt="hauke.cloud logo" width="109" height="123" align="right">

A Kubernetes operator that manages PostgreSQL/TimescaleDB database connections for sensor measurement storage. This operator watches `Database` custom resources and establishes connections to PostgreSQL databases, providing a centralized way to manage database connectivity across multiple applications.

## Overview

The Database Manager operator is part of the hauke.cloud IoT infrastructure and works alongside other components like mqtt-sensor-exporter. It provides:

- **Declarative Database Configuration**: Define database connections using Kubernetes CRDs
- **Automatic Connection Management**: Establishes and maintains database connections
- **Secure Credentials**: Store database credentials in Kubernetes Secrets
- **Multi-Database Support**: Connect to multiple databases with different configurations
- **TLS/SSL Support**: Full support for encrypted connections with client certificates
- **Sensor Type Routing**: Route different sensor types to different databases
- **Connection Pooling**: Configurable connection pooling for performance
- **Status Monitoring**: Track connection status and measurement statistics

## Architecture

The operator uses the shared `database-api` library for CRD definitions, allowing multiple applications to consume the same Database resources without importing controller dependencies.

### Components

- **database-api**: Shared Go library containing the Database CRD types
- **database-manager**: Controller that watches Database resources and manages connections
- **Applications** (e.g., mqtt-sensor-exporter): Import database-api and use Database resources

## Getting Started

### Prerequisites

- Kubernetes cluster (1.28+)
- kubectl configured
- PostgreSQL or TimescaleDB instance

### Installation

1. **Install CRDs:**

```bash
kubectl apply -f config/crd/database.hauke.cloud_databases.yaml
```

2. **Deploy the operator:**

Development mode:
```bash
make run
```

Production deployment:
```bash
make docker-build docker-push IMG=<your-registry>/database-manager:tag
kubectl apply -k config/manager
```

### Quick Start

1. **Create database credentials secret:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: timescaledb-credentials
  namespace: default
type: Opaque
stringData:
  password: "your-database-password"
```

2. **Create a Database resource:**

```yaml
apiVersion: database.hauke.cloud/v1alpha1
kind: Database
metadata:
  name: moisture-db
  namespace: default
spec:
  host: "timescaledb.default.svc.cluster.local"
  port: 5432
  database: "sensors"
  username: "sensor_writer"
  passwordSecretRef:
    name: timescaledb-credentials
  sslMode: "require"
  supportedSensorTypes:
    - "moisture"
    - "water_level"
  maxConnections: 10
  minConnections: 2
  batchSize: 100
  batchTimeout: 10
```

3. **Check connection status:**

```bash
kubectl get databases
```

## Configuration

### Database Spec

- **host**: Database hostname or IP address
- **port**: Database port (default: 5432)
- **database**: Database name
- **username**: Database username
- **passwordSecretRef**: Reference to Secret containing password
- **clientCertSecretRef**: Reference to Secret containing client certificate (for mutual TLS)
- **sslMode**: SSL mode (disable, require, verify-ca, verify-full)
- **caSecretRef**: Reference to Secret containing CA certificate
- **supportedSensorTypes**: List of sensor types this database handles
- **maxConnections**: Maximum connection pool size
- **minConnections**: Minimum connection pool size
- **batchSize**: Number of measurements to batch before writing
- **batchTimeout**: Maximum time to wait before writing partial batch (seconds)

## Integration

### Using database-api in Your Application

To use Database resources in your Go application:

1. **Import the shared API:**

```go
import databasev1alpha1 "github.com/hauke-cloud/database-api/api/v1alpha1"
```

2. **Add to your go.mod:**

```go
require (
    github.com/hauke-cloud/database-api v0.0.0
)

replace github.com/hauke-cloud/database-api => ../database-api  // for local development
```

3. **Register the scheme:**

```go
utilruntime.Must(databasev1alpha1.AddToScheme(scheme))
```

4. **Watch Database resources:**

```go
func (r *YourReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        Watches(&databasev1alpha1.Database{}, handler.EnqueueRequestsFromMapFunc(r.findRelatedResources)).
        Complete(r)
}
```

## Development

### Building

```bash
# Generate manifests and code
make manifests generate

# Build binary
make build

# Run locally
make run
```

### Project Structure

```
database-manager/
├── cmd/main.go              # Main entry point
├── internal/
│   ├── controller/          # Database controller
│   └── database/            # Database connection management
│       ├── manager.go       # Connection manager
│       ├── models.go        # GORM models
│       └── *_handler.go     # Sensor-specific handlers
├── config/
│   ├── crd/                 # Generated CRDs (from database-api)
│   └── manager/             # Deployment manifests
└── Makefile

database-api/                # Shared API library
└── api/v1alpha1/
    ├── groupversion_info.go
    ├── database_types.go
    └── zz_generated.deepcopy.go
```



## 📄 License

This Project is licensed under the GNU General Public License v3.0

- see the [LICENSE](LICENSE) file for details.


## :coffee: Contributing

To become a contributor, please check out the [CONTRIBUTING](CONTRIBUTING.md) file.


## :email: Contact

For any inquiries or support requests, please open an issue in this
repository or contact us at [contact@hauke.cloud](mailto:contact@hauke.cloud).


## Best Practices

This architecture follows Kubernetes ecosystem best practices:

- **Separation of Concerns**: API types are separate from controller logic, similar to how `k8s.io/api` is separate from `k8s.io/kubernetes`.
- **Shared Library Pattern**: Multiple applications can import `database-api` without pulling in controller dependencies, keeping their binaries lean.
- **Declarative Configuration**: Database connections are managed declaratively through Kubernetes resources.
- **GitOps Ready**: All configuration can be version-controlled and deployed via GitOps tools like ArgoCD or Flux.
