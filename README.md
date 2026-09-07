<!-- llm-readme-management spec=1 commit=be2690bad0829ff2bdbc7bed29e7eda67c1116ee template=golang model=qwen3.8-27b-q4 digest=b3d4b07f2e19 generated=2026-09-07T18:48:19Z -->
<a href="https://hauke.cloud" target="_blank"><img src="https://img.shields.io/badge/home-hauke.cloud-brightgreen" alt="hauke.cloud" style="display: block;" /></a>
<a href="https://github.com/hauke-cloud" target="_blank"><img src="https://img.shields.io/badge/github-hauke.cloud-blue" alt="hauke.cloud Github Organisation" style="display: block;" /></a>
<a href="https://github.com/hauke-cloud/llm-readme-management" target="_blank"><img src="https://img.shields.io/badge/template-golang-orange" alt="Repository type - golang" style="display: block;" /></a>


# Database Manager


<img src="https://raw.githubusercontent.com/hauke-cloud/.github/main/resources/img/organisation-logo-small.png" alt="hauke.cloud logo" width="109" height="123" align="right">


<llm header hint="Name the Go module path and say whether this is a service, a CLI or a library.">

This is a Go service (module `github.com/hauke-cloud/database-manager`) that runs as a Kubernetes operator, reconciling `Database` custom resources to manage PostgreSQL and TimescaleDB connections for storing IoT sensor measurements. You would deploy it into a cluster running the hauke.cloud IoT stack.

</llm>


## :book: Description

<llm description>

database-manager is a Kubernetes operator that manages PostgreSQL and TimescaleDB connections for the hauke.cloud IoT stack. It watches `Database` custom resources (group `iot.hauke.cloud`, v1alpha1) and maintains a pooled connection to each referenced database, handling TLS configuration, connection pooling, and automatic table migration.

On reconciliation the operator builds a DSN from the resource spec, applies the requested `sslMode` and any CA or client-certificate Secrets, opens a GORM/pgx pool, and auto-migrates measurement tables for each sensor type listed in `spec.supportedSensorTypes`. It reports `connectionState` and `lastConnectedTime` in the resource status, requeuing every 30 seconds if the connection fails.

- Installs or updates the `databases.iot.hauke.cloud` CRD at startup from an embedded manifest.
- Reconciles `Database` resources: DSN/TLS construction, connection pooling (`maxConnections`/`minConnections`), and status reporting.
- Auto-migrates GORM tables for four sensor types: `moisture`, `water_level`, `valve`, and `room`.
- Provides a `StoreMeasurement` function for in-process consumers to insert Tasmota-style payloads into the correct table.

The operator ships as a Helm chart at `ghcr.io/hauke-cloud/charts/database-manager` and runs alongside other hauke-cloud services such as `mqtt-sensor-exporter`; shared CRD types come from the external `kubernetes-iot-api` module.

</llm>


## :clipboard: Requirements

<llm requirements hint="Give the Go version from the go directive in go.mod. Mention Docker only if the repository actually builds an image.">

- Go 1.25.3 (pinned in `go.mod`).
- Docker, for building and pushing the container image (`make docker-build`, `make docker-push`).
- `kubectl` with a reachable Kubernetes cluster (valid kubeconfig context).
- A PostgreSQL or TimescaleDB instance reachable from the cluster, with a database and user matching the `Database` resource spec.
- A Kubernetes Secret in the target namespace containing the key `password` (optionally `ca.crt`, `tls.crt`, `tls.key`).
- RBAC for the operator ServiceAccount: CRUD on `databases` (including finalizers and status) in group `iot.hauke.cloud`, get/list/watch on `secrets`, and create/get/list/update on `customresourcedefinitions`.
- `pre-commit` and `gitleaks` v8.18.0 for contributors.

</llm>


## 🚀 Getting started

<llm getting_started hint="Cover go build, go run and go test with the real package paths. If a Makefile or Taskfile exists, prefer its targets over raw go commands.">

1. Clone the repository and enter the project directory.

```bash
git clone https://github.com/hauke-cloud/database-manager.git
cd database-manager
```

2. Build the operator binary; this also runs code generation, formatting, and vetting.

```bash
make build
```

3. Run the unit tests.

```bash
make test
```

4. With a reachable Kubernetes cluster and a matching kubeconfig, start the operator.

```bash
make run
```

</llm>


## :airplane: Usage

<llm usage hint="For a library, show a small import-and-call example using real exported identifiers. For a service or CLI, show how it is started and the flags or subcommands it accepts.">

