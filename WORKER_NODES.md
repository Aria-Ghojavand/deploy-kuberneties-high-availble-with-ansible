# Worker Nodes Management

## Overview
This section covers the complete worker node role that includes:
1. Adding `titimod.ir` entry to `/etc/hosts`
2. Installing required packages
3. Configuring kubelet
4. Joining worker nodes to the cluster
5. Verification and labeling

## Files Structure
```
roles/join-worker/
├── tasks/main.yaml      # Main tasks for joining worker nodes
├── defaults/main.yaml   # Default variables
└── vars/main.yaml       # Role variables
```

## Configuration

### Inventory Configuration
Make sure your `inventory/hosts/es.ini` has the `[k8s_node]` group properly defined:

```ini
[k8s_node]
k8s-worker-0
k8s-worker-1
```

### titimod.ir Resolution
The role automatically adds this entry to `/etc/hosts`:
```
172.16.25.2 titimod.ir
```

## Usage

### Option 1: Full Deployment (Recommended)
Run the complete cluster setup including workers:
```bash
ansible-playbook -i inventory/hosts/es.ini site.yml
```

### Option 2: Worker Nodes Only
If you want to add worker nodes to an existing cluster:
```bash
ansible-playbook -i inventory/hosts/es.ini join-workers.yml
```

### Option 3: Verification Only
To check worker nodes status:
```bash
ansible-playbook -i inventory/hosts/es.ini verify-workers.yml
```

## What the Worker Role Does

1. **System Configuration**
   - Adds `titimod.ir` domain resolution to `/etc/hosts`
   - Installs `socat` package (required for kubeadm join)

2. **Pre-flight Checks**
   - Checks if node is already joined
   - Verifies kubelet configuration exists

3. **Join Process**
   - Generates worker join token from master-1
   - Configures kubelet with proper node-ip and hostname
   - Executes `kubeadm join` command
   - Enables and starts kubelet service

4. **Post-join Verification**
   - Waits for node to become Ready
   - Labels node as worker
   - Displays cluster status

## Troubleshooting

### Common Issues

1. **titimod.ir resolution fails**
   - Check if master-1 (172.16.25.152) is accessible
   - Verify /etc/hosts entry was added correctly

2. **Join command fails**
   - Ensure master cluster is fully initialized
   - Check if tokens are still valid
   - Verify network connectivity to titimod.ir:6443

3. **Node not ready**
   - Check kubelet logs: `journalctl -u kubelet`
   - Verify containerd is running: `systemctl status containerd`
   - Check CNI plugin is installed on master

### Manual Commands

Check worker node status:
```bash
# On master-1
kubectl get nodes -l node-role.kubernetes.io/worker

# Check specific node
kubectl describe node k8s-worker-0
```

Re-generate join token if needed:
```bash
# On master-1
kubeadm token create --print-join-command
```

## Variables

### Default Variables (defaults/main.yaml)
- `cluster_domain`: "titimod.ir"
- `cluster_ip`: "172.16.25.2"
- `worker_node_ready_timeout`: 300 seconds

### Role Variables (vars/main.yaml)
- `kubelet_service_dir`: "/etc/systemd/system/kubelet.service.d"
- `kubelet_config_file`: "/etc/kubernetes/kubelet.conf"
- `node_ready_check_retries`: 10

## Security Notes
- Join tokens are temporary and expire after 24 hours
- Worker nodes only get kubelet.conf, not admin access
- All communication uses TLS certificates

## Integration
This role is designed to work with:
- `common` role (system preparation)
- `containerd` role (container runtime)
- `kubelet` role (kubelet configuration)
- `calico-cni` role (network plugin)