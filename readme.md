# Kubernetes Cluster Setup Script

A Python-based automation tool for setting up production-ready Kubernetes clusters on Ubuntu/Debian systems using SSH connections. This script handles the complete lifecycle from system preparation to cluster initialization and worker node joining.

## Features

- **Automated cluster bootstrapping** with master and worker nodes
- **SSH-based remote execution** with connection pooling
- **Concurrent worker node setup** for faster deployment
- **Comprehensive logging** to both console and file
- **Inventory-based configuration** similar to Ansible
- **Cleanup and reset capabilities** for cluster teardown
- **Flexible deployment options** (master-only, workers-only, or full cluster)
- **Built-in error handling** and retry mechanisms

## Prerequisites

### System Requirements
- **Operating System**: Ubuntu 18.04+ or Debian 10+
- **Architecture**: x86_64
- **Memory**: Minimum 2GB RAM per node (4GB+ recommended for master)
- **CPU**: Minimum 2 cores per node
- **Disk**: 20GB+ free space per node
- **Network**: All nodes must be able to communicate with each other

### Software Dependencies
- Python 3.7+
- SSH access to all target nodes
- Sudo privileges on target nodes
- Internet connectivity for package downloads

### Network Requirements
- Port 6443 (Kubernetes API server)
- Port 2379-2380 (etcd server client API)
- Port 10250 (kubelet API)
- Port 10257 (kube-controller-manager)
- Port 10259 (kube-scheduler)
- Port 30000-32767 (NodePort services)

## Installation

1. **Clone or download the project files**:
   ```bash
   # Ensure you have these files:
   # - k8s_cluster_setup.py
   # - requirements.txt
   # - inventory.ini (example provided)
   ```

2. **Install Python dependencies**:
   ```bash
   pip3 install -r requirements.txt
   ```

3. **Make the script executable**:
   ```bash
   chmod +x k8s_cluster_setup.py
   ```

## Configuration

### Inventory File Format

Create an `inventory.ini` file with your cluster configuration:

```ini
[all:vars]
ansible_port=22
ansible_python_interpreter=/usr/bin/python3

[master]
master1 ansible_host=10.11.3.251 ansible_user=docker ansible_ssh_pass='your_password'

[workers]
worker1 ansible_host=10.11.4.88 ansible_user=test ansible_ssh_pass='your_password'
worker2 ansible_host=192.168.1.12 ansible_user=ubuntu ansible_ssh_pass='another_password' ansible_port=2222
```

### Configuration Options

#### Global Variables (`[all:vars]`)
- `ansible_port`: SSH port (default: 22)
- `ansible_python_interpreter`: Python interpreter path
- `ansible_user`: Default SSH username
- `ansible_ssh_pass`: Default SSH password

#### Host Specifications
- `ansible_host`: IP address or hostname
- `ansible_user`: SSH username for this host
- `ansible_ssh_pass` / `ansible_password` / `ansible_pass`: SSH password
- `ansible_ssh_private_key_file`: Path to SSH private key (alternative to password)
- `ansible_port`: Custom SSH port for this host

### Authentication Methods

**Password Authentication** (as shown in example):
```ini
worker1 ansible_host=10.11.4.88 ansible_user=test ansible_ssh_pass='mypassword'
```

**SSH Key Authentication**:
```ini
worker1 ansible_host=10.11.4.88 ansible_user=test ansible_ssh_private_key_file='/path/to/private/key'
```

## Usage

### Basic Commands

**Full cluster setup**:
```bash
./k8s_cluster_setup.py -i inventory.ini
```

**Setup master node only**:
```bash
./k8s_cluster_setup.py -i inventory.ini --master-only
```

**Setup worker nodes only** (requires existing master):
```bash
./k8s_cluster_setup.py -i inventory.ini --workers-only
```

**Reset entire cluster**:
```bash
./k8s_cluster_setup.py -i inventory.ini --reset
```

**Verbose logging**:
```bash
./k8s_cluster_setup.py -i inventory.ini -v
```

### Command Line Options

