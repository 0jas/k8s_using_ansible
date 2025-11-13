# HA Kubernetes Cluster on AlmaLinux 8.x

This project provides a set of Ansible roles to automate the deployment of a production-ready, high-availability (HA) Kubernetes cluster on AlmaLinux 8.x.

The playbook uses `kubeadm` to bootstrap the cluster, `containerd` as the container runtime, the standard `kube-proxy` for service proxying, and **Cilium** as the CNI for networking. It also configures an external **HAProxy** load balancer for the control plane API endpoint.

## 📚 Table of Contents

  * [Features](https://www.google.com/search?q=%23features)
  * [Cluster Architecture](https://www.google.com/search?q=%23cluster-architecture)
  * [Prerequisites](https://www.google.com/search?q=%23prerequisites)
      * [Control Node](https://www.google.com/search?q=%23control-node)
      * [Target Nodes](https://www.google.com/search?q=%23target-nodes)
  * [Configuration](https://www.google.com/search?q=%23configuration)
      * [1. Inventory](https://www.google.com/search?q=%231-inventory)
      * [2. Ansible Configuration](https://www.google.com/search?q=%232-ansible-configuration)
      * [3. Cluster Variables](https://www.google.com/search?q=%233-cluster-variables)
  * [Usage](https://www.google.com/search?q=%23usage)
      * [Usage with Tags](https://www.google.com/search?q=%23usage-with-tags)
  * [Playbook Workflow](https://www.google.com/search?q=%23playbook-workflow)
  * [Post-Deployment Verification](https://www.google.com/search?q=%23post-deployment-verification)

## ✨ Features

  * **High-Availability:** Deploys a multi-master (3-node) control plane.
  * **External Load Balancer:** Automatically configures HAProxy for a stable control plane endpoint.
  * **Modern CNI:** Installs **Cilium** for high-performance networking.
  * **Standard Service Proxy:** Uses the standard, built-in **`kube-proxy`** for cluster services.
  * **Modern Runtime:** Uses `containerd` as the container runtime (CRI).
  * **Automated Prerequisites:** Handles all node-level setup:
      * Disables SELinux
      * Disables swap
      * Configures `firewalld` with all necessary rules for Kubernetes, `kube-proxy`, and Cilium
      * Loads required kernel modules (`overlay`, `br_netfilter`)
      * Sets required `sysctl` parameters
      * Sets timezone and syncs time with `chrony`
  * **Dynamic Hostfile:** Generates and distributes an `/etc/hosts` file to all nodes for easy intra-cluster communication.
  * **Idempotent:** Playbook can be re-run safely.

## 🏗️ Cluster Architecture

This playbook provisions the following architecture:

  * **1 x Load Balancer Node:**
      * Runs HAProxy to balance traffic for the K8s API server (port 6443) across all master nodes.
  * **3 x Control Plane (Master) Nodes:**
      * Run the Kubernetes API server, `etcd`, scheduler, and controller manager.
  * **2 x Worker Nodes:**
      * Run `kubelet` and workloads (pods).

All nodes in the cluster (masters and workers) will have `containerd`, `kubelet`, `kube-proxy`, `socat`, and `iproute-tc` installed.

## ⚠️ Prerequisites

Before running this playbook, ensure your environment meets the following requirements.

### Control Node

  * **Ansible:** Ansible 2.12+ installed.
  * **SSH Keys:** An SSH keypair (e.g., `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`) used to access the target nodes.

### Target Nodes

  * **OS:** A fresh, minimal installation of **AlmaLinux 8.x** on all servers (masters, workers, and load balancer).
  * **Network:** All nodes must have static IP addresses and be able to communicate with each other. The IPs must match what you define in the inventory.
  * **SSH Access:** An SSH user (e.g., `ansible_user`) must exist on **all** target nodes.
  * **Sudo Privileges:** The `ansible_user` **must have passwordless `sudo` privileges** on all target nodes. This is critical for Ansible's `become: yes` to function.

> **How to set up the SSH user:**
>
> 1.  On each target node, create the user:
>
>     ```bash
>     sudo useradd ansible_user
>     sudo mkdir -p /home/ansible_user/.ssh
>     sudo chmod 700 /home/ansible_user/.ssh
>     ```
>
> 2.  Copy your **public key** from the control node to each target node's `authorized_keys` file:
>
>     ```bash
>     sudo-edit /home/ansible_user/.ssh/authorized_keys
>     # Paste your ~/.ssh/id_rsa.pub content here
>     ```
>
> 3.  Set correct permissions:
>
>     ```bash
>     sudo chown -R ansible_user:ansible_user /home/ansible_user/.ssh
>     sudo chmod 600 /home/ansible_user/.ssh/authorized_keys
>     ```
>
> 4.  Add passwordless sudo:
>
>     ```bash
>     sudo-edit /etc/sudoers.d/ansible
>     # Add the following line:
>     ansible_user ALL=(ALL) NOPASSWD: ALL
>     ```

## ⚙️ Configuration

There are three main files to configure:

### 1\. Inventory

`inventories/production/hosts.ini`

Define your hosts, their IP addresses, and assign them to the correct groups. The playbook logic is built on these groups.

```ini
[masters]
master1 ansible_host=192.168.157.100
master2 ansible_host=192.168.157.101
master3 ansible_host=192.168.157.102

[workers]
worker1 ansible_host=192.168.157.200
worker2 ansible_host=192.168.157.201

[loadbalancer]
lb1 ansible_host=192.168.157.50

[kube_cluster:children]
masters
workers

[all:children]
kube_cluster
loadbalancer
```

### 2\. Ansible Configuration

`ansible.cfg`

Update the `remote_user` and `private_key_file` to match your environment.

```ini
[defaults]
inventory = inventories/production/hosts.ini
remote_user = ansible_user  # <-- !! Change this
private_key_file = ~/.ssh/id_rsa # <-- !! Change this
host_key_checking = False
retry_files_enabled = False
```

### 3\. Cluster Variables

`inventories/production/group_vars/all.yaml`

These variables control the Kubernetes and Cilium deployment.

```yaml
---
# System configuration
system_timezone: "Australia/Melbourne"

# Kubernetes configuration
k8s_version: "1.29.0"        # K8s version to install
pod_network_cidr: "10.244.0.0/16"
service_network_cidr: "10.96.0.0/12"

# HA Proxy Endpoint (Must match your loadbalancer IP)
cluster_endpoint_ip: "192.168.157.50"
cluster_endpoint_port: 6443

# Cilium configuration
cilium_cli_version: "v0.18.8"
cilium_helm_version: "1.18.3"
```

## ▶️ Usage

Once all prerequisites and configuration steps are complete, run the playbook from the root of the project directory.

```bash
# Navigate to the project directory
cd k8s_using_ansible/

# 1. (Recommended) Run a syntax check
ansible-playbook playbook.yaml --check

# 2. Execute the full playbook
ansible-playbook playbook.yaml
```

If your `sudo` user requires a password (not recommended), you will need to add the `--ask-become-pass` flag:

```bash
ansible-playbook playbook.yaml --ask-become-pass
```

The playbook will take several minutes to run, as it includes reboots (for SELinux) and downloading container images.

### ▶️ Usage with Tags

This playbook includes tags for every task, allowing for granular control over what runs. This is useful for debugging, re-running failed steps, or skipping time-consuming tasks.

A tag consists of the role name and the task (e.g., `common-hostname`, `cilium-install`). Each role also has a "group" tag (e.g., `common`, `cilium`).

#### To Run *Only* Specific Tasks:

Use the `--tags` (or `-t`) flag.

```bash
# Example 1: Run only the task that sets the system timezone
ansible-playbook playbook.yaml --tags "common-timezone"

# Example 2: Run all tasks related to the 'common' role
ansible-playbook playbook.yaml --tags "common"

# Example 3: Run the Cilium installation AND the swap configuration
ansible-playbook playbook.yaml --tags "cilium-install,common-swap"
```

#### To *Exclude* Specific Tasks:

Use the `--skip-tags` flag.

```bash
# Example 1: Run the entire 'common' role, but skip the common-sysctl
ansible-playbook playbook.yaml --tags "common" --skip-tags "common-sysctl"

# Example 2: Run the whole playbook, but skip all Cilium-related tasks
ansible-playbook playbook.yaml --skip-tags "cilium"

# Example 3: Run the whole playbook, but skip the firewall configuration
ansible-playbook playbook.yaml --skip-tags "firewall"
```

## 📋 Playbook Workflow

The main `playbook.yaml` executes a series of roles in a specific order to build the cluster:

1.  **`01-common`**

      * Runs on: **All nodes**
      * Sets hostname, generates `/etc/hosts` file, sets timezone, and syncs time.
      * Disables SELinux (and reboots if changed).
      * Disables swap (`swapoff -a` and removes from `fstab`).
      * Loads and persists kernel modules (`overlay`, `br_netfilter`).
      * Applies kernel `sysctl` settings for Kubernetes.

2.  **`02-firewall`**

      * Runs on: **All nodes**
      * Configures `firewalld` with the necessary ports for master, worker, `kube-proxy`, and load balancer nodes, including rules for Cilium's VXLAN/WireGuard.

3.  **`03-loadbalancer`**

      * Runs on: **Load balancer node**
      * Installs and configures **HAProxy** to forward traffic from `192.168.157.50:6443` to all master nodes.

4.  **`04-containerd`**

      * Runs on: **Cluster nodes (masters, workers)**
      * Installs `containerd.io` from the Docker CE repository.
      * Configures `containerd` to use the `systemd` cgroup driver, which is required by `kubelet`.

5.  **`05-kube-prereqs`**

      * Runs on: **Cluster nodes (masters, workers)**
      * Adds the official Kubernetes YUM repository.
      * Installs `kubelet`, `kubeadm`, and `kubectl`.
      * Installs required helper tools: `socat` (for port-forwarding) and `iproute-tc` (for Cilium).
      * Enables the `kubelet` service.

6.  **`06-kube-master`**

      * Runs on: **Master nodes**
      * Generates a `kubeadm-config.yaml` on `master1` (configured to use `kube-proxy`).
      * Initializes the cluster on `master1`.
      * Copies the `admin.conf` to the user's home directory on `master1`.
      * Generates and stores join tokens.
      * Joins `master2` and `master3` to the control plane.

7.  **`06a-Clean up kube-proxy`**

      * This play is **skipped by default** (since `kube_proxy_skip: false`). It is only used for `kube-proxy` replacement scenarios.

8.  **`07-cilium`**

      * Runs on: **`master1`**
      * Downloads the Cilium CLI.
      * Uses the Cilium CLI to install the CNI onto the cluster to run *alongside* `kube-proxy`.

9.  **`08-kube-worker`**

      * Runs on: **Worker nodes**
      * Uses the worker join token (generated in role `06-kube-master`) to join the cluster.

## ✅ Post-Deployment Verification

After the playbook finishes, you can verify the cluster health.

1.  SSH into `master1`:

    ```bash
    ssh ansible_user@192.168.157.100
    ```

2.  Check the node status. All nodes should be in the `Ready` state. (This may take a minute or two after the playbook finishes as the Cilium pods roll out).

    ```bash
    $ kubectl get nodes -o wide

    NAME      STATUS   ROLES           AGE     VERSION   INTERNAL-IP        EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION   CONTAINER-RUNTIME
    master1   Ready    control-plane   5m10s   v1.29.0   192.168.157.100   <none>        AlmaLinux 8.x    4.18.0...        containerd://1.7.1
    master2   Ready    control-plane   4m15s   v1.29.0   192.168.157.101   <none>        AlmaLinux 8.x    4.18.0...        containerd://1.7.1
    master3   Ready    control-plane   3m20s   v1.29.0   192.168.157.102   <none>        AlmaLinux 8.x    4.18.0...        containerd://1.7.1
    worker1   Ready    <none>          2m5s    v1.29.0   192.168.157.200   <none>        AlmaLinux 8.x    4.18.0...        containerd://1.7.1
    worker2   Ready    <none>          1m45s   v1.29.0   192.168.157.201   <none>        AlmaLinux 8.x    4.18.0...        containerd://1.7.1
    ```

3.  Check the `kube-system` pods. You should see `cilium` pods running on every node, **as well as `kube-proxy` pods**.

    ```bash
    $ kubectl get pods -n kube-system -o wide

    NAME                                READY   STATUS    RESTARTS   AGE     IP                 NODE      NOMINATED NODE   READINESS GATES
    cilium-operator-...                 1/1     Running   0          4m42s   192.168.157.101   master2   <none>           <none>
    cilium-operator-...                 1/1     Running   0          4m42s   192.168.157.102   master3   <none>           <none>
    cilium-qjgrz                          1/1     Running   0          3m20s   192.168.157.102   master3   <none>           <none>
    cilium-r7v6n                          1/1     Running   0          3m20s   192.168.157.101   master2   <none>           <none>
    cilium-r9vj4                          1/1     Running   0          2m5s    192.168.157.200   worker1   <none>           <none>
    cilium-tfd42                          1/1     Running   0          1m45s   192.168.157.201   worker2   <none>           <none>
    cilium-x82m8                          1/1     Running   0          3m20s   192.168.157.100   master1   <none>           <none>
    coredns-76f75df...                  1/1     Running   0          5m4s    10.244.0.4         master1   <none>           <none>
    coredns-76f75df...                  1/1     Running   0          5m4s    10.244.0.3         master1   <none>           <none>
    etcd-master1                        1/1     Running   0          5m16s   192.168.157.100   master1   <none>           <none>
    etcd-master2                        1/1     Running   0          4m20s   192.168.157.101   master2   <none>           <none>
    etcd-master3                        1/1     Running   0          3m25s   192.168.157.102   master3   <none>           <none>
    kube-apiserver-master1              1/1     Running   0          5m16s   192.168.157.100   master1   <none>           <none>
    ... (kube-controller-manager, kube-scheduler) ...
    kube-proxy-8x45f                      1/1     Running   0          2m5s    192.168.157.200   worker1   <none>           <none>
    kube-proxy-k8qtl                      1/1     Running   0          4m20s   192.168.157.101   master2   <none>           <none>
    kube-proxy-m4b8g                      1/1     Running   0          5m16s   192.168.157.100   master1   <none>           <none>
    kube-proxy-p5gdb                      1/1     Running   0          1m45s   192.168.157.201   worker2   <none>           <none>
    kube-proxy-z2k4p                      1/1     Running   0          3m25s   192.168.157.102   master3   <none>           <none>
    ```

Your cluster is now ready.