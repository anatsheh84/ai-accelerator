# OpenShift Data Foundation (ODF) Operator

OpenShift Data Foundation provides persistent storage for stateful applications.

## Overview

This component deploys ODF using a GitOps approach with the following features:

- **Operator Installation**: Deploys the ODF operator (channel: stable-4.17)
- **Storage System**: Creates the ODF StorageSystem
- **Storage Cluster**: Configures Ceph-based storage with 3 replicas
- **Storage Classes**: Provides multiple storage classes for different use cases

## Structure

```
odf/
├── operator/
│   ├── base/              # Base operator installation
│   └── overlays/
│       └── stable/        # Stable ODF channel
├── instance/
│   ├── base/              # Base StorageCluster configuration
│   └── overlays/
│       └── default/       # Default instance configuration
└── aggregate/
    └── overlays/
        └── default/       # Combines operator + instance
```

## Storage Classes Created

After deployment, the following storage classes will be available:

- `ocs-storagecluster-ceph-rbd` - Block storage (RWO)
- `ocs-storagecluster-ceph-rgw` - Object storage (S3 compatible)
- `ocs-storagecluster-cephfs` - Shared filesystem storage (RWX)

## Configuration

### Storage Requirements

The default configuration provisions:

- **3 OSDs** (Object Storage Daemons) - one per node
- **512 Gi per OSD** using gp3-csi storage class
- **Total raw storage**: ~1.5 TB (3 x 512 Gi)
- **Usable storage**: ~512 Gi (with 3x replication)

### Node Requirements

ODF requires:

- **Minimum 3 worker nodes** for high availability
- **Sufficient CPU/Memory** per node:
  - MDS: 1-3 CPU, 8 Gi RAM
  - MGR: 500m-1 CPU, 3 Gi RAM
  - OSD: 1-2 CPU, 5 Gi RAM per OSD

## Deployment Sync Waves

- **Wave 0**: Namespace creation
- **Wave 1**: Operator installation
- **Wave 2**: StorageSystem and StorageCluster creation

## Verification

After deployment, verify ODF is ready:

```bash
# Check StorageCluster status
oc get storagecluster ocs-storagecluster -n openshift-storage

# Check all pods are running
oc get pods -n openshift-storage

# Verify storage classes
oc get sc | grep ocs-storagecluster

# Check Ceph cluster health
oc rsh -n openshift-storage deployment/rook-ceph-tools
ceph status
ceph osd tree
exit
```

## Storage Usage

### For 3scale

3scale uses the CephFS storage class for system file storage:

```yaml
storageClassName: ocs-storagecluster-cephfs
```

### For Other Applications

```yaml
# Block storage (RWO) - for databases
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-database-pvc
spec:
  storageClassName: ocs-storagecluster-ceph-rbd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi

# Shared filesystem (RWX) - for shared data
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-shared-data-pvc
spec:
  storageClassName: ocs-storagecluster-cephfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
```

## Monitoring

ODF integrates with OpenShift monitoring:

```bash
# View ODF dashboards
oc get route -n openshift-storage prometheus-k8s

# Check Ceph metrics
oc get servicemonitor -n openshift-storage
```

## Troubleshooting

### OSD Pods Not Starting

```bash
# Check OSD pods
oc get pods -n openshift-storage | grep osd

# Check events
oc get events -n openshift-storage --sort-by='.lastTimestamp'

# Verify underlying PVCs
oc get pvc -n openshift-storage
```

### Storage Class Not Available

```bash
# Check if operator is ready
oc get csv -n openshift-storage

# Check StorageCluster status
oc describe storagecluster ocs-storagecluster -n openshift-storage
```

### Ceph Health Issues

```bash
# Access Ceph tools
oc rsh -n openshift-storage deployment/rook-ceph-tools

# Check cluster health
ceph status
ceph health detail

# Check OSD status
ceph osd status
ceph osd tree
```

## Scaling and Maintenance

### Adding More Storage

To increase storage capacity, update the `storageDeviceSets.count` or `storage` value in the StorageCluster CR.

### Removing ODF

**⚠️ Warning**: Removing ODF will delete all data stored in ODF storage classes.

```bash
# Delete StorageCluster (will take time to cleanup)
oc delete storagecluster ocs-storagecluster -n openshift-storage

# Delete operator subscription
oc delete subscription odf-operator -n openshift-storage
```

## References

- [OpenShift Data Foundation Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/)
- [Ceph Documentation](https://docs.ceph.com/)
