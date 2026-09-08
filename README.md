<!-- llm-readme-management spec=1 commit=be2690bad0829ff2bdbc7bed29e7eda67c1116ee template=golang model=qwen3.8-27b-q4 digest=b3d4b07f2e19 generated=2026-09-08T13:37:24Z -->
<a href="https://hauke.cloud" target="_blank"><img src="https://img.shields.io/badge/home-hauke.cloud-brightgreen" alt="hauke.cloud" style="display: block;" /></a>
<a href="https://github.com/hauke-cloud" target="_blank"><img src="https://img.shields.io/badge/github-hauke.cloud-blue" alt="hauke.cloud Github Organisation" style="display: block;" /></a>
<a href="https://github.com/hauke-cloud/llm-readme-management" target="_blank"><img src="https://img.shields.io/badge/template-golang-orange" alt="Repository type - golang" style="display: block;" /></a>


# Database Manager


<img src="https://raw.githubusercontent.com/hauke-cloud/.github/main/resources/img/organisation-logo-small.png" alt="hauke.cloud logo" width="109" height="123" align="right">


<llm header hint="Name the Go module path and say whether this is a service, a CLI or a library.">

This is a Go service (module `github.com/hauke-cloud/database-manager`) that runs as a Kubernetes operator in a hauke.cloud IoT cluster. It watches `Database` custom resources and manages PostgreSQL/TimescaleDB connection pools for storing IoT sensor measurements. It is for cluster operators who need to provision and monitor the database connections backing their sensor infrastructure.

</llm>


## :book: Description

<llm description>

`database-manager` is a Kubernetes operator that manages PostgreSQL and TimescaleDB connections for storing IoT sensor measurements. It watches `Database` custom resources (group `iot.hauke.cloud`, version `v1alpha1`) and, for each one, opens a GORM connection pool to the referenced database, auto-migrates the tables for the sensor types listed in the resource, and reports connection status back to the CR's status subresource.

The operator runs as a single Deployment inside a hauke.cloud IoT cluster. It self-installs its CRD at startup, reads credentials from Kubernetes Secrets, and routes incoming measurement payloads to the first connected database whose `supportedSensorTypes` includes the relevant sensor type.

- Reconciles `Database` CRs: builds a TLS-aware DSN from the spec and a referenced password Secret, then opens a pooled GORM connection.
- Auto-migrates four measurement tables on connect: `moisture_measurements`, `valve_measurements`, `water_level_measurements`, `room_measurements`.
- Routes sensor payloads via `StoreMeasurement` to the matching database handler.
- Reports `Connected` or `Error` state (with a 30 s requeue) in the CR status; closes the connection on resource deletion.

The reusable CRD types live in the external `github.com/hauke-cloud/kubernetes-iot-api` module; everything in this repository under `internal/` is non-importable.

</llm>


## :clipboard: Requirements

<llm requirements hint="Give the Go version from the go directive in go.mod. Mention Docker only if the repository actually builds an image.">

- Go 1.25.3 (pinned in `go.mod`).
- `controller-gen` v0.20.1, installed automatically by `make controller-gen`.
- A reachable Kubernetes cluster with a valid kubeconfig. The operator requires RBAC for `iot.hauke.cloud/databases` (including `/status` and `/finalizers`), `secrets` read, and `apiextensions.k8s.io/customresourcedefinitions` create/get/list/update.
- A PostgreSQL or TimescaleDB instance, with a Kubernetes Secret in the same namespace as the `Database` CR holding the password under key `password`; optional CA and client-certificate Secrets for mTLS.
- Docker (or another container runtime) for `make docker-build`.
- `kubectl` for `make install` and `make uninstall`.
- Helm for the OCI chart install path.
- `pre-commit` for contributors.

</llm>


## 🚀 Getting started

<llm getting_started hint="Cover go build, go run and go test with the real package paths. If a Makefile or Taskfile exists, prefer its targets over raw go commands.">

1. Clone the repository and enter the project directory.

```bash
git clone https://github.com/hauke-cloud/database-manager.git
cd database-manager
```

2. Build the operator binary into `bin/manager` (this also fetches Go module dependencies).

```bash
make build
```

3. Run the operator against a Kubernetes cluster; a valid kubeconfig must be available in your environment.

```bash
make run
```

4. Run the full test suite with coverage.

```bash
make test
```

</llm>


## :airplane: Usage

<llm usage hint="For a library, show a small import-and-call example using real exported identifiers. For a service or CLI, show how it is started and the flags or subcommands it accepts.">

Once the operator is running in your cluster, you interact with it by creating `Database` custom resources and checking their status.

**Deploy the operator**

The Helm chart is the supported deployment path. It creates the Deployment, ServiceAccount, RBAC, and leader-election resources:

```bash
helm install database-manager \
  oci://ghcr.io/hauke-cloud/charts/database-manager \
  --version 0.1.0 \
  --namespace database-manager-system \
  --create-namespace
```

