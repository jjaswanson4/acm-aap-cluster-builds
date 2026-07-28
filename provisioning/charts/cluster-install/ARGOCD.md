# ArgoCD Deployment Guide

This chart supports deployment via ArgoCD. Set `deployMethod: "argocd"` in your values to enable ArgoCD-specific hook annotations.

## The `deployMethod` Value

By default, `deployMethod` is `"helm"`. When set to `"argocd"`:

- The reset-bmcs pre-install hook uses `argocd.argoproj.io/hook: PreSync` instead of `helm.sh/hook: pre-install`
- Hook delete policies use `argocd.argoproj.io/hook-delete-policy` instead of `helm.sh/hook-delete-policy`

This is necessary because ArgoCD strips `helm.sh/hook`-annotated resources during template rendering, making Helm hooks invisible to ArgoCD sync.

## Sync Waves

Resources are created in order using ArgoCD sync waves:

| Wave | Resources | Purpose |
|------|-----------|---------|
| PreSync (-10) | Namespace (reset-bmcs hook) | Created before sync when `resetBMCs: "true"` |
| PreSync (-5) | ConfigMap, SA, RBAC (reset-bmcs hook) | Hook supporting resources |
| PreSync (0) | Job (reset-bmcs hook) | BMC reset, media eject, Ironic cleanup |
| -5 | Namespace | Created first during sync |
| -4 | Pull Secret | Required by other resources |
| -3 | BMC Secrets | Required by BareMetalHosts |
| -2 | Cluster Manifests ConfigMap | Required by AgentClusterInstall |
| -1 | ClusterCurator, node-configs ConfigMap | Pre-requisites for provisioning |
| 0 | BareMetalHosts, ClusterDeployment, AgentClusterInstall, InfraEnv, NMStateConfigs, ManagedCluster, etc. | Core cluster resources |
| 1 | configure-bmcs Job + RBAC | Deploy TLS certs to BMCs, unpause BMHs |
| 2 | patch-bmh-image Job + RBAC | Set discovery ISO URL on BMHs |
| 3 | auto-approve-hosts Job + RBAC | Wait for agents, approve them |
| 3 | update-agents Job + RBAC (pull mode) | Patch agents with hostnames/roles |
| 4 | configure-bmc-boot Job + RBAC | Eject media, set boot to HDD+UEFI |
| 5 | update-manifests Job + RBAC (pull mode) | Create import manifests, release holdInstallation |

Between waves 2 and 3, Ironic/BMO automatically powers on nodes and mounts the discovery ISO. Agents boot and register with the hub.

## Resource Policies

Resources respect both Helm and ArgoCD resource policies:

**Namespace** (when `namespaceResourcePolicy: keep`):
```yaml
annotations:
  "helm.sh/resource-policy": keep
  "argocd.argoproj.io/sync-options": "Prune=false"
```

**BareMetalHost** (when `baremetalHostResourcePolicy: keep`):
```yaml
annotations:
  "helm.sh/resource-policy": keep
  "argocd.argoproj.io/sync-options": "Prune=false"
```

## Deployment Options

### Option 1: Git Repository Source

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-cluster
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-repo.git
    targetRevision: main
    path: provisioning/charts/cluster
    helm:
      values: |
        clusterName: my-cluster
        namespace: my-cluster
        baseDomain: example.com
        deployMethod: "argocd"
        # ... rest of values
  destination:
    server: https://kubernetes.default.svc
    namespace: my-cluster
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

### Option 2: Helm Repository Source

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-cluster
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.example.com
    chart: cluster
    targetRevision: 0.1.0
    helm:
      values: |
        deployMethod: "argocd"
  destination:
    server: https://kubernetes.default.svc
    namespace: my-cluster
```

## Recommended Sync Policy

```yaml
syncPolicy:
  automated:
    prune: false        # Don't auto-delete (recommended for clusters)
    selfHeal: true      # Auto-sync on drift
  syncOptions:
    - CreateNamespace=false  # Chart creates namespace
    - PruneLast=true         # Respect dependencies
    - RespectIgnoreDifferences=true
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

## Ignore Differences

Add these to prevent ArgoCD from detecting false drift on operator-managed status fields:

```yaml
ignoreDifferences:
  - group: agent-install.openshift.io
    kind: InfraEnv
    jsonPointers:
      - /status
  - group: agent-install.openshift.io
    kind: AgentClusterInstall
    jsonPointers:
      - /status
  - group: hive.openshift.io
    kind: ClusterDeployment
    jsonPointers:
      - /status
  - group: metal3.io
    kind: BareMetalHost
    jsonPointers:
      - /status
      - /metadata/annotations/baremetalhost.metal3.io~1status
```

## Differences from Helm Install

### Hook Execution

- **Helm**: `helm.sh/hook: pre-install` runs the reset-bmcs Job before chart resources
- **ArgoCD** (`deployMethod: "argocd"`): `argocd.argoproj.io/hook: PreSync` runs the reset-bmcs Job in the PreSync phase before the main sync
- **Result**: Same behavior, different mechanism

### Resource Deletion

- **Helm**: `helm.sh/resource-policy: keep` prevents deletion
- **ArgoCD**: `argocd.argoproj.io/sync-options: Prune=false` prevents deletion
- **Chart**: Includes both annotations for compatibility

### NOTES.txt Behavior

- **Helm install**: NOTES displayed after install with `lookup` results
- **ArgoCD**: NOTES not displayed; `lookup` may not work during template rendering
- **Workaround**: Check InfraEnv directly for ISO URL:
  ```bash
  kubectl get infraenv my-cluster -n my-cluster -o jsonpath='{.status.isoDownloadURL}'
  ```

## Monitoring

```bash
# Watch application sync
argocd app get my-cluster --watch

# View sync status
argocd app sync my-cluster

# View resources by sync wave
argocd app resources my-cluster
```

## Troubleshooting

### Application OutOfSync

Check if status fields are causing drift:

```bash
argocd app diff my-cluster
```

Add `ignoreDifferences` for operator-managed fields (see above).

### Resources Not Creating in Order

Verify sync waves are being respected:

```bash
argocd app resources my-cluster
```

Resources should appear in wave order: namespace (-5) first, then secrets (-4 to -2), core resources (0), then provisioning Jobs (1-5).

### PreSync Hook Not Running

Ensure `deployMethod: "argocd"` is set. Without it, the reset-bmcs resources use `helm.sh/hook` annotations which ArgoCD strips during rendering.
