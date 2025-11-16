# Talos Linux Configuration

This directory contains example Talos Linux configurations for the homelab cluster.

## Overview

- **Talos Version**: v1.11.1
- **Kubernetes Version**: v1.34.0
- **CNI**: Default (Flannel alternative configuration available)
- **Control Plane Endpoint**: Configure your own endpoint

## Files

- `controlplane.yaml.example` - Control plane node configuration template
- `worker.yaml.example` - Worker node configuration template
- `patches/` - Configuration patches (e.g., CNI customization)

## Setup Instructions

1. **Generate fresh configurations**:
   ```bash
   talosctl gen config my-cluster https://<YOUR-CONTROL-PLANE-IP>:6443
   ```

2. **Apply patches if needed**:
   - See `patches/` directory for examples
   - Merge patches using `talosctl gen config --config-patch`

3. **Apply configurations**:
   ```bash
   talosctl apply-config --insecure \
     --nodes <NODE-IP> \
     --file controlplane.yaml
   ```

## Important Security Notes

⚠️ **NEVER commit actual Talos configurations to Git!**

Actual configurations contain:
- Machine tokens
- CA private keys
- Kubernetes certificates and keys
- Bootstrap tokens
- Cluster secrets
- Encryption keys

These should be stored securely (e.g., password manager, vault, encrypted storage).

## Configuration Features Enabled

- **KubePrism**: Local load balancer on port 7445
- **Host DNS**: CoreDNS forwarding with host DNS caching
- **RBAC**: Role-based access control
- **Disk Quotas**: Project quota support
- **Control Plane Scheduling**: Allows pods on control plane nodes
- **Stable Hostname**: Predictable node naming

## Network Configuration

Default pod and service subnets:
- **Pod Subnet**: `10.244.0.0/16`
- **Service Subnet**: `10.96.0.0/12`
- **DNS Domain**: `cluster.local`

Adjust these in your generated configuration as needed.

## References

- [Talos Linux Documentation](https://www.talos.dev/latest/)
- [Talos Configuration Reference](https://www.talos.dev/latest/reference/configuration/)
- [Getting Started Guide](https://www.talos.dev/latest/introduction/getting-started/)