The operator self-installs the `databases.iot.hauke.cloud` CRD at startup, so no separate `kubectl apply` is needed.

**Create a Database resource**

Point the operator at a PostgreSQL or TimescaleDB instance. The password must live in a Kubernetes Secret (key `password`) in the same namespace as the resource:

```yaml
apiVersion: iot.hauke.cloud/v1alpha1
kind: Database
metadata:
  name: sensor-db
  namespace: database-manager-system
spec:
  host: postgres.example.internal
  database: iot
  username: iot_writer
  port: 5432
  passwordSecretRef:
    name: sensor-db-password
  sslMode: require
  supportedSensorTypes:
    - moisture
    - water_level
    - valve
    - room
  maxConnections: 10
  minConnections: 2
```

On the first successful reconcile the operator opens a GORM connection pool, runs `AutoMigrate` for each supported sensor type, and sets `status.connectionState` to `Connected`.

**Check connection status**

```bash
kubectl get databases -n database-manager-system
```

The short names `db` and `dbs` also work. If the connection fails the status shows `Error` with the message and the reconciler requeues after 30 seconds.

</llm>


## :wrench: Configuration

<llm configuration hint="Environment variables and CLI flags, taken from the flag definitions or the config struct.">

The operator binary (`cmd/main.go`) accepts four CLI flags:

| Flag | Default | Description |
|------|---------|-------------|
| `--metrics-bind-address` | `:8080` | Prometheus metrics listen address |
| `--health-probe-bind-address` | `:8081` | Liveness/readiness probe address |
| `--leader-elect` | `false` | Enable leader election (ID `database-manager.hauke.cloud`) |
| `--log-level` | `info` | Zap level: `debug`, `info`, `warn`, `error` |

Each `Database` CR (`databases.iot.hauke.cloud/v1alpha1`) carries the connection parameters. The full schema is in `config/crd/iot.hauke.cloud_databases.yaml`; the table below lists the fields the reconciler reads at connect time. `clientCertSecretRef`, `caSecretRef`, `batchSize`, and `batchTimeout` are also defined in the CRD but are omitted here (the last two have no implementation yet).

| Field | Type | Default | Required | Description |
|-------|------|---------|----------|-------------|
| `host` | string | — | yes | PostgreSQL/TimescaleDB host |
| `database` | string | — | yes | Database name |
| `username` | string | — | yes | DB user |
| `port` | int | `5432` | no | DB port |
| `passwordSecretRef` | object | — | no | Secret ref (key `password`) holding the DB password |
| `sslMode` | enum | `require` | no | `disable`, `require`, `verify-ca`, `verify-full` |
| `supportedSensorTypes` | []string | — | yes (minItems 1) | Sensor types this database serves |
| `maxConnections` | int | `10` | no | Pool max open conns (max 100) |
| `minConnections` | int | `2` | no | Pool max idle conns (max 10) |

The Helm chart (`deployments/helm/database-manager/values.yaml`) exposes `replicaCount` (1), `image.*`, `operator.leaderElection` (true), `operator.metrics.port` (8080), `operator.health.port` (8081), `rbac.create` (true), `logging.level` (info), and `logging.format` (json). See `values.yaml` for the remaining keys (`resources`, `nodeSelector`, `tolerations`, `affinity`, `podAnnotations`).

</llm>


## :hammer: Development

<llm development hint="Include go test, go vet and gofmt only where the CI workflows actually run them.">

Before pushing, regenerate the checked-in generated files so CI does not reject your diff:

```bash
make generate
make manifests
```

`make generate` runs controller-gen deepcopy for `./...`; `make manifests` regenerates the CRD YAML and RBAC from the `kubernetes-iot-api` types. Both are pinned to controller-gen v0.20.1.

Run the full test suite locally:

```bash
make test
```

This chains `manifests generate fmt vet` and then `go test ./... -coverprofile cover.out`. CI additionally runs the tests with the race detector:

```bash
go test -v -race -coverprofile=coverage.out -covermode=atomic ./...
```

For formatting and static analysis:

```bash
make fmt
make vet
```

CI's lint step is `gofmt -s -l .` followed by `go vet ./...`. Note that `gofmt -s -l` only *lists* unformatted files; it does not fail the build on its own, but `go vet` will.

Install the pre-commit hooks (gitleaks and basic checks) before your first commit:

```bash
pre-commit install
pre-commit run --all-files
```

PR titles are validated by a workflow: the type must be one of `fix`, `feat`, `docs`, `ci`, or `chore`, and the subject must start with an uppercase letter.

</llm>


## 📄 License

This Project is licensed under the GNU General Public License v3.0

- see the [LICENSE](LICENSE) file for details.


## :coffee: Contributing

To become a contributor, please check out the [CONTRIBUTING](CONTRIBUTING.md) file.


## :email: Contact

For any inquiries or support requests, please open an issue in this
repository or contact us at [contact@hauke.cloud](mailto:contact@hauke.cloud).
