# Quick Start Guide

Deploy an OpenShift cluster in 5 minutes.

## Prerequisites

- Helm 3.x installed
- Access to OpenShift cluster with ACM/Assisted Installer
- Red Hat pull secret
- SSH public key
- Baremetal hosts with BMC access

## Quick Deploy

### 1. Create Your Values File

```bash
cat > my-cluster.yaml << 'EOF'
clusterName: my-cluster
namespace: my-cluster
baseDomain: example.com

pullSecret:
  dockerconfigjson: 'YOUR_PULL_SECRET_HERE'

sshPublicKey: "ssh-ed25519 AAAA... user@host"

agentClusterInstall:
  apiVIP: "192.168.1.10"
  ingressVIP: "192.168.1.11"
  imageSetRef: img4.20.26-x86-64-appsub

nodes:
  - name: master-0
    bmcAddress: redfish-virtualmedia+https://192.168.100.10/redfish/v1/Systems/system
    bmcCredentials:
      username: "admin"
      password: "password"
    bootMACAddress: "aa:bb:cc:dd:ee:01"
    hostname: "master-0.my-cluster.example.com"
    role: "master"
    rootDeviceWWN: "eui.YOUR_WWN_HERE"
    network:
      interface: eno1
      macAddress: "aa:bb:cc:dd:ee:01"
      ipv4:
        address: "192.168.1.21"
        prefixLength: 24
      gateway: "192.168.1.1"
      dns: ["192.168.1.1"]

  # Add 2 more nodes...
EOF
```

### 2. Install the Chart

```bash
helm install my-cluster ./provisioning/charts/cluster -f my-cluster.yaml
```

### 3. Monitor Progress

```bash
# Watch cluster deployment
watch oc get agentclusterinstall -n my-cluster

# Get ISO URL (when ready)
oc get infraenv my-cluster -n my-cluster -o jsonpath='{.status.isoDownloadURL}'

# View installation logs
oc logs -n multicluster-engine -l app=assisted-service -f
```

### 4. Access Your Cluster

```bash
# Wait for installation to complete
oc wait --for=condition=Completed agentclusterinstall/my-cluster -n my-cluster --timeout=60m

# Get kubeconfig
oc extract secret/my-cluster-admin-kubeconfig -n my-cluster --to=- > kubeconfig

# Access cluster
export KUBECONFIG=./kubeconfig
oc get nodes
```

## Key Commands

```bash
# Template without installing
helm template my-cluster ./provisioning/charts/cluster -f my-cluster.yaml

# Dry run
helm install my-cluster ./provisioning/charts/cluster -f my-cluster.yaml --dry-run

# Install
helm install my-cluster ./provisioning/charts/cluster -f my-cluster.yaml

# Upgrade
helm upgrade my-cluster ./provisioning/charts/cluster -f my-cluster.yaml

# Uninstall (keeps BareMetalHosts)
helm uninstall my-cluster

# List releases
helm list
```

## Common Issues

### Nodes not discovered
- Check BMC credentials and connectivity
- Verify `oc get bmh -n <namespace>`

### Network configuration issues
- Verify NMStateConfig with `oc get nmstateconfig -n <namespace>`
- Check MAC addresses match actual hardware

### Installation stuck
- Check logs: `oc logs -n multicluster-engine -l app=assisted-service`
- Verify `oc describe agentclusterinstall <name> -n <namespace>`

## Next Steps

- See `INSTALL.md` for detailed installation guide
- See `README.md` for complete documentation
- See `values-example.yaml` for all configuration options
- See `CONVERSION-NOTES.md` for technical details

## Quick Tips

1. **Copy the default values.yaml as a starting point**
   ```bash
   cp provisioning/charts/cluster/values.yaml my-cluster.yaml
   ```

2. **Validate before installing**
   ```bash
   helm lint provisioning/charts/cluster
   helm template test provisioning/charts/cluster -f my-cluster.yaml | less
   ```

3. **Use meaningful release names**
   - Release name can differ from cluster name
   - Allows multiple clusters in same Helm repository

4. **Check resource ordering**
   - Namespace created first (hook weight -100)
   - Secrets next (hook weights -90 to -70)
   - Main resources in normal install phase

5. **Monitor with kubectl/oc**
   ```bash
   # Watch all resources
   watch oc get all,bmh,agent,agentclusterinstall,infraenv -n <namespace>
   ```