| Option | Description |
|--------|-------------|
| `-i, --inventory` | Path to inventory file (default: inventory.ini) |
| `--reset` | Reset and clean up the cluster |
| `--master-only` | Setup only the master node |
| `--workers-only` | Setup only worker nodes |
| `-v, --verbose` | Enable debug logging |

## What the Script Does

### Preparation Phase (All Nodes)
1. **System Cleanup**: Removes existing Kubernetes installations
2. **Package Installation**: Installs required dependencies
   - Docker and containerd
   - Network utilities
   - Transport packages
3. **System Configuration**:
   - Disables swap
   - Loads kernel modules (overlay, br_netfilter, bridge)
   - Configures sysctl parameters for networking
4. **Container Runtime Setup**: Configures containerd with systemd cgroup driver
5. **Kubernetes Package Installation**: Installs kubelet, kubeadm, and kubectl

### Master Node Initialization
1. **Cluster Initialization**: Runs `kubeadm init` with Flannel network CIDR
2. **Kubeconfig Setup**: Configures kubectl access for root and SSH user
3. **CNI Installation**: Deploys Flannel networking plugin
4. **Join Command Generation**: Creates join command for worker nodes

### Worker Node Setup
1. **Preparation**: Same as master preparation phase
2. **Cluster Joining**: Uses generated join command to join the cluster
3. **Concurrent Processing**: Multiple workers are set up in parallel

### Verification
- Waits for all nodes to reach "Ready" state
- Validates cluster health before completion

## Output Files

- **`k8s_setup.log`**: Detailed execution log
- **`join_command.yml`**: Saved join command for manual worker addition

## Troubleshooting

### Common Issues

**SSH Connection Failures**:
- Verify host connectivity: `ping <host_ip>`
- Check SSH service: `ssh user@host_ip`
- Validate credentials in inventory file

**Permission Denied**:
- Ensure SSH user has sudo privileges
- Verify password/key authentication

**Network Issues**:
- Check firewall rules on all nodes
- Ensure required ports are open
- Verify network connectivity between nodes

**Package Installation Failures**:
- Check internet connectivity
- Verify DNS resolution
- Update package repositories manually

### Debug Mode

Enable verbose logging to see detailed command execution:
```bash
./k8s_cluster_setup.py -i inventory.ini -v
```

### Manual Verification

After setup completion, verify the cluster:
```bash
# On master node
kubectl get nodes
kubectl get pods --all-namespaces
kubectl cluster-info
```

### Recovery Procedures

**Partial Failure Recovery**:
1. Check logs for specific error messages
2. Use `--reset` to clean up
3. Fix underlying issues
4. Re-run setup

**Adding New Workers**:
1. Add worker to inventory file
2. Use `--workers-only` flag
3. Or manually use saved join command from `join_command.yml`

## Security Considerations

- **Password Storage**: Inventory files contain plaintext passwords - secure appropriately
- **SSH Keys**: Prefer SSH key authentication over passwords
- **Network Security**: Configure firewalls to restrict access to required ports only
- **File Permissions**: Restrict access to inventory and key files

## Architecture Details

### Components Installed
- **Kubernetes**: v1.29 (stable release)
- **Container Runtime**: containerd
- **CNI Plugin**: Flannel
- **Pod Network CIDR**: 10.244.0.0/16

### Directory Structure
```
/etc/kubernetes/          # Kubernetes configuration
/var/lib/kubelet/         # Kubelet data
/var/lib/etcd/            # etcd data (master only)
/etc/containerd/          # containerd configuration
/etc/cni/                 # CNI configuration
```

## Contributing

To extend or modify the script:

1. **Add new phases**: Extend the `K8s` class with additional methods
2. **Modify configurations**: Update system configuration in `_sys()` method
3. **Change Kubernetes version**: Update repository URL in `_k8s_pkgs()`
4. **Add CNI options**: Modify `_init_master()` for different network plugins

## License

This project is provided as-is for educational and production use. Modify and distribute according to your organization's policies.

## Support

For issues and questions:
1. Check the troubleshooting section
2. Review log files for error details
3. Verify system requirements and prerequisites
4. Test connectivity and permissions manually

---

**Note**: This script is designed for Ubuntu/Debian systems. For other distributions, package names and system commands may need modification.