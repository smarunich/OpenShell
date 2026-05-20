# OpenShell on OpenShift Runbook

Use this reference when a user asks to deploy, validate, or debug OpenShell on OpenShift. The user-facing equivalent of this runbook is `docs/kubernetes/openshift-deployment-guide.mdx`; this file is the agent-facing recipe.

## Decision Tree

Start by identifying the target mode:

| User goal | Recommended path |
|---|---|
| Quick smoke test | Plaintext MVP install with `oc port-forward`, dynamic local-storage as default SC if cluster default is broken. |
| Enterprise-like namespace test | One Helm release per team namespace, still plaintext unless mTLS is explicitly requested. Re-apply the PSA label, SCC, and per-release `server.grpcEndpoint`. |
| mTLS test | cert-manager experiment plus manual client bundle extraction. Use `https://...cluster.local.:8080` (trailing dot) for the gateway endpoint. |
| Kata test | Install OpenShift sandboxed containers and create sandboxes through the Python SDK runtime-class workaround. |
| Air-gapped test | Mirror gateway, supervisor, sandbox, Agent Sandbox, cert-manager, local-path-provisioner, busybox, and Kata operator artifacts before install. |
| Shared multi-user gateway | Explain that object ownership is not ready; use gateway-per-team. |

## Current Truths to Preserve

- The OpenShift path is experimental and evaluation-only today.
- The documented current install disables `pkiInitJob` and TLS.
- Sandbox pods require the `privileged` SCC AND the namespace must be labeled `pod-security.kubernetes.io/enforce=privileged` (PSA alone defaults to `restricted` on OpenShift 4.x).
- There is no Helm `openshift.enabled` value and no chart-rendered SCC.
- The Kubernetes driver supports `runtimeClassName` through `SandboxTemplate.runtime_class_name`, but the CLI has no `--runtime-class` flag.
- The Kubernetes driver overwrites `OPENSHELL_SANDBOX_COMMAND` with `sleep infinity` on current main. PR #1326 changes that behavior but is not assumed merged.
- Gateway/certgen image pull secrets are first-class; sandbox image pull secrets are not. Link pull secrets to the namespace default service account as a workaround.
- Both the gateway PVC and per-sandbox PVCs hardcode RWO + no `storageClassName`. They depend on a healthy cluster default StorageClass; ODF/Ceph in HEALTH_WARN silently blocks the entire install.
- The gateway image declares `USER nvs` (non-numeric). With `runAsNonRoot: true` and `runAsUser: null` (the old documented override), kubelet refuses the pod. Use `runAsNonRoot: null` on OpenShift.
- The sandbox supervisor's DNS resolver fails on OpenShift's `ndots:5` resolv.conf when given the auto-derived `openshell.openshell.svc.cluster.local`. Use the trailing-dot FQDN (`...cluster.local.:8080`) or the cluster IP as `server.grpcEndpoint`.
- External Agent Sandbox CR adoption is incomplete and should not be presented as supported GitOps.
- Shared gateway multi-tenancy is not ready because object ownership and provider scoping are missing.

## Pre-Flight Storage Bootstrap (Required if Default SC is Unhealthy)

```bash
LPP_VERSION=v0.0.31

oc apply -f "https://raw.githubusercontent.com/rancher/local-path-provisioner/${LPP_VERSION}/deploy/local-path-storage.yaml"

oc adm policy add-scc-to-user privileged \
  -z local-path-provisioner-service-account -n local-path-storage
oc adm policy add-scc-to-user privileged -z default -n local-path-storage

for NODE in $(oc get nodes -o name | sed 's|node/||'); do
  oc debug -q node/${NODE} \
    --image=registry.redhat.io/ubi9/ubi-minimal:latest -- \
    chroot /host bash -c '
      mkdir -p /opt/local-path-provisioner
      chmod 1777 /opt/local-path-provisioner
      chcon -Rt container_file_t /opt/local-path-provisioner
    '
done

DEFAULT_SC=$(oc get sc -o jsonpath='{range .items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")]}{.metadata.name}{"\n"}{end}')
[ -n "${DEFAULT_SC}" ] && oc annotate sc "${DEFAULT_SC}" storageclass.kubernetes.io/is-default-class=false --overwrite
oc annotate sc local-path storageclass.kubernetes.io/is-default-class=true --overwrite
```

