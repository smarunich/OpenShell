---
name: deploy-openshell-openshift
description: Deploy, validate, and debug OpenShell on Red Hat OpenShift using the verified Helm chart path. Use when users ask for OpenShell on OpenShift, OpenShift SCCs, Pod Security Admission labels, Kata Containers runtimeClassName, cert-manager mTLS on OpenShift, air-gapped OpenShell installs, gateway-per-team OpenShift deployments, broken default StorageClass workarounds, supervisor DNS resolver failures, or OpenShift readiness gap analysis.
---

# Deploy OpenShell on OpenShift

## Overview

Use this skill to guide an OpenShift deployment without overstating support. The current reliable path is an evaluation install: one gateway release in one namespace, Agent Sandbox installed separately, sandbox pods admitted through the `privileged` SCC with namespace PSA `enforce: privileged`, TLS disabled, dynamic local-storage as the default StorageClass when the cluster's default is broken, an explicit trailing-dot gateway FQDN, and CLI access through `oc port-forward`.

The bulletproof, step-by-step user-facing guide is at `docs/kubernetes/openshift-deployment-guide.mdx`. Mirror its order of operations; do not invent your own.

## First Checks

Establish these facts before giving commands:

- OpenShift version, Kubernetes version, RHCOS kernel, CRI-O version, and whether SELinux is enforcing.
- Whether the user wants plaintext evaluation, cert-manager mTLS, Kata, air-gapped mirroring, or multi-team operation.
- Whether `oc`, `helm`, `jq`, `skopeo`, and `openshell` are available, and which `openshell` binary is on `PATH` (uv installs at `~/.local/bin/openshell` shadow Homebrew/RPM installs).
- Whether Agent Sandbox CRDs and controller are already installed.
- Whether the **cluster default StorageClass is healthy** (this is the single most common silent failure: ODF Ceph clusters in `HEALTH_WARN` block every install).
- Whether the cluster can pull GHCR images or must use an internal registry.
- Whether cert-manager and OpenShift sandboxed containers are already installed if those modes are needed.

## Workflow (Always In This Order)

1. Pin the CLI to the same release as the chart. Published Helm chart can lead the published CLI; mixing versions is unsupported.
2. Install or verify Agent Sandbox before OpenShell.
3. Verify the cluster default StorageClass actually provisions a 1Gi RWO PVC end-to-end. If not, deploy `rancher/local-path-provisioner` with the SCC + SELinux fixes from the runbook and make it the new default.
4. Create the OpenShell namespace. Label it `pod-security.kubernetes.io/enforce=privileged`. Bind the `privileged` SCC to the namespace `default` service account.
5. Wipe any stale data directory if the install is reusing a hostPath/local PV.
6. Install the Helm chart with the verified values:
   - `pkiInitJob.enabled=false`, `certManager.enabled=false`, `server.disableTls=true`.
   - `podSecurityContext.fsGroup: null`.
   - `securityContext.runAsNonRoot: null`, `runAsUser: null`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`.
   - `supervisor.sideloadMethod: init-container`.
   - **Required:** `server.grpcEndpoint: "http://openshell.openshell.svc.cluster.local.:8080"` (trailing dot bypasses the OpenShift `ndots:5` DNS resolver bug in the supervisor binary).
7. `oc port-forward svc/openshell 18080:8080`, then `openshell gateway add http://127.0.0.1:18080 --name openshift-<cluster>` and `openshell status`.
8. Smoke-test with `echo "" | openshell sandbox exec -n <name> -- /bin/sh -c '...'` (non-TTY). Confirm OCSF JSONL logs via `openshell logs <name> --source sandbox`.
9. Add Kata only after the base path works. Use the Python SDK `SandboxTemplate.runtime_class_name` because the CLI has no `--runtime-class` flag.
10. For air-gapped clusters, mirror gateway, supervisor, sandbox, Agent Sandbox, cert-manager, local-path-provisioner, busybox helper, and Kata operator images with `skopeo copy --all`.
11. For multi-team setups, use one Helm release per namespace. Re-apply the PSA label, SCC binding, and per-namespace `server.grpcEndpoint` override per release. Do not recommend a shared gateway for untrusted users.

## Non-Negotiable Current-State Notes

- Do not promise `openshift.enabled=true`; that value does not exist.
- Do not promise chart-rendered SCCs; manual SCC binding is required today.
- Do not recommend `securityContext.runAsNonRoot: true` for the gateway pod; the image declares `USER nvs` (non-numeric) and kubelet will refuse the pod.
- Do not skip the namespace `pod-security.kubernetes.io/enforce=privileged` label. SCC binding alone is not enough.
- Do not assume the cluster default StorageClass works; check it explicitly before installing.
- Do not skip the `server.grpcEndpoint` trailing-dot override on OpenShift. Without it the install completes, the gateway runs, and every sandbox crash-loops with `Policy fetch failed after 5 attempts: failed to connect to OpenShell server`.
- Do not promise `openshell sandbox create --runtime-class kata`; use the SDK workaround.
- Do not promise declarative daemon startup; current main overwrites `OPENSHELL_SANDBOX_COMMAND` with `sleep infinity`.
- Do not claim cert-manager is the supported OpenShift path. It is chart-supported but not the current OpenShift quick path.
- Do not claim shared-gateway multi-tenancy is ready. Current authz is method/role/scope based, not object-owner based.
- Do not claim external Agent Sandbox CRs are a complete GitOps interface. Gateway adoption is partial and loses spec/policy/provider intent.

## Common Debug Path

Run these in order:

```bash
openshell gateway list
openshell status
helm -n "${OPENSHELL_NAMESPACE:-openshell}" status openshell
oc -n "${OPENSHELL_NAMESPACE:-openshell}" get statefulset,pod,svc,pvc,sandbox
oc -n "${OPENSHELL_NAMESPACE:-openshell}" logs statefulset/openshell --tail=200
oc -n "${OPENSHELL_NAMESPACE:-openshell}" describe pod -l openshell.ai/managed-by=openshell
oc -n "${OPENSHELL_NAMESPACE:-openshell}" get events --sort-by=.lastTimestamp | tail -50
oc -n "${OPENSHELL_NAMESPACE:-openshell}" logs <sandbox-pod> --tail=50
oc get sc
oc -n "${OPENSHELL_NAMESPACE:-openshell}" get pvc -o wide
```

Interpret the results against the runbook's troubleshooting decision tree. The most frequent root causes in order are:

1. `Policy fetch failed after 5 attempts` → missing `server.grpcEndpoint` trailing-dot override.
2. `container has runAsNonRoot and image has non-numeric user (nvs)` → wrong `securityContext.runAsNonRoot` value.
3. PSA `violates PodSecurity ... capabilities.add` → missing namespace `pod-security.kubernetes.io/enforce=privileged` label.
4. PVC `Pending ... failed to provision volume ... DeadlineExceeded` → broken default StorageClass.
5. `migration N was previously applied but is missing in the resolved migrations` → stale SQLite DB in reused hostPath/local PV.

## References

- Detailed commands: [openshift-runbook.md](references/openshift-runbook.md).
- User-facing guide: `docs/kubernetes/openshift-deployment-guide.mdx`.
- Requirement gaps: `docs/kubernetes/openshift-prd-gap-appendix.mdx`.
- For local lab evidence, store install transcripts and verified values under the operator's gitignored `architecture/plans/<lab-name>/` directory rather than in tracked docs.
