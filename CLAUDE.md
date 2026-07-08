# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## EgressIP Notes

When working with EgressIP code, be aware of the distinction between the zone controller pattern and the cluster manager pattern for node readiness checks. Changes that work in one context may fail CI in the other.

## Project Overview

OVN-Kubernetes is a Kubernetes CNI (Container Network Interface) plugin that provides robust networking using OVN (Open Virtual Networking) and Open vSwitch. The project implements Kubernetes networking by translating Kubernetes network objects into OVN logical network entities.

## Build and Development Commands

### Building

All build commands should be run from the `go-controller/` directory:

```bash
cd go-controller/

# Build all binaries
make build

# Build with debugging symbols (for delve)
make all GCFLAGS=all="-N -l"

# Build Windows hybrid overlay
make windows

# Clean build artifacts
make clean
```

### Testing

```bash
cd go-controller/

# Run all unit tests
make test

# Run tests with specific focus using Ginkgo
GINKGO_FOCUS="test pattern" make test

# Run tests for specific packages
PKGS="./pkg/allocator/..." make test

# Run tests in container (useful for CI)
# Tests run automatically in docker/podman if container runtime is available
```

### Linting and Formatting

```bash
cd go-controller/

# Run linter (golangci-lint)
make lint

# Auto-fix lint issues
make lint-fix

# Check go formatting
make gofmt

# Verify go.mod and vendor are in sync
make verify-go-mod-vendor
```

### Code Generation

```bash
cd go-controller/

# Generate OVN database bindings from schemas
# Downloads latest ovn-nb.ovsschema and ovn-sb.ovsschema
make modelgen

# Generate CRD clientsets, informers, listers, and YAML manifests
# Output goes to _output/crd/ and dist/templates/
make codegen

# Generate mocks for testing
make mocksgen
```

### Local Development with KIND

The primary development workflow uses KIND (Kubernetes in Docker):

```bash
# From repo root, not go-controller/
cd /path/to/ovn-kubernetes

# Create a basic KIND cluster with ovn-kubernetes
./contrib/kind.sh

# Common development options:
./contrib/kind.sh --num-workers 2              # Set worker node count
./contrib/kind.sh --ha-enabled                 # Enable HA mode
./contrib/kind.sh --ipv6                       # Enable IPv6
./contrib/kind.sh --gateway-mode local         # Use local gateway mode (default: shared)
./contrib/kind.sh --multicast-enabled          # Enable multicast

# Delete the cluster
./contrib/kind.sh delete
```

## Architecture

### Deployment Modes

OVN-Kubernetes supports two deployment modes:

1. **Interconnect Mode (DEFAULT)** - Distributed control plane where each node runs its own OVN databases (NBDB/SBDB)
   - Each node is a "zone" with local databases
   - Better stability (no RAFT), horizontal scalability, improved performance
   - Control plane: `ovnkube-control-plane` pod (cluster-manager only)
   - Data plane: `ovnkube-node` pod (controller + databases + northd + ovn-controller)

2. **Central Mode (DEPRECATED)** - Centralized databases running on control plane nodes
   - Will be removed in future releases
   - Not recommended for new deployments

### Key Components

**Control Plane (Interconnect Mode):**
- `ovnkube-cluster-manager`: Allocates subnets to nodes, consolidates zone statuses

**Data Plane (per node):**
- `ovnkube-controller`: Watches K8s API, translates objects to OVN entities, handles IPAM, runs CNI
- `nbdb`: OVN Northbound database (local to node)
- `northd`: Converts NBDB logical elements to SBDB logical flows
- `sbdb`: OVN Southbound database (local to node)
- `ovn-controller`: Converts SBDB logical flows to OpenFlow rules
- `ovs-daemons`: Open vSwitch daemon and database

### Code Structure

The `go-controller/pkg/` directory contains the main implementation:

