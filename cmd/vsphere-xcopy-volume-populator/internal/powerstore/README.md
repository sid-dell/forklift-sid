# Dell PowerStore XCOPY Configuration

This document describes the configuration required to enable XCOPY (extended copy) operations with Dell PowerStore storage arrays for the Forklift vSphere XCOPY Volume Populator.

## Overview

The PowerStore XCOPY functionality enables efficient storage-to-storage data migration by offloading copy operations to the PowerStore array. This reduces network overhead and improves migration performance during virtual machine migrations.

## Prerequisites and PowerStore Configuration

- Increase the Max Lun to 16K as per [PowerStore: Volumes Mapped Through REST API By an Application Are Not Visible on the host | Dell India](https://www.dell.com/support/kbdoc/en-in/000199943/powerstore-volumes-mapped-through-rest-by-an-appliance-are-not-visible-on-esxi)

- If required to enforce VM concurrently to 1, set the controller_max_vm_inflight to 1. 
  - oc patch forkliftcontrollers.forklift.konveyor.io forklift-controller -n openshift-mtv  --type='merge' -p '{"spec":{"controller_max_vm_inflight":1}}'
- Apply Dell Best practice setting on Host side for optimital XCOPY performorance
  - [PowerStore: How to configure ESXi hosts for optimal XCOPY performance | Dell US](https://www.dell.com/support/kbdoc/en-us/000202386/powerstore-how-to-configure-esxi-hosts-for-optimal-xcopy-performance)
  - https://infohub.delltechnologies.com/en-us/l/dell-powerstore-vmware-vsphere-best-practices-2/xcopy/
  - https://drewtonnesen.wordpress.com/2025/03/20/pstore-vm-bps/#more-7316
- [Apply Dell Best practice multi path setting on Openshift side](https://infohub.delltechnologies.com/en-us/l/dell-powerstore-vmware-vsphere-best-practices-2/introduction-5016/)

- Also refer to latest version of  [Chapter 2. Prerequisites | Installing and using the Migration Toolkit for Virtualization | Migration Toolkit for Virtualization | 2.1 | Red Hat Documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.1/html/installing_and_using_the_migration_toolkit_for_virtualization/prerequisites)


## OpenShift Configuration

### 1. Create PowerStore Credentials Secret

Create a secret containing PowerStore API credentials in the same namespace as the `StorageMap` object (typically `openshift-mtv`).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: powerstore-xcopy-secret
  namespace: openshift-mtv
type: Opaque
stringData:
  STORAGE_HOSTNAME: https://powerstore.example.com/api/rest
  STORAGE_USERNAME: admin
  STORAGE_PASSWORD: your-secure-password
  STORAGE_SKIP_SSL_VERIFICATION: 'true'
```

#### Secret Configuration Parameters

| Key | Required | Description | Example |
|-----|----------|-------------|---------|
| `STORAGE_HOSTNAME` | Yes | PowerStore management API URL | `https://10.1.2.14/api/rest` |
| `STORAGE_USERNAME` | Yes | PowerStore API username | `admin` | # user ocpuser refer PST recently vcreated with non admin role
| `STORAGE_PASSWORD` | Yes | PowerStore API password | `Password` |
| `STORAGE_SKIP_SSL_VERIFICATION` | Yes | Skip SSL certificate verification (set to `'true'` or `'false'`) | `'true'` |

### 2. Verify Secret Creation

```bash
oc get secret powerstore-xcopy-secret -n openshift-mtv -o yaml
```
Note: Keys will be readable but values will be encrypted in the output yaml.

## StorageMap Configuration

When creating a `StorageMap` for PowerStore XCOPY, reference the secret created above.

**YAML based Configuration Steps:**

```yaml
apiVersion: forklift.konveyor.io/v1beta1
kind: StorageMap
metadata:
  name: powerstore-storagemap
  namespace: openshift-mtv
spec:
  source:
    provider: vsphere
    # ... source configuration
  destination:
    provider: powerstore
    secretRef:
      name: powerstore-xcopy-secret
      namespace: openshift-mtv
```
**UI Configuration Steps:**

When using the OpenShift console to create a StorageMap, follow these steps in the Offload configuration section:

1. **Offload plugin**: Select `vSphere XCOPY` from the dropdown list
2. **Storage secret**: Choose your PowerStore secret (e.g., `powerstore-xcopy-secret`) from the dropdown list
3. **Storage product**: Select `Dell PowerStore` from the dropdown list

## Validation and Testing

### Pre-Migration Validation
Before starting migrations, validate the setup:

```bash
# 1. Verify PowerStore connectivity from the OpenShift cluster
oc run powerstore-test --image=curlimages/curl --rm -i --restart=Never -- \
  curl -k -u $STORAGE_USERNAME:$STORAGE_PASSWORD $STORAGE_HOSTNAME/rest_api/v1/cluster

# 2. Check XCOPY support from the OpenShift cluster  
oc run powerstore-test --image=curlimages/curl --rm -i --restart=Never -- \
  curl -k -u $STORAGE_USERNAME:$STORAGE_PASSWORD $STORAGE_HOSTNAME/rest_api/v1/volume/capabilities

# 3. Verify StorageMap status, replace <powerstore-storagemap> with your actual StorageMap name
oc get storagemap <powerstore-storagemap> -n openshift-mtv -o yaml

# 4. Check Forklift operator and VSphereXcopyVolumePopulator status
oc get migrations -n openshift-mtv | grep plan-1d50gb-10vm-1snap
```

### Test Migration
Perform a small test migration with 1 VM with 1GB disk to validate the setup. Verify successful completion and data integrity
