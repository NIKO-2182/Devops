# Kubernetes Single‑VPC 3‑Node Cluster Setup

Running all 3 nodes inside a single AWS account and VPC simplifies setup significantly. Because all instances share the same private network (172.31.x.x), you do not need WireGuard or public IP configurations for inter-node communication.

---

## Part 1: Single‑VPC Setup Steps

### 1.1 Configure a Shared Security Group

Create a single AWS Security Group and attach it to all 3 EC2 instances:

- **Inbound Rule 1 (Internal Cluster Traffic):** Allow All Traffic where Source is set to the Security Group ID itself (or your VPC CIDR `172.31.0.0/16`).
- **Inbound Rule 2 (SSH Access):** Allow TCP 22 from your IP.
- **Inbound Rule 3 (Optional NodePort access):** Allow TCP 30000–32767 from your IP to access deployed web services.

**Verification:** Run `ping <WORKER_PRIVATE_IP>` from your master instance to verify direct private reachability.

---

### 1.2 Initialize Master Node

SSH into your Master EC2 instance and install default K3s:

```bash
curl -sfL https://get.k3s.io | sh -
```

Retrieve the master's Private IP and Node Token:

```bash
# Private IP
hostname -I | awk '{print $1}'
# Node Token
sudo cat /var/lib/rancher/k3s/server/node-token
```

**Verification:** Run `sudo kubectl get nodes` to confirm the master node reports status **Ready**.

---

### 1.3 Join Worker Nodes

SSH into each Worker EC2 instance and run the join command targeting the Master's Private IP:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_PRIVATE_IP>:6443 \
  K3S_TOKEN=<NODE_TOKEN> \
  sh -
```

**Verification:** Run `sudo systemctl status k3s-agent` on each worker instance to confirm **active (running)**.

---

### 1.4 Verify Cluster Status

Return to the Master EC2 instance and list all connected nodes:

```bash
sudo kubectl get nodes -o wide
```

**Verification:** All 3 nodes appear in the list with STATUS: **Ready** using their internal `172.31.x.x` IPs.

---

## Notes

- All instances must be in the same VPC and subnet (or allow cross‑subnet traffic via the shared Security Group).
- The Security Group rules enable private communication without exposing nodes to the internet (except for SSH and optional NodePort access from your IP).
- After the cluster is up, you can deploy workloads using `kubectl` from the master or any machine with `kubeconfig` access.