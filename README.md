# apps/kubic

One directory per operator. Everything here is plain Kustomize — no Helm, so the
Argo CD repo-server needs no `--enable-helm`.

Each directory contains:

| File                 | Purpose                                                        |
| -------------------- | -------------------------------------------------------------- |
| `kustomization.yaml` | Makes the directory a Kustomize app.                             |
| `config.yaml`        | ApplicationSet metadata: `name`, `syncWave`, `description`.      |
| `*.yaml`             | Namespace, OperatorGroup, Subscription, and the operator's CRs.  |

## Apps

| Directory                          | Operator                                  | Namespace                                  | Wave |
| ---------------------------------- | ----------------------------------------- | ------------------------------------------ | ---- |
| `kernel-module-management`         | Kernel Module Management                  | `openshift-kmm`                             | 0    |
| `connectivity-link-operator`       | Red Hat Connectivity Link                 | `openshift-connectivity-link`               | 0    |
| `cluster-observability-operator`   | Cluster Observability Operator            | `openshift-cluster-observability-operator`  | 0    |
| `opentelemetry-operator`           | Red Hat build of OpenTelemetry            | `openshift-opentelemetry-operator`          | 0    |
| `tempo-operator`                   | Tempo                                     | `openshift-tempo-operator`                  | 0    |
| `custom-metrics-autoscaler`        | Custom Metrics Autoscaler (KEDA)          | `openshift-keda`                            | 0    |
| `leader-worker-set-operator`       | Leader Worker Set                         | `openshift-lws-operator`                    | 0    |
| `jobset-operator`                  | JobSet                                    | `openshift-jobset-operator`                 | 0    |
| `kueue-operator`                   | Kueue (RHBOK)                             | `openshift-kueue-operator`                  | 0    |
| `node-feature-discovery`           | Node Feature Discovery                    | `openshift-nfd`                             | 10   |
| `sriov-network-operator`           | SR-IOV Network Operator                   | `openshift-sriov-network-operator`          | 11   |
| `nvidia-gpu-operator`              | NVIDIA GPU Operator                       | `nvidia-gpu-operator`                       | 12   |
| `rhoai-operator`                   | Red Hat OpenShift AI                      | `redhat-ods-operator`                       | 30   |

Three more apps are parked in `../../optional-apps/` because they are gated off
in the current Helm values: `nvidia-network-operator` (RDMA),
`rhoai-spark-ui` (Spark), and `cluster-monitoring-user-workload`. Each has a
README explaining how to turn it on. Moving the directory into `apps/kubic/`
is what enables it.

## Two levels of sync-wave

- **Resource waves** — the `argocd.argoproj.io/sync-wave` annotations on the
  manifests, carried over unchanged from the Helm templates. These order
  Namespace → OperatorGroup → Subscription → CR *within* an app.
- **Application waves** — `syncWave` in `config.yaml`, stamped onto the
  generated `Application` by the ApplicationSet. These order apps relative to
  each other and depend on the `Application` health check in
  `gitops-config/argocd-instance.yaml`, which keeps a parent waiting for a child
  to report Healthy + Synced.

Cross-app ordering is a soft guarantee either way: every CR keeps
`SkipDryRunOnMissingResource=true` and every app retries, so a CR applied before
its CRD exists simply retries until the operator is up.

## If the discovering ApplicationSet uses a directory generator

`applicationsets/kubic.yaml` uses a git **files** generator so each app can
carry its own `syncWave`. If the platform team's ApplicationSet instead uses a
plain directory generator:

```yaml
generators:
  - git:
      repoURL: https://github.com/redhat-ai-americas/rhoai-argo.git
      revision: main
      directories:
        - path: apps/kubic/*
```

then every app is created in one wave and the per-app `config.yaml` is ignored
(Kustomize skips it — it is not in `resources`). The resource-level waves and
retry behaviour still hold, so the stack converges; it just converges with more
churn while the GPU and RHOAI CRs wait for their CRDs.