Once the operator is running in your cluster, you interact with it by creating `Database` custom resources. The operator watches those resources, opens a connection to the referenced PostgreSQL or TimescaleDB instance, auto-migrates the measurement tables for each supported sensor type, and reports connection status in the resource's `.status` field.

**Deploy the operator**

The Helm chart is the supported deployment path. Install it into a dedicated namespace:

```bash
helm install database-manager oci://ghcr.io/hauke-cloud/charts/database-manager \
  --version 0.1.0 \
  --namespace database-manager-system \
  --create-namespace
```

The chart enables leader election by default and exposes metrics on port 8080 and health probes on port 8081.

**Create a Database resource**

Define a `Database` CR (group `iot.hauke.cloud`, short names `db`/`dbs`) pointing at your database. The password is read from the Secret key `password` referenced by `passwordSecretRef`:

```yaml
apiVersion: iot.hauke.cloud/v1alpha1
kind: Database
metadata:
  name: sensor-db
  namespace: database-manager-system
spec:
  host: postgres.example.internal
  port: 5432
  database: sensors
  username: sensor_writer
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

On first connect the operator creates the corresponding tables (`moisture_measurements`, `water_level_measurements`, `valve_measurements`, `room_measurements`) and requeues every 30 s if the connection fails.

**Run locally**

For development against your current kubeconfig:

```bash
make run
```

Useful flags include `--log-level debug` and `--leader-elect` (disabled by default).

</llm>


## :wrench: Configuration

<llm configuration hint="Environment variables and CLI flags, taken from the flag definitions or the config struct.">

The operator binary accepts four CLI flags (defined in `cmd/main.go`). No environment variables are consumed by the Go code; the Helm chart sets `LOG_LEVEL` and `LOG_FORMAT` but the binary ignores them.

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--metrics-bind-address` | string | `:8080` | Listen address for the Prometheus metrics endpoint |
| `--health-probe-bind-address` | string | `:8081` | Listen address for liveness and readiness probes |
| `--leader-elect` | bool | `false` | Enable leader election for multi-replica deployments |
| `--log-level` | string | `info` | Log level: `debug`, `info`, `warn`, or `error` |

Each `Database` custom resource (group `iot.hauke.cloud`, v1alpha1) carries its own connection configuration: `host`, `port` (default `5432`), `database`, `username`, `passwordSecretRef`, `sslMode` (default `require`), `supportedSensorTypes`, `maxConnections` (default `10`), and `minConnections` (default `2`). Optional `caSecretRef` and `clientCertSecretRef` fields enable TLS with CA or mutual authentication. The schema also declares `batchSize` and `batchTimeout`, but the operator does not implement batching. The full schema lives in `config/crd/iot.hauke.cloud_databases.yaml`.

Deployment-level settings (replica count, image, resource limits, RBAC, leader election, metrics and health ports) are controlled through Helm values. See `deployments/helm/database-manager/values.yaml` for the complete list.

</llm>


## :hammer: Development

<llm development hint="Include go test, go vet and gofmt only where the CI workflows actually run them.">

Before pushing, install the pre-commit hooks so that `pre-commit-hooks` (v4.4.0) and `gitleaks` (v8.18.0) run on every commit:

```bash
pre-commit install
```

Run them manually against the whole tree with `pre-commit run --all-files`.

**Tests**

```bash
make test
```

This runs `manifests generate fmt vet` and then `go test ./... -coverprofile cover.out`. CI additionally enables the race detector:

```bash
go test -v -race -coverprofile=coverage.out -covermode=atomic ./...
```

**Lint and format**

CI checks formatting with `gofmt -s -l .` (note the `-s` simplification flag) and runs `go vet ./...`. To match CI locally:

```bash
gofmt -s -w .
go vet ./...
```

**Generated files**

`make generate` (controller-gen object/deepcopy) and `make manifests` (RBAC + CRD) must be run after any change to types or kubebuilder markers, and the output committed. CI re-runs both as a pre-build step; stale generated files will cause a mismatch.

```bash
make generate
make manifests
```

**PR title**

A workflow (`.github/workflows/pr-title.yml`) validates the pull-request title. Follow the conventional-commit format described in CONTRIBUTING.md.

</llm>


## 📄 License

This Project is licensed under the GNU General Public License v3.0

- see the [LICENSE](LICENSE) file for details.


## :coffee: Contributing

To become a contributor, please check out the [CONTRIBUTING](CONTRIBUTING.md) file.


## :email: Contact

For any inquiries or support requests, please open an issue in this
repository or contact us at [contact@hauke.cloud](mailto:contact@hauke.cloud).
