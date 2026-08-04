# Changelog

All notable changes to this Helm chart are documented in this file.

## [Unreleased]

### Added

#### Additional NTP Sources
- Added `infraEnv.additionalNTPSources` to configure NTP servers on cluster nodes via chrony
- Rendered as `spec.additionalNTPSources` on the InfraEnv CR
- Merged with DHCP-provided NTP servers (option 42) by the Assisted Service
- MachineConfig resources for chrony are generated automatically — no post-install configuration needed
- Optional: omit or leave empty to use DHCP-provided NTP only

#### Governance Standalone Hub Templating Add-on
- Added `addons.governanceStandaloneHubTemplating` to deploy the `governance-standalone-hub-templating` ManagedClusterAddOn on provisioned clusters
- Enables hub-side policy template resolution, allowing policies to reference hub resources (secrets, configmaps) without syncing them to the spoke
- Default: `false` (disabled) — set to `true` to enable
- Added `addons.hubTemplatingSecretAccess` to grant the hub-templating agent read access to secrets in specified hub namespaces
- Creates a shared Role (with `keep` policy) and a per-cluster RoleBinding in each listed namespace
- Common use case: allow governance policies to template in ACS init-bundle secrets from the `stackrox` namespace

#### Proxy Configuration
- Added `agentClusterInstall.installConfigOverrides.proxy` for cluster-wide HTTP/HTTPS proxy settings
- Supports `httpProxy`, `httpsProxy`, and `noProxy` fields
- Rendered conditionally in the `install-config-overrides` annotation on AgentClusterInstall
- Optional: omitted entirely when not present in values -- backward compatible

#### Boot Mode (UEFI / UEFISecureBoot / legacy)
- Added `bootMode` field to BareMetalHost spec
- Configurable globally via `baremetalHostBootMode` (default: `UEFI`)
- Per-node override via `bootMode` on individual node entries (takes precedence over global)
- Supports `UEFI`, `UEFISecureBoot`, and `legacy` values

#### Helm Chart vs. SiteConfig Comparison Document
- Added `CHART-VS-SITECONFIG.md` comparing this chart with the ACM SiteConfig operator
- Covers feature parity, architecture diagrams, and when-to-use guidance

#### Disconnected Install / Mirror Registry Support
- Added conditional rendering of `imageDigestSources` and `additionalTrustBundle` in the AgentClusterInstall `install-config-overrides` annotation
- `imageDigestSources`: array of source-to-mirror mappings for disconnected installs
- `additionalTrustBundle`: CA certificate PEM for mirror registry trust
- Both fields are optional and backward compatible — omitted when not present in values

#### ArgoCD Agent Mode Enrollment
- Added conditional `argocdAgent` feature for enrolling spoke clusters with an ArgoCD principal via mTLS
- Hub-side Job generates client certificates, creates cluster registration secret, and injects spoke-side manifests
- Spoke-side setup Job installs GitOps operator and creates ArgoCD CR with agent mode enabled
- Controlled by `argocdAgent.enabled: "true"` (disabled by default)
- `holdInstallation` now also triggers when `argocdAgent.enabled` is true

#### Flexible Cluster Manifests
- **Breaking Change**: Cluster manifests now accept arbitrary YAML content instead of structured configuration
- Users can now add any Kubernetes/OpenShift manifests to the `clusterManifests` ConfigMap
- Format changed from structured (with predefined fields) to key-value map where:
  - Key: filename (e.g., `99-master-kargs.yaml`)
  - Value: raw YAML manifest content (using `|` multiline string)
- Set `clusterManifests: {}` to disable cluster manifests entirely
- **Migration**: Update your values.yaml to use the new format (see examples in values.yaml and values-example.yaml)

#### BareMetalHost Paused Control
- Added `baremetalHostPaused` value to control the `baremetalhost.metal3.io/paused` annotation
- Default: `"true"` (ACM will NOT automatically manage/provision hosts)
- Set to `"false"` to enable ACM automatic provisioning via BMC
- This gives users explicit control over whether ACM manages baremetal hosts

#### BareMetalHost Resource Policy Control
- Added `baremetalHostResourcePolicy` value to control the `helm.sh/resource-policy` annotation
- Default: `"keep"` (BareMetalHost resources preserved on helm uninstall)
- Set to `"delete"` to allow full cleanup during helm uninstall
- Prevents accidental deletion of physical host state by default

#### Namespace Resource Policy Control
- Added `namespaceResourcePolicy` value to control the `helm.sh/resource-policy` annotation on the namespace
- Default: `"keep"` (namespace preserved on helm uninstall)
- Set to `"delete"` to allow namespace cleanup during helm uninstall
- Prevents accidental deletion of namespace and its resources by default

#### ISO Download URL Display (Post-Install Hook)
- Added post-install hook Job that waits for the InfraEnv ISO URL to be generated
- `helm install` now waits and displays the actual ISO download URL (no manual checking needed)
- Includes ready-to-use wget and curl commands with cluster-specific filename
- Configurable timeout via `isoUrlFetcher.timeout` (default: 600 seconds)
- Hook automatically cleans up after successful completion
- NOTES.txt also uses Helm's `lookup` function as a fallback to check InfraEnv status

### Changed

- Cluster manifests ConfigMap template simplified to use range over map
- BareMetalHost template now uses conditional for resource-policy annotation
- Default values.yaml updated with inline manifest examples
- Documentation updated across README.md, values-example.yaml, and new sections

### Migration Guide

If you have existing values.yaml files using the old cluster manifests format, update them as follows:

**OLD FORMAT:**
```yaml
clusterManifests:
  masterKernelArgs:
    - console=ttyS0,115200n8
  wipeSecondaryDisks:
    enabled: true
    wwns:
      - "/dev/disk/by-id/nvme-eui.000..."
```

**NEW FORMAT:**
```yaml
clusterManifests:
  99-master-kargs.yaml: |
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      labels:
        machineconfiguration.openshift.io/role: "master"
      name: 99-master-kargs
    spec:
      kernelArguments:
        - console=ttyS0,115200n8

  99-wipe-secondary-disks.yaml: |
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      labels:
        machineconfiguration.openshift.io/role: master
      name: 99-wipe-secondary-disks
    spec:
      config:
        ignition:
          version: 3.4.0
        systemd:
          units:
            - contents: |
                [Unit]
                Description=Wipe Secondary Drive
                ...
```

See `values.yaml` for complete examples.

## [0.1.0] - Initial Release

### Added
- Initial Helm chart for deploying OpenShift clusters via AgentClusterInstall
- Support for BareMetalHost resources
- Support for NMStateConfig network configuration
- ClusterDeployment and AgentClusterInstall resources
- InfraEnv resource for ISO generation
- Helm hooks for proper resource ordering
- Pre-install hooks for namespace, secrets, and manifests
- Comprehensive documentation (README.md, INSTALL.md, QUICK-START.md)
- Example values file
- Chart helpers template
- Post-install NOTES.txt

### Features
- Multi-node cluster support via nodes array
- Configurable networking (VIPs, CIDRs, DNS)
- BMC credential management
- Pull secret configuration
- SSH key configuration
- OpenShift capabilities configuration
- CPU partitioning support
- Custom kernel arguments support
- Cluster manifests ConfigMap support