Skip this if the cluster's existing default SC successfully provisions a fresh 1Gi RWO PVC.

## Plaintext MVP Install (Verified)

Use this for the first test.

```bash
export OPENSHELL_NAMESPACE=openshell
export OPENSHELL_CHART_VERSION=0.0.43

oc new-project "${OPENSHELL_NAMESPACE}" || oc project "${OPENSHELL_NAMESPACE}"
oc label ns "${OPENSHELL_NAMESPACE}" \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged \
  --overwrite
oc adm policy add-scc-to-user privileged -z default -n "${OPENSHELL_NAMESPACE}"
```

If recycling a hostPath data dir:

```bash
for NODE in $(oc get nodes -o name | sed 's|node/||'); do
  oc debug -q node/${NODE} \
    --image=registry.redhat.io/ubi9/ubi-minimal:latest -- \
    chroot /host bash -c 'rm -rf /var/openshell-data/* /var/openshell-data/.[!.]* 2>/dev/null || true'
done
```

Create `openshift-values.yaml`:

```yaml
image:
  repository: ghcr.io/nvidia/openshell/gateway
  tag: "0.0.43"

supervisor:
  image:
    repository: ghcr.io/nvidia/openshell/supervisor
    tag: "0.0.43"
  sideloadMethod: init-container

server:
  disableTls: true
  sandboxNamespace: openshell
  sandboxImage: ghcr.io/nvidia/openshell-community/sandboxes/base:latest
  logLevel: info
  grpcEndpoint: "http://openshell.openshell.svc.cluster.local.:8080"

pkiInitJob:
  enabled: false

certManager:
  enabled: false

podSecurityContext:
  fsGroup: null

securityContext:
  runAsNonRoot: null
  runAsUser: null
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL

networkPolicy:
  enabled: true
```

Install:

```bash
helm upgrade --install openshell \
  oci://ghcr.io/nvidia/openshell/helm-chart \
  --version "${OPENSHELL_CHART_VERSION}" \
  --namespace "${OPENSHELL_NAMESPACE}" \
  --values openshift-values.yaml \
  --wait --timeout 5m
```

Connect:

```bash
oc -n "${OPENSHELL_NAMESPACE}" port-forward svc/openshell 18080:8080 &

openshell gateway remove openshift 2>/dev/null || true
openshell gateway add http://127.0.0.1:18080 --name openshift
openshell gateway select openshift
openshell status
```

Smoke test (use non-TTY exec; TTY hangs over port-forward):

```bash
openshell sandbox create --name ocp-smoke --from ghcr.io/nvidia/openshell-community/sandboxes/base:latest &
sleep 60   # wait for image pull + Ready

echo "" | openshell sandbox exec -n ocp-smoke -- /bin/sh -c 'id; cat /etc/os-release | head -3'
echo "" | openshell sandbox exec -n ocp-smoke -- /bin/sh -c 'ls -la /sandbox | head -10'
openshell logs ocp-smoke --source sandbox | tail -10
```

Expected sandbox log highlights:

```
CONFIG:APPLYING  Applying Landlock filesystem sandbox [abi:V2 compat:BestEffort ro:7 rw:3]
CONFIG:BUILT     Landlock ruleset built [rules_applied:9 skipped:1]
SSH:OPEN         ALLOWED
NET:OPEN         [MED] DENIED /usr/bin/curl(...) -> github.com:443 [policy:- engine:opa]
```

## Kata Workaround

Only test after the base install works.

```bash
oc get runtimeclass
```

Use the Python SDK because the CLI has no runtime-class flag:

```bash
python - <<'PY'
from openshell.sandbox import SandboxClient
from openshell._proto import openshell_pb2

spec = openshell_pb2.SandboxSpec(
    template=openshell_pb2.SandboxTemplate(
        image="ghcr.io/nvidia/openshell-community/sandboxes/base:latest",
        runtime_class_name="kata",
    )
)

client = SandboxClient.from_active_cluster(cluster="openshift", timeout=60)
try:
    sandbox = client.create(spec=spec)
    print(f"created {sandbox.name} id={sandbox.id}")
    client.wait_ready(sandbox.name, timeout_seconds=300)
finally:
    client.close()
PY
```

Verify:

```bash
oc -n "${OPENSHELL_NAMESPACE}" get pod -l openshell.ai/managed-by=openshell \
  -o jsonpath='{range .items[*]}{.metadata.name}{" runtimeClass="}{.spec.runtimeClassName}{"\n"}{end}'
```

## Air-Gapped Checklist

Mirror at least:

- `ghcr.io/nvidia/openshell/gateway:<version>`
- `ghcr.io/nvidia/openshell/supervisor:<version>`
- `ghcr.io/nvidia/openshell-community/sandboxes/base:latest`
- Agent Sandbox controller image(s)
- cert-manager images (if mTLS is tested)
- OpenShift sandboxed containers operator/catalog artifacts (if Kata is tested)
- `rancher/local-path-provisioner` image and the `busybox:latest` it pulls (if the local-storage workaround is needed)

Use `skopeo copy --all` for multi-arch safety.

In values:

```yaml
image:
  repository: registry.example.com/openshell/gateway
  tag: "0.0.43"

supervisor:
  image:
    repository: registry.example.com/openshell/supervisor
    tag: "0.0.43"
  sideloadMethod: init-container

server:
  sandboxImage: registry.example.com/openshell/sandboxes/base:latest
  grpcEndpoint: "http://openshell.openshell.svc.cluster.local.:8080"

imagePullSecrets:
  - name: internal-registry
```

If registry auth is needed, link the pull secret to both service accounts:

```bash
oc -n "${OPENSHELL_NAMESPACE}" secrets link openshell internal-registry --for=pull || true
oc -n "${OPENSHELL_NAMESPACE}" secrets link default   internal-registry --for=pull
```

## cert-manager Experiment

Do not present this as the main supported path.

Values (incremental over the verified base):

```yaml
server:
  disableTls: false
  sandboxNamespace: openshell
  grpcEndpoint: "https://openshell.openshell.svc.cluster.local.:8080"

pkiInitJob:
  enabled: false

certManager:
  enabled: true
  serverDnsNames:
    - openshell
    - openshell.openshell.svc
    - openshell.openshell.svc.cluster.local
    - localhost
  serverIpAddresses:
    - 127.0.0.1
```

Extract CLI bundle manually:

```bash
GATEWAY_NAME=openshift-mtls
MTLS_DIR="${HOME}/.config/openshell/gateways/${GATEWAY_NAME}/mtls"
mkdir -p "${MTLS_DIR}"

oc -n "${OPENSHELL_NAMESPACE}" get secret openshell-client-tls -o jsonpath='{.data.ca\.crt}'  | base64 -d > "${MTLS_DIR}/ca.crt"
oc -n "${OPENSHELL_NAMESPACE}" get secret openshell-client-tls -o jsonpath='{.data.tls\.crt}' | base64 -d > "${MTLS_DIR}/tls.crt"
oc -n "${OPENSHELL_NAMESPACE}" get secret openshell-client-tls -o jsonpath='{.data.tls\.key}' | base64 -d > "${MTLS_DIR}/tls.key"
```

## Debug Commands

```bash
helm -n "${OPENSHELL_NAMESPACE}" status openshell
helm -n "${OPENSHELL_NAMESPACE}" get values openshell
oc -n "${OPENSHELL_NAMESPACE}" get statefulset,pod,svc,pvc,sandbox
oc -n "${OPENSHELL_NAMESPACE}" logs statefulset/openshell --tail=200
oc -n "${OPENSHELL_NAMESPACE}" describe pod -l openshell.ai/managed-by=openshell
oc -n "${OPENSHELL_NAMESPACE}" get events --sort-by=.lastTimestamp | tail -50
oc -n "${OPENSHELL_NAMESPACE}" logs <sandbox-pod> --all-containers --tail=100
```

Targeted probes:

```bash
# Verify the supervisor can reach the gateway (run inside a temp pod on the same node as the sandbox)
oc run net-test -n "${OPENSHELL_NAMESPACE}" \
  --image=registry.access.redhat.com/ubi9/ubi-minimal:latest --restart=Never \
  --command -- bash -c '
    getent hosts openshell.openshell.svc.cluster.local
    curl -sS --http2-prior-knowledge -o /dev/null -w "HTTP %{http_code} time=%{time_total}s\n" \
      --max-time 5 http://openshell.openshell.svc.cluster.local.:8080/
  '
oc logs net-test -n "${OPENSHELL_NAMESPACE}"
oc delete pod net-test -n "${OPENSHELL_NAMESPACE}" --force --grace-period=0
```

If curl works but the supervisor fails with `failed to connect to OpenShell server`, the supervisor DNS resolver bug is hitting you — fix by setting `server.grpcEndpoint` with the trailing-dot FQDN.

## Failure Table (Empirically Confirmed)

| Symptom | Root cause | Action |
|---|---|---|
| `helm upgrade` times out, PVC `Pending`, event `failed to provision volume ... DeadlineExceeded` | Cluster default StorageClass backend (commonly ODF Ceph) unhealthy. | Apply the Pre-Flight Storage Bootstrap section. |
| Pod `CreateContainerConfigError`, event `container has runAsNonRoot and image has non-numeric user (nvs)` | `securityContext.runAsNonRoot: true` set against an image with `USER nvs`. | Set `runAsNonRoot: null` in values. |
| Pod admission rejected: `violates PodSecurity ... capabilities.add` | Namespace PSA `restricted`. | Label namespace `pod-security.kubernetes.io/enforce=privileged`. |
| Gateway logs `migration N was previously applied but is missing in the resolved migrations` | Stale SQLite DB in reused hostPath/local PV. | Wipe the data dir. |
| Gateway healthy but every sandbox crash-loops with `Policy fetch failed after 5 attempts: failed to connect to OpenShell server` | Supervisor DNS resolver crashes on OpenShift resolv.conf with auto-derived FQDN. | Set `server.grpcEndpoint: "http://openshell.openshell.svc.cluster.local.:8080"` and `helm upgrade`, then `oc rollout restart sts/openshell`. |
| Sandbox CR flaps between `Ready` and `Provisioning` repeatedly | Same as above; supervisor session never stays up. | Same as above. |
| `openshell sandbox exec` hangs forever with TTY allocation | TTY allocation over SSH-over-gRPC stalls through `oc port-forward`. | Pipe stdin (`echo ""` or `< /dev/null`). |
| `helm uninstall` leaves PVCs Pending forever | Stuck Ceph CSI ops. | `oc patch pvc <name> -p '{"metadata":{"finalizers":null}}' --type=merge` and use the local-path SC. |
| CLI warns `mTLS certificates found for gateway 'X'` even after switching to plaintext | Stale `~/.config/openshell/gateways/<name>/mtls/`. | Remove that subdirectory before `openshell gateway add`. |
| `openshell --version` reports an older version than expected | Stale uv-installed binary in `~/.local/bin` shadowing Homebrew/RPM. | Remove the uv install or call `/opt/homebrew/bin/openshell` (macOS) explicitly. |

## Gap References

Use these tracking references in explanations:

- #899: restricted SCC/platform mode.
- #1012: HA Kubernetes roadmap.
- #1015 and #1030: private sandbox image pull secrets.
- #1020: Agent Sandbox subchart.
- #1024: cert-manager mTLS.
- #1026 and #1336: Kubernetes E2E/canary testing.
- #1018 and #1250: RBAC docs.
- #848 and #1326: daemon command/custom startup.
- #1145: multi-tenant roadmap.
- #1354 and #1404: per-sandbox supervisor auth.
- #1363: Postgres DB secret.
- #1414: SPIFFE.
- #1436: workspace PVC size.

New gaps from the May 20 2026 live walkthrough (file upstream issues as needed):

- Supervisor DNS resolver crashes on OpenShift `ndots:5` + four search domains; `server.grpcEndpoint` trailing-dot workaround.
- Gateway image declares `USER nvs` (non-numeric); chart/docs incorrectly recommend `runAsNonRoot: true` + `runAsUser: null`.
- Namespace `pod-security.kubernetes.io/enforce=privileged` label required in addition to the SCC binding.
- Gateway + sandbox PVCs lack `storageClassName` / size overrides.
