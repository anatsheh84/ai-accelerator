# 3scale Operator

Red Hat 3scale API Management deployment for OpenShift.

## Overview

This component deploys 3scale using a GitOps approach with the following features:

- **Operator Installation**: Deploys the 3scale operator (channel: threescale-2.14)
- **APIManager Instance**: Creates and configures the 3scale APIManager
- **Custom Policies**: Includes a remove-bearer policy for token handling
- **Route Patching**: Automatically configures timeout settings on routes
- **Status Checking**: Validates APIManager readiness before proceeding

## Structure

```
3scale/
├── operator/
│   ├── base/              # Base operator installation
│   └── overlays/
│       └── stable-2.14/   # 3scale 2.14 channel
├── instance/
│   ├── base/              # Base APIManager configuration
│   └── overlays/
│       └── default/       # Default instance configuration
└── aggregate/
    └── overlays/
        └── stable-2.14/   # Combines operator + instance
```

## Deployment Sync Waves

The deployment follows these sync waves to ensure proper ordering:

- **Wave 0**: Namespace creation
- **Wave 1**: Operator installation + RBAC for checkers
- **Wave 2**: APIManager + CustomPolicyDefinition
- **Wave 3**: APIManager readiness check
- **Wave 4**: Route patching

## Storage Requirements

3scale requires persistent storage provided by ODF (OpenShift Data Foundation):

- **Storage Class**: `ocs-storagecluster-cephfs`
- **Used For**: System file storage (assets, attachments, etc.)

## Configuration

### Wildcard Domain

The wildcard domain is automatically configured via the bootstrap script, which:
1. Detects the cluster's route domain: `oc get ingresses.config/cluster -o jsonpath='{.spec.domain}'`
2. Updates the ConfigMap in `instance/base/kustomization.yaml`
3. Applies the domain to the APIManager CR via Kustomize replacements

### Custom Policies

The deployment includes a **remove-bearer** policy that strips the "Bearer " prefix from Authorization headers, useful for backends that expect raw tokens.

## Access

After deployment, access 3scale admin portal:

```bash
# Get admin URL
oc get route -n 3scale | grep admin

# Default credentials
Username: admin
Password: <set during bootstrap>
```

## Dependencies

- OpenShift Data Foundation (ODF) for storage
- OpenShift Pipelines (if using CMS uploads - future)

## External Images

This deployment uses the following external images:
- `quay.io/agnosticd/ee-multicloud:v0.1.2` - For checker and patcher jobs

## Troubleshooting

### APIManager Not Ready

```bash
# Check APIManager status
oc get apimanager apimanager -n 3scale -o yaml

# Check operator logs
oc logs -n 3scale deployment/threescale-operator-controller-manager-v2

# Check all pods
oc get pods -n 3scale
```

### Storage Issues

```bash
# Verify storage class exists
oc get sc ocs-storagecluster-cephfs

# Check PVC
oc get pvc -n 3scale
```

## Future Enhancements

- API Product configurations
- Developer Portal customization (CMS)
- Multi-environment overlays
- Monitoring and alerting integration
