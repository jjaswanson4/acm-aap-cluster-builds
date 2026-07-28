# Installation Guide

This guide provides step-by-step instructions for deploying an OpenShift cluster using this Helm chart.

## Prerequisites

1. **Helm 3.x** installed
2. **kubectl** or **oc** CLI configured to access your OpenShift cluster
3. **Red Hat OpenShift pull secret** from https://console.redhat.com/openshift/install/pull-secret
4. **Access to baremetal hosts** with BMC (Redfish/IPMI)
5. **OpenShift Advanced Cluster Management (ACM)** or **Assisted Installer Operator** deployed

## Step 1: Prepare Your Values File

1. Copy the example values file:
   ```bash
   cp values-example.yaml my-cluster-values.yaml
   ```

2. Edit `my-cluster-values.yaml` and update:
   - `clusterName`: Your cluster name
   - `namespace`: Namespace for cluster resources
   - `baseDomain`: Your base domain
   - `pullSecret.dockerconfigjson`: Your Red Hat pull secret (base64 encoded)
   - `sshPublicKey`: Your SSH public key
   - `agentClusterInstall.apiVIP`: API virtual IP
   - `agentClusterInstall.ingressVIP`: Ingress virtual IP
   - `nodes[]`: Array of your baremetal nodes with:
     - BMC addresses and credentials
     - MAC addresses
     - Network configuration (IPs, gateway, DNS)
     - Root device WWN

## Step 2: Validate Your Configuration

Render the templates without installing to verify:

```bash
helm template my-cluster . -f my-cluster-values.yaml
```

Check for any errors or unexpected values.

## Step 3: Install the Chart

Install the chart with your custom values:

```bash
helm install my-cluster . -f my-cluster-values.yaml
```

You can also install from the parent directory:

```bash
helm install my-cluster ./provisioning/charts/cluster -f my-values.yaml
```

## Step 4: Monitor Installation Progress

### Check Overall Status

```bash
# View all resources in the cluster namespace
oc get all -n <namespace>

# Check cluster deployment
oc get clusterdeployment -n <namespace>

# Check agent cluster install
oc get agentclusterinstall -n <namespace>

# Check infraenv (contains ISO download URL)
oc get infraenv -n <namespace>

# Check baremetal hosts
oc get baremetalhosts -n <namespace>
```

### Get ISO Download URL

Once the InfraEnv is ready, get the ISO download URL:

```bash
oc get infraenv <cluster-name> -n <namespace> -o jsonpath='{.status.isoDownloadURL}'
```

The ISO will be automatically used by the BareMetalHost resources.

### Monitor Installation Logs

```bash
# View assisted-service logs
oc logs -n multicluster-engine -l app=assisted-service -f

# Watch cluster install progress
watch oc get agentclusterinstall -n <namespace>
```

### Check Node Discovery

```bash
# List discovered agents
oc get agents -n <namespace>

# View agent details
oc describe agent <agent-name> -n <namespace>
```

## Step 5: Access Your Cluster

Once installation completes, access your cluster:

### Get Admin Credentials

```bash
# Get admin password
oc extract secret/admin-password -n <namespace> --to=-

# Get kubeconfig
oc extract secret/<cluster-name>-admin-kubeconfig -n <namespace> --to=- > kubeconfig

# Use the kubeconfig
export KUBECONFIG=./kubeconfig
oc get nodes
```

### Access Web Console

Navigate to:
```
https://console-openshift-console.apps.<cluster-name>.<base-domain>
```

Login with username `kubeadmin` and the password from the admin-password secret.

## Troubleshooting

### Nodes Not Booting

1. Check BareMetalHost status:
   ```bash
   oc get bmh -n <namespace>
   oc describe bmh <node-name> -n <namespace>
   ```

2. Verify BMC credentials:
   ```bash
   oc get secret <node-name>-bmc -n <namespace> -o yaml
   ```

3. Check BMC connectivity from the cluster

### Installation Stuck

1. Check agent logs:
   ```bash
   oc logs -n <namespace> $(oc get pods -n <namespace> -l job-name=<cluster-name>-<agent-id> -o name)
   ```

2. Check infraenv status:
   ```bash
   oc describe infraenv <cluster-name> -n <namespace>
   ```

3. View cluster validation:
   ```bash
   oc get agentclusterinstall <cluster-name> -n <namespace> -o yaml | grep -A 20 conditions
   ```

### Network Issues

1. Verify NMStateConfig:
   ```bash
   oc get nmstateconfig -n <namespace>
   oc describe nmstateconfig <config-name> -n <namespace>
   ```

2. Check agent network status:
   ```bash
   oc get agent -n <namespace> -o yaml | grep -A 10 inventory
   ```

## Upgrading

To upgrade with new values:

```bash
helm upgrade my-cluster . -f my-cluster-values.yaml
```

**Note**: Some resources (like BareMetalHost) have `helm.sh/resource-policy: keep` annotation and won't be deleted on uninstall.

## Uninstalling

```bash
helm uninstall my-cluster
```

**Important**: This will NOT delete:
- BareMetalHost resources (due to resource policy)
- The namespace (if it contains other resources)

To fully clean up:

```bash
# Delete baremetal hosts
oc delete bmh --all -n <namespace>

# Delete namespace
oc delete namespace <namespace>
```

## Advanced Configuration

### Custom Image Set

To use a different OpenShift version, update:

```yaml
agentClusterInstall:
  imageSetRef: img4.X.Y-x86-64-appsub
```

### Additional Kernel Arguments

Add more kernel arguments:

```yaml
infraEnv:
  kernelArguments:
    - operation: append
      value: "console=ttyS0,115200n8"
    - operation: append
      value: "rd.net.timeout.carrier=30"
```

### Worker Nodes

To add worker nodes, update node definitions and provision requirements:

```yaml
agentClusterInstall:
  provisionRequirements:
    controlPlaneAgents: 3
    workerAgents: 2

nodes:
  - name: worker-0
    role: "worker"
    # ... other worker configuration
```

## Support

For issues specific to this Helm chart, check the chart's README.md.

For OpenShift installation issues, consult:
- Red Hat OpenShift documentation
- OpenShift Advanced Cluster Management documentation
- Assisted Installer Operator documentation