- `allocator/`: IP address and resource allocation
- `cni/`: CNI plugin implementation
- `clustermanager/`: Cluster-wide network management, subnet allocation
- `controller/`: Kubernetes resource watchers and reconcilers
- `ovn/`: OVN-specific logic, translates K8s objects to OVN entities
- `node/`: Node-level networking setup
- `kube/`: Kubernetes API client interactions
- `libovsdb/`: Low-level OVS/OVN database client
- `nbdb/`, `sbdb/`: Auto-generated OVN database bindings
- `networkmanager/`: NAD (Network Attachment Definition) controller, network configuration

### Controller Architecture

OVN-Kubernetes uses **level-driven controllers** built with `pkg/controller/controller.go`:

- Controllers are **User Defined Network (UDN) aware**: a single controller instance reconciles objects across all networks
- Network Manager (`pkg/networkmanager/`) is the source of truth for NADs (Network Attachment Definitions)
- Controllers register reconcilers via `RegisterNADReconciler()` to receive NAD events
- Use APIs like `GetActiveNetworkForNamespace()`, `GetPrimaryNADForNamespace()`, `GetNetInfoForNADKey()` to query network info
- Register NAD reconcilers **before** starting controller workers to avoid missing startup events
- See `pkg/ovn/controller/egressfirewall/egressfirewall.go` for a reference implementation

## Commit Message Convention

Follow these guidelines for commit messages:

1. **First line**: Lowercase, 50 chars or less, prefixed with subcomponent
   - Format: `subcomponent: short description`
   - Examples:
     - `networkpolicy: validate ipBlock strictly`
     - `egressip: fix frequently rebalancing IPs`
     - `services: fix etp=local + session-affinity integration`

2. **Body**: Wrap at 72 columns, explain why (not what)

3. **References**: Use `Fixes: #<issue>` or `Refs: #<issue1>, #<issue2>`

4. **Sign-off**: Required (DCO)
   ```
   Signed-off-by: Your Name <your.name@example.com>
   ```

5. **Logical commits**: Squash checkpoint commits, use interactive rebase to edit commits when addressing review comments (don't add "Fix review comments" commits)

## Important Development Patterns

### Schema Version Management

- OVN schema version is set in `go-controller/Makefile` as `OVN_SCHEMA_VERSION`
- When Fedora OVN version is bumped in KIND, update `OVN_SCHEMA_VERSION` and regenerate bindings with `make modelgen`

### Container Runtime

- Build/test scripts auto-detect podman or docker
- Override with `CONTAINER_RUNTIME=podman` or `CONTAINER_RUNTIME=docker`

### Root Privileges

- Some tests require root/sudo (see `root_pkgs` in test scripts)
- Tests automatically compile and run with sudo when needed
- Use `NOROOT=TRUE` to skip privileged operations in constrained environments

### Working Directory

- Most Go commands run from `go-controller/` directory
- KIND scripts run from repository root

## End-to-End Testing

E2E tests are located in `test/` directory. To run specific tests:

```bash
cd test/
go test -timeout=0 -v . -ginkgo.v -ginkgo.focus '<test pattern>' -provider skeleton -kubeconfig <path>
```

## Documentation and Resources

- Architecture: `docs/design/architecture.md`
- Developer guide: `docs/developer-guide/developer.md`
- Contributing: `CONTRIBUTING.md`
- Book: https://ovn-kubernetes.io/
- Slack: #ovn-kubernetes on cloud-native.slack.com

## Agent skills

### Issue tracker

Issues are tracked in both GitHub Issues (ovn-kubernetes/ovn-kubernetes) and Jira (OCPBUGS for bugs, CORENET for features). External PRs are not a triage surface. See `docs/agents/issue-tracker.md`.

### Triage labels

Default label vocabulary (needs-triage, needs-info, ready-for-agent, ready-for-human, wontfix). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context layout (one CONTEXT.md + docs/adr/ at repo root). See `docs/agents/domain.md`.
