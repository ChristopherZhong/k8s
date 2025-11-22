# Kubernetes Home Lab

A comprehensive guide to building a production-ready Kubernetes home lab, starting with a single Raspberry Pi 5 and growing incrementally. This project documents the journey of learning Kubernetes while building a well-structured cluster for home use.

## 🎯 Project Goals

- **Document Home Lab Build**: Step-by-step instructions for building a Kubernetes home lab from scratch
- **Learn Kubernetes**: Hands-on experience with Kubernetes concepts, tools, and best practices
- **Incremental Growth**: Start simple with one node and expand the cluster over time
- **Production Practices**: Implement monitoring, security, and operational best practices

## 🏗️ Architecture Overview

The home lab starts with a single Raspberry Pi 5 and can be expanded to a multi-node cluster:

- **Hardware**: Raspberry Pi 5 with M.2 NVMe SSD for improved I/O performance
- **OS**: Raspberry Pi OS (headless configuration)
- **Kubernetes Distribution**: K3s (lightweight, perfect for edge/IoT scenarios)
- **Storage**: Local storage with expansion to distributed storage solutions
- **Networking**: Traefik for ingress and load balancing

## 🚀 Getting Started

### Prerequisites

- Raspberry Pi 5 (8 GB or more RAM recommended)
- M.2 NVMe SSD (512 GB or larger recommended)
- MicroSD card (for initial boot)
- Network connection (Ethernet preferred)
- Another computer for SSH access

### Step 1: Prepare Raspberry Pi OS

The following instructions are for preparing the SSD. We assume that the SD card is already flashed. If not, the following instructions may still work.

1. **Download Raspberry Pi Imager** from https://www.raspberrypi.com/software/.

1. **Flash Raspberry Pi OS**
   - Use Raspberry Pi Imager to flash Raspberry Pi OS Lite (64-bit) to your SSD.
   - **Important**: Configure the following in Advanced Options:
     - **Hostname**: Set to `raspberrypi-1` (or your preferred hostname)
     - **Enable SSH**: Use public-key authentication only (recommended)
     - **Set username**: `pi` (or your preferred username)
     - **SSH Public Key**: Paste your Ed25519 public key (see SSH key generation below)
     - Configure WiFi if needed (Ethernet recommended)
     - Set locale settings

