# Kubernetes Cluster Setup with Ansible

This Ansible project automates the deployment of a Kubernetes cluster on Debian-based systems using containerd as the container runtime and Calico as the CNI (Container Network Interface) plugin.

## 🏗️ Architecture

- **Kubernetes Version**: 1.30.9
- **Container Runtime**: containerd 1.6.21
- **CNI Plugin**: Calico v3.30.3
- **Pod Network**: 192.168.0.0/16
- **Service Network**: 10.96.0.0/12
- **Control Plane Endpoint**: titimod.ir:6443

## 📋 Prerequisites

- Debian-based Linux distribution (tested on Debian)
- SSH access to all nodes with sudo privileges
- Ansible installed on the control machine
- At least 2GB RAM and 2 CPU cores per node
- Network connectivity between all nodes on private subnet 

## 🗂️ Project Structure

```
├── inventory
│   └── hosts
│       └── es.ini
├── README.md
├── roles
│   ├── calico-cni
│   │   ├── defaults
│   │   │   └── main.yml
│   │   └── tasks
│   │       └── main.yml
│   ├── common
│   │   └── tasks
│   │       └── main.yml
│   ├── containerd
│   │   ├── defaults
│   │   │   └── main.yml
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   └── main.yml
│   │   └── vars
│   │       └── main.yml
│   ├── initiialize
│   │   ├── files
│   │   │   └── kubeadm_config
│   │   ├── tasks
│   │   │   └── main.yaml
│   │   ├── templates
│   │   └── vars
│   │       └── main.yaml
│   ├── join-masters
│   │   └── tasks
│   │       └── main.yml
│   ├── join-worker
│   │   ├── defaults
│   │   │   └── main.yaml
│   │   ├── tasks
│   │   │   └── main.yaml
│   │   └── vars
│   │       └── main.yaml
│   ├── kubectl
│   │   └── tasks
│   │       └── main.yml
│   └── kubelet
│       ├── defaults
│       │   └── main.yml
│       ├── tasks
│       │   └── main.yml
│       └── templates
│           └── kubelet.yml.j2
├── site.yml
└── WORKER_NODES.md        
```

## 📝 Configuration

Before deploying the cluster, you'll need to configure your inventory and customize variables according to your environment. the env files are located into hosts and defualt and var in each role

### Inventory Setup

Edit `inventory/hosts/es.ini` with your node details:
here is the example base on our architect 

```ini
[all]
k8s-master-1 ansible_host=37.32.9.27 ansible_user=debian private_ip=192.168.0.11
k8s-worker-1 ansible_host=37.32.9.225 ansible_user=debian private_ip=192.168.0.12
k8s-worker-2 ansible_host=37.32.9.128 ansible_user=debian private_ip=192.168.0.13

[k8s_master]
k8s-master-1

[k8s_node]
k8s-worker-1
k8s-worker-2

[k8s_master_1]
k8s-master-1

[all:vars]
node_type=worker
kubernetes_kubelet_extra_args=""
```

### Cluster Configuration

The cluster is configured with the following settings in `roles/initiialize/files/kubeadm_config`:

- **Cluster Name**: demo-cluster
- **Kubernetes Version**: 1.30.9
- **Pod Subnet**: 192.168.0.0/16
- **Service Subnet**: 10.96.0.0/12
- **Control Plane Endpoint**: titimod.ir:6443
- **CRI Socket**: unix:///var/run/containerd/containerd.sock

## 🚀 Deployment

### Full Cluster Deployment

Deploy the complete Kubernetes cluster:

```bash
ansible-playbook -i inventory/hosts/es.ini site.yml
```

### Step-by-Step Deployment

You can also run individual components:

1. **System preparation and containerd**:
```bash
ansible-playbook -i inventory/hosts/es.ini site.yml --tags "common,containerd"
```

2. **Kubernetes binaries and kubelet**:
```bash
ansible-playbook -i inventory/hosts/es.ini site.yml --tags "kubelet,kubectl"
```

3. **Initialize master node**:
```bash
ansible-playbook -i inventory/hosts/es.ini site.yml --tags "initialize"
```

4. **Install CNI**:
```bash
ansible-playbook -i inventory/hosts/es.ini site.yml --tags "calico"
```

5. **Join worker nodes**:
```bash
ansible-playbook -i inventory/hosts/es.ini site.yml --tags "join-worker"
```

## 📦 Components

### Roles Overview

1. **common**: System preparation and basic configuration
2. **containerd**: Container runtime installation and configuration
3. **kubelet**: Kubelet service setup and configuration
4. **kubectl**: kubectl binary installation
5. **initiialize**: Kubernetes cluster initialization
6. **calico-cni**: Calico CNI plugin installation
7. **join-masters**: Additional master nodes joining (HA setup)
8. **join-worker**: Worker nodes joining

### Key Features

- ✅ **Automated binary installation**: Downloads kubectl, kubeadm, kubelet v1.30.0
- ✅ **Container runtime**: containerd with proper CRI configuration
- ✅ **CNI networking**: Calico v3.30.3 with automatic pod networking
- ✅ **Security**: RBAC enabled, TLS certificates managed
- ✅ **High availability**: Support for multiple master nodes (but we use as one master and 2 worker so its not ha but it support ha too)
- ✅ **Node management**: Automatic worker node joining and verification
- ✅ **Idempotent**: Safe to run multiple times

## 🔧 Customization

This project provides several customization options to adapt the Kubernetes cluster deployment to your specific environment and requirements. You can modify variables, versions, and configuration files to suit your needs.

### Variables

#### Playbook Variables (site.yml)
- `containerd_version`: "1.6.21" - Version of containerd to install
- `docker_repo`: "https://download.docker.com/linux/debian" - Docker repository URL for containerd

#### Inventory Variables (inventory/hosts/es.ini)
- `ansible_host`: Public IP address of the node
- `ansible_user`: SSH user (default: debian)
- `private_ip`: Private IP address for cluster communication
- `node_type`: "worker" - Default node type
- `kubernetes_kubelet_extra_args`: "" - Additional kubelet arguments

#### containerd role
- `containerd_config_default_write`: true - Write default containerd config
- `containerd_config_file`: "/etc/containerd/config.toml" - Path to containerd config
- `kubernetes_version`: "1.30" - Kubernetes major version
- `kubernetes_repo_url`: Repository URL for Kubernetes packages
- `kubernetes_key_url`: GPG key URL for Kubernetes repository
- `keyring_path`: "/etc/apt/keyrings/kubernetes-apt-keyring.gpg" - GPG keyring path

#### kubelet role
- `kubernetes_packages`: List of Kubernetes packages to install
- `kubernetes_kubelet_extra_args`: "" - Extra arguments for kubelet
- `kubelet_environment_file_path`: "/etc/default/kubelet" - Kubelet environment file

#### join-worker role
- `cluster_domain`: "titimod.ir" - Cluster domain name (currently unused)
- `cluster_ip`: "172.16.25.2" - Cluster IP address (currently unused)
- `kubelet_extra_args`: "--node-ip={{ private_ip }} --hostname-override={{ inventory_hostname }}" - Kubelet configuration
- `worker_node_ready_timeout`: 300 - Timeout for node ready check (seconds)
- `worker_join_retries`: 3 - Number of join retry attempts
- `kubelet_service_dir`: "/etc/systemd/system/kubelet.service.d" - Kubelet service directory
- `kubelet_config_file`: "/etc/kubernetes/kubelet.conf" - Kubelet config file path
- `kubeadm_join_timeout`: 300 - Timeout for kubeadm join operation
- `node_ready_check_retries`: 10 - Number of retries for node ready check
- `node_ready_check_delay`: 30 - Delay between node ready checks (seconds)

#### calico-cni role
- `calico_crds_url`: "https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/operator-crds.yaml" - Calico CRDs manifest URL
- `tigera_operator_url`: "https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/tigera-operator.yaml" - Tigera operator manifest URL
- `calico_custom_resources_url`: "https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/custom-resources.yaml" - Calico custom resources URL
- `pod_subnet`: "192.168.0.0/16" - Pod network subnet

#### initiialize role
- `pod_network_cidr`: "192.168.0.0/16" - Pod network CIDR
- `private_ip_controlplane`: "172.16.25.2" - Private IP of control plane
- `public_ip_controlplane`: "37.32.9.27" - Public IP of control plane
- `admin_conf_dir`: "/etc/kubernetes/admin.conf" - Admin kubeconfig path
- `k8s_config`: "/etc/kubernetes/kubelet.conf" - Kubelet config path

### Modifying Kubernetes Version

To change the Kubernetes version, update the following files:
1. `roles/containerd/tasks/main.yml` - kubectl, kubeadm, kubelet download URLs
2. `roles/initiialize/files/kubeadm_config` - kubernetesVersion field

## 🎯 Post-Deployment

### Verification

Check cluster status:
```bash
# On the master node
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
```

### Accessing the Cluster

The kubeconfig is automatically set up for the `debian` user on the master node:
```bash
# On master node
export KUBECONFIG=/home/debian/.kube/config
kubectl get nodes
```

### Adding More Worker Nodes

1. Add the new node to `inventory/hosts/es.ini` under `[k8s_node]`
2. Run the worker deployment:
```bash
ansible-playbook -i inventory/hosts/es.ini site.yml --limit=new-worker-node
```

## 🐛 Troubleshooting

### Common Issues

1. **Node not ready**:
   - Check kubelet logs: `journalctl -u kubelet`
   - Verify containerd: `systemctl status containerd`
   - Check CNI: `kubectl get pods -n calico-system`

2. **Join token expired**:
   - Generate new token: `kubeadm token create --print-join-command`

3. **Network issues**:
   - Verify connectivity to control plane endpoint
   - Check firewall rules of your cloud provider
   - Ensure pod subnet doesn't conflict with host network ip range

### Logs

- Kubelet: `journalctl -u kubelet -f`
- containerd: `journalctl -u containerd -f`
- Pods: `kubectl logs -n kube-system <pod-name>`

## 🔒 Security Considerations

- Join tokens are temporary (24-hour expiration)
- TLS certificates are automatically managed
- RBAC is enabled by default
- Worker nodes have limited cluster access
- Regular security updates recommended

## 📚 Documentation

- [Worker Nodes Documentation](WORKER_NODES.md)
- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [Calico Documentation](https://docs.projectcalico.org/)
- [containerd Documentation](https://containerd.io/)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with a clean environment
5. Submit a pull request

## 📄 License

This project is open source and available under the [MIT License](LICENSE).