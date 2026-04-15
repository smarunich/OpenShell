# Running AI Agents with Kata Containers

Run OpenShell sandboxes inside Kata Container VMs for hardware-level
isolation on Kubernetes.  This example covers Kata setup, custom sandbox
images, policy authoring, and sandbox creation with `runtimeClassName`.

## Architecture

```mermaid
graph TB
  subgraph userMachine [Developer Machine]
    CLI["openshell CLI"]
    PySDK["Python SDK"]
    DockerBuild["Docker"]
  end

  subgraph k8sCluster [Kubernetes Cluster]
    subgraph controlPlane [Control Plane Components]
      GW["OpenShell Gateway<br/>StatefulSet - port 8080"]
      DB["SQLite / PostgreSQL"]
      CRD["Sandbox CRD<br/>agents.x-k8s.io/v1"]
      Controller["agent-sandbox-controller<br/>Reconciles CR to Pod"]
      RC["RuntimeClass<br/>kata-containers"]
      KataDeploy["kata-deploy DaemonSet<br/>Installs Kata on nodes"]
    end

    subgraph workerNode [Worker Node]
      Containerd["containerd"]
      KataRT["Kata runtime<br/>QEMU / Cloud Hypervisor"]
      SupervisorBin["/opt/openshell/bin/<br/>openshell-sandbox"]
      InstallerDS["supervisor-installer<br/>DaemonSet"]

      subgraph kataVM [Kata MicroVM - dedicated guest kernel]
        subgraph sandboxPod [Sandbox Pod]
          Supervisor["openshell-sandbox<br/>supervisor process"]

          subgraph isolationLayers [OpenShell Isolation Layers]
            Landlock["Landlock LSM<br/>filesystem rules"]
            SeccompBPF["seccomp-BPF<br/>syscall filter"]
            NetNS["Network Namespace<br/>veth 10.200.0.0/24"]
          end

          subgraph netEnforce [Network Enforcement]
            Proxy["HTTP CONNECT Proxy<br/>:3128"]
            OPAEngine["OPA / Rego Engine<br/>per-binary policy"]
            L7["L7 Inspector<br/>REST / TLS terminate"]
            InfRouter["Inference Router<br/>inference.local"]
          end

          AgentProc["AI Agent<br/>Claude Code / OpenClaw /<br/>Hermes / Codex"]
          SSHServer["Embedded SSH Server<br/>:2222"]
        end
      end
    end
  end

  subgraph external [External Services]
    APIs["Allowed APIs<br/>Anthropic / GitHub / npm"]
    BlockedEP["Blocked endpoints<br/>policy denied"]
    LLM["LLM Backends<br/>Ollama / vLLM"]
  end

  CLI -->|"gRPC + mTLS"| GW
  PySDK -->|"gRPC + mTLS<br/>runtime_class_name"| GW
  CLI -->|"SSH tunnel"| SSHServer
  DockerBuild -->|"build + push image"| Containerd

  GW --> DB
  GW -->|"creates Sandbox CR"| CRD
  CRD --> Controller
  Controller -->|"creates Pod with<br/>runtimeClassName"| RC
  RC --> KataRT
  KataRT -->|"boots VM"| kataVM

  InstallerDS -->|"copies binary"| SupervisorBin
  SupervisorBin -->|"hostPath + virtiofs"| Supervisor

  Supervisor --> Landlock
  Supervisor --> SeccompBPF
  Supervisor --> NetNS
  Supervisor --> Proxy
  Supervisor --> SSHServer
  Supervisor -->|"spawns"| AgentProc

  AgentProc -->|"all egress via netns"| Proxy
  Proxy --> OPAEngine
  OPAEngine -->|"allow"| L7
  OPAEngine -->|"deny"| BlockedEP
  L7 --> APIs
  Proxy --> InfRouter
  InfRouter --> LLM

  GW -->|"provider credentials<br/>via placeholder"| Supervisor
  GW -->|"policy YAML<br/>hot-reload"| Supervisor
```

Three layers of isolation protect your infrastructure:

1. **Kata VM** -- each sandbox pod runs inside its own QEMU/Cloud Hypervisor
   microVM with a dedicated guest kernel, providing hardware-level isolation.
2. **OpenShell sandbox** -- inside the VM, the supervisor enforces Landlock
   filesystem rules, seccomp-BPF syscall filters, and a dedicated network
   namespace.
3. **Egress proxy and OPA** -- all outbound traffic passes through an HTTP
   CONNECT proxy that evaluates per-binary, per-endpoint network policy via
   an embedded OPA/Rego engine.

## Prerequisites

- A Kubernetes cluster (v1.26+) with admin access
- Nodes with hardware virtualization (Intel VT-x / AMD-V)
- `kubectl`, `helm` v3, and Docker on your local machine
- OpenShell CLI installed (`curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh`)

## What's in this example

| File                          | Description                                                        |
| ----------------------------- | ------------------------------------------------------------------ |
| `Dockerfile.claude-code`      | Sandbox image for Claude Code (Node.js base)                       |
| `Dockerfile.python-agent`     | Sandbox image for Python-based agents (Hermes, custom)             |
| `policy-claude-code.yaml`     | Network policy: Anthropic API, GitHub, npm, PyPI                   |
| `policy-minimal.yaml`         | Minimal policy: single API endpoint                                |
| `policy-l7-github.yaml`       | L7 read-only policy for the GitHub REST API                        |
| `supervisor-daemonset.yaml`   | DaemonSet that installs the supervisor binary on every node        |
| `create-kata-sandbox.py`      | Python SDK script to create a sandbox with Kata runtime class      |
| `kata-runtimeclass.yaml`      | RuntimeClass manifest for Kata                                     |

## Quick start

### 1. Install Kata Containers on your cluster

```bash
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-rbac/base/kata-rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kata-containers/kata-containers/main/tools/packaging/kata-deploy/kata-deploy/base/kata-deploy.yaml
kubectl -n kube-system wait --for=condition=Ready pod -l name=kata-deploy --timeout=600s
```

Verify the RuntimeClass exists:

```bash
kubectl get runtimeclass
```

If `kata-containers` is missing:

```bash
kubectl apply -f examples/kata-containers/kata-runtimeclass.yaml
```

### 2. Deploy the supervisor binary to nodes

```bash
kubectl apply -f examples/kata-containers/supervisor-daemonset.yaml
```

### 3. Deploy the OpenShell gateway

Follow the [Kata Containers tutorial](../../docs/tutorials/kata-containers.mdx)
for full Helm-based gateway deployment, or use the built-in local gateway:

```bash
openshell gateway start
```

### 4. Create a provider

```bash
export ANTHROPIC_API_KEY=sk-ant-...
openshell provider create --name claude --type claude --from-existing
```

### 5. Create a Kata-isolated sandbox

Using the Python SDK (the CLI does not yet expose `--runtime-class`):

```bash
uv run examples/kata-containers/create-kata-sandbox.py \
  --name my-claude \
  --image myregistry.com/claude-sandbox:latest \
  --provider claude \
  --runtime-class kata-containers
```

Or create via CLI without Kata and patch afterward:

```bash
openshell sandbox create --name my-claude \
  --from examples/kata-containers/Dockerfile.claude-code \
  --provider claude \
  --policy examples/kata-containers/policy-claude-code.yaml

kubectl -n openshell patch sandbox my-claude --type=merge -p '{
  "spec": {
    "podTemplate": {
      "spec": {
        "runtimeClassName": "kata-containers"
      }
    }
  }
}'
```

### 6. Connect and run your agent

```bash
openshell sandbox connect my-claude
# Inside the sandbox:
claude
```

### 7. Verify Kata isolation

```bash
# Guest kernel (should differ from host)
openshell sandbox exec my-claude -- uname -r

# Sandbox network namespace
openshell sandbox exec my-claude -- ip netns list

# Proxy is running
openshell sandbox exec my-claude -- ss -tlnp | grep 3128
```

## Cleanup

```bash
openshell sandbox delete my-claude
openshell provider delete claude
kubectl delete -f examples/kata-containers/supervisor-daemonset.yaml
```