1. **Generate SSH Key Pair** (if you don't have one)
   ```shell
   # On your local machine, generate Ed25519 key (most secure and performant)
   $ ssh-keygen -t ed25519 -C "you@example.com"
   
   # Display your public key to copy into Raspberry Pi Imager
   $ cat ~/.ssh/id_ed25519.pub
   ```

   **Note**: Copy the entire public key output and paste it into the SSH Public Key field in Raspberry Pi Imager.

1. **Booting from the M.2 NVMe SSD**. Follow the official instructions on how to boot from the SSD [here](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#nvme-ssd-boot).

1. **Initial Boot and Setup**
   ```shell
   # SSH into your Pi using the hostname or IP
   $ ssh pi@raspberrypi-1.local
   
   # Update the system
   $ sudo apt update && sudo apt full-upgrade   
   ```

1. **Install Git**
   ```shell
   # Install Git to clone this repository
   $ sudo apt install git
   ```

1. **Configure Git**. First configure the user name and email.
   ```shell
   $ git config --global user.name "John Doe"
   $ git config --global user.email johndoe@example.com
   ```
   For more details see the [docs](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup).

### Step 2: Install Kubernetes (K3s)

[K3s](https://k3s.io/) is chosen for its lightweight nature and excellent ARM64 support.

1. **Enable cgroups**
   K3s needs `cgroups` to be enabled (see https://docs.k3s.io/installation/requirements?os=pi).
   Append `cgroup_memory=1 cgroup_enable=memory` to `/boot/firmware/cmdline.txt` and reboot.

1. **Install K3s**. Follow the Quick-Start [guide](https://docs.k3s.io/quick-start).
   ```shell
   # Install K3s
   $ curl -sfL https://get.k3s.io | sh -
   
   # Verify installation
   $ sudo systemctl status k3s
   
   # Check nodes
   $ sudo kubectl get nodes
   ```

1. **Configure kubectl for non-root user**
   ```shell
   # Copy kubeconfig to user directory
   $ mkdir -p ~/.kube
   $ sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
   $ sudo chown $USER:$USER ~/.kube/config
   
   # Test access
   $ kubectl get all --all-namespaces
   ```

1. **Optional: Install kubectl on your local machine**
   ```shell
   # Copy the kubeconfig from Pi to your local machine
   $ scp pi@192.168.1.100:/etc/rancher/k3s/k3s.yaml ~/.kube/config-homelab
   
   # Edit the server URL in the config file to point to your Pi's IP
   # Then use: export KUBECONFIG=~/.kube/config-homelab
   ```

### Step 3: Essential Cluster Components

#### Install Helm (Package Manager)
```shell
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

#### Setup Ingress with Traefik
K3s comes with Traefik by default, but we'll configure it properly:

```shell
# Apply Traefik dashboard configuration
kubectl apply -f kube-system/traefik.yaml

# Access dashboard at: http://traefik.raspberrypi.local
# (Add entry to /etc/hosts: <PI_IP> traefik.raspberrypi.local)
```

## 📊 Monitoring and Observability

### Grafana Setup
Deploy Grafana for monitoring and visualization:

```shell
# Create grafana namespace
kubectl create namespace grafana

# Deploy Grafana
kubectl apply -f grafana/grafana.yaml

# Access Grafana at: http://<PI_IP>:3000
# Default credentials: admin/admin
```

## 🔐 Security and Secrets Management

### Kubernetes Dashboard
```shell
# Install Kubernetes Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Create admin user
kubectl apply -f kubernetes-dashboard/kubernetes-dashboard-admin-user.yml
kubectl apply -f kubernetes-dashboard/kubernetes-dashboard-admin-user-binding.yml

# Get access token
kubectl -n kubernetes-dashboard create token admin-user
```

### Infisical (Secrets Management)
```shell
# Add Infisical Helm repository
helm repo add infisical-helm-charts 'https://dl.cloudsmith.io/public/infisical/helm-charts/helm/charts/'
helm repo update

# Create namespace
kubectl create namespace infisical

# Apply secrets
kubectl apply -f infisical/infisical-secrets.yaml

# Install Infisical
helm install infisical infisical-helm-charts/infisical -f infisical/values.yaml -n infisical
```

## 💾 Database Services

### PostgreSQL with CloudNativePG
```shell
# Install CloudNativePG operator
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.20/releases/cnpg-1.20.0.yaml

# Create postgres namespace
kubectl create namespace postgres

# Deploy PostgreSQL cluster
kubectl apply -f postgres/k8s.yaml
```

## 🔧 Useful Commands

### Cluster Management
```shell
# View all resources across namespaces
kubectl get all --all-namespaces

# Describe a problematic pod
kubectl describe pod <pod-name> -n <namespace>

# View logs
kubectl logs <pod-name> -n <namespace> -f

# Port forward for local access
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>

# Scale deployment
kubectl scale deployment <deployment-name> --replicas=3 -n <namespace>
```

### Troubleshooting
```shell
# Check node resources
kubectl top nodes

# Check pod resources
kubectl top pods --all-namespaces

# View events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check cluster info
kubectl cluster-info

# Verify K3s status
sudo systemctl status k3s
```

## 📈 Next Steps

### Expanding the Cluster
1. **Add Worker Nodes**: Install additional Raspberry Pis and join them to the cluster
2. **High Availability**: Configure multiple master nodes for production resilience
3. **Storage Solutions**: Implement Longhorn or other distributed storage
4. **GitOps**: Set up ArgoCD or Flux for automated deployments
5. **Service Mesh**: Explore Istio or Linkerd for advanced networking

### Advanced Monitoring
1. **Prometheus Stack**: Deploy full monitoring stack with Prometheus, AlertManager
2. **Log Aggregation**: Set up ELK stack or Loki for centralized logging
3. **Distributed Tracing**: Implement Jaeger for application tracing

### Security Hardening
1. **Network Policies**: Implement Kubernetes network policies
2. **Pod Security Standards**: Configure pod security contexts
3. **RBAC**: Fine-tune role-based access control
4. **Certificate Management**: Set up cert-manager for TLS automation

## 📚 Learning Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [K3s Documentation](https://docs.k3s.io/)
- [Raspberry Pi Kubernetes Cluster](https://github.com/k8s-at-home/awesome-home-kubernetes)
- [CloudNative Landscape](https://landscape.cncf.io/)

## 🤝 Contributing

This is a learning project! Feel free to:
- Suggest improvements to the setup process
- Share your own home lab configurations
- Report issues or corrections
- Add new service configurations

## 📝 Notes

- **Performance**: Raspberry Pi 5 with NVMe SSD provides excellent performance for a home lab
- **Power**: Consider a UPS for production-like uptime
- **Networking**: Static IP configuration recommended for stability
- **Backups**: Regular etcd backups are crucial for cluster recovery

---

*Last updated: 2024-10-24*
*Cluster Status: Single Node (Raspberry Pi 5)*