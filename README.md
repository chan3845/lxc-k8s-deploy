## Kubernetes Cluster Setup with LXD Containers
This repository contains scripts to quickly set up a Kubernetes (K8s) cluster using **LXD containers**. It is intended for **learning, testing, and demo purposes**.

> ⚠️ **Note:** Since this setup uses LXD containers, it is **not recommended for production use**. LXD is excellent for lightweight, fast, and isolated environments, but it lacks the robustness and security guarantees of full VM or bare-metal deployments.


This project is heavily inspired by and modified from the original work by [justmeandopensource](https://github.com/justmeandopensource/kubernetes.git)
. While this repository is not a direct fork, full credit goes to their project for providing the foundational ideas and structure.

<br>


You can check info about LXD here: https://documentation.ubuntu.com/lxd/stable-5.21/tutorial/first_steps/
<br>


## Why install Kubernetes on LXD?

Even though tools like **Minikube** or **VM-based clusters** exist, running K8s on LXD has some advantages:

- **Faster spin-up time** than full VMs.
- **Lower resource overhead** while still providing a multi-node cluster experience.
- **Great for learning** real K8s networking, multi-node communication, and cluster operations without needing multiple VMs.
    
This makes it ideal for **labs, CI/CD testing, and demos**.

<br>

---

# Setup
Clone The Repository
```shell
git clone https://github.com/chan3845/lxc-k8s-deploy.git
cd lxc-k8s-deploy/
```

Create and edit a new custom profile for Kubernetes cluster nodes
```shell
lxc profile create k8s
lxc profile edit k8s
```

In the editor, paste the following contents from the `lxc-profile-k8s` file
```shell
config:
  limits.cpu: "2"
  limits.memory: 4GB
  limits.memory.swap: "false"
  linux.kernel_modules: ip_tables,ip6_tables,nf_nat,overlay,br_netfilter
  raw.lxc: "lxc.apparmor.profile=unconfined\nlxc.cap.drop= \nlxc.cgroup.devices.allow=a\nlxc.mount.auto=proc:rw
    sys:rw"
  security.privileged: "true"
  security.nesting: "true"
description: LXD profile for Kubernetes
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: lxdbr0
    type: nic
  kmsg:
    path: /dev/kmsg
    source: /dev/kmsg
    type: unix-char
  root:
    path: /
    pool: default
    type: disk
name: k8s
used_by: []
```

Create LXC containers
```shell
lxc launch ubuntu:24.04 srv-k8s-m1 --profile k8s
lxc launch ubuntu:24.04 srv-k8s-w1 --profile k8s
lxc launch ubuntu:24.04 srv-k8s-w2 --profile k8s
lxc launch ubuntu:24.04 srv-k8s-w3 --profile k8s
```

> [!IMPORTANT]
> **Naming Convention:** 
> Master node must include the word `master` in its name and worker nodes must include `worker` to avoid issues with the bootstrap script.

**Verify containers are running**
```shell
lxc ls
```

**Sample Output:**
```shell
+------------+---------+------------------------+----------------------------------------------+-----------+-----------+
|    NAME    |  STATE  |          IPV4          |                     IPV6                     |   TYPE    | SNAPSHOTS |
+------------+---------+------------------------+----------------------------------------------+-----------+-----------+
| srv-k8s-m1 | RUNNING | 192.168.201.104 (eth0) | fd42:977e:cc5:b133:216:3eff:fe08:310b (eth0) | CONTAINER | 0         |
+------------+---------+------------------------+----------------------------------------------+-----------+-----------+
| srv-k8s-w1 | RUNNING | 192.168.201.61 (eth0)  | fd42:977e:cc5:b133:216:3eff:fe87:2e24 (eth0) | CONTAINER | 0         |
+------------+---------+------------------------+----------------------------------------------+-----------+-----------+
| srv-k8s-w2 | RUNNING | 192.168.201.107 (eth0) | fd42:977e:cc5:b133:216:3eff:fe52:2f0c (eth0) | CONTAINER | 0         |
+------------+---------+------------------------+----------------------------------------------+-----------+-----------+
| srv-k8s-w3 | RUNNING | 192.168.201.191 (eth0) | fd42:977e:cc5:b133:216:3eff:fea0:2eb (eth0)  | CONTAINER | 0         |
+------------+---------+------------------------+----------------------------------------------+-----------+-----------+

```
<br>

### Assign Static IP (optional but recommended)

Assign a static IP for each node instead of using dynamically assigned IP from LXD network.

```shell
lxc config device add srv-k8s-m1 eth0 nic   nictype=bridged   parent=lxdbr0   name=eth0   ipv4.address=192.168.201.50
lxc config device add srv-k8s-w1 eth0 nic   nictype=bridged   parent=lxdbr0   name=eth0   ipv4.address=192.168.201.51
lxc config device add srv-k8s-w2 eth0 nic   nictype=bridged   parent=lxdbr0   name=eth0   ipv4.address=192.168.201.52
lxc config device add srv-k8s-w3 eth0 nic   nictype=bridged   parent=lxdbr0   name=eth0   ipv4.address=192.168.201.53
```
<br>
Restart containers to apply IP changes.
```shell
lxc restart srv-k8s-m1 srv-k8s-w1 srv-k8s-w2 srv-k8s-w3
```

Now verify the IP again.
```shell
lxc ls
```
Output:
```shell
+------------+---------+-----------------------+----------------------------------------------+-----------+-----------+
|    NAME    |  STATE  |         IPV4          |                     IPV6                     |   TYPE    | SNAPSHOTS |
+------------+---------+-----------------------+----------------------------------------------+-----------+-----------+
| srv-k8s-m1 | RUNNING | 192.168.201.50 (eth0) | fd42:977e:cc5:b133:216:3eff:fe08:310b (eth0) | CONTAINER | 0         |
+------------+---------+-----------------------+----------------------------------------------+-----------+-----------+
| srv-k8s-w1 | RUNNING | 192.168.201.51 (eth0) | fd42:977e:cc5:b133:216:3eff:fe87:2e24 (eth0) | CONTAINER | 0         |
+------------+---------+-----------------------+----------------------------------------------+-----------+-----------+
| srv-k8s-w2 | RUNNING | 192.168.201.52 (eth0) | fd42:977e:cc5:b133:216:3eff:fe52:2f0c (eth0) | CONTAINER | 0         |
+------------+---------+-----------------------+----------------------------------------------+-----------+-----------+
| srv-k8s-w3 | RUNNING | 192.168.201.53 (eth0) | fd42:977e:cc5:b133:216:3eff:fea0:2eb (eth0)  | CONTAINER | 0         |
+------------+---------+-----------------------+----------------------------------------------+-----------+-----------+ 
```
<br>

To avoid issues with `kube-proxy` pods:
```shell
sudo sysctl -w net.netfilter.nf_conntrack_max=131072
```

Verify
```shell
sysctl net.netfilter.nf_conntrack_max
#net.netfilter.nf_conntrack_max = 131072
```

<br>

### Run the Bootstrap Script
Now run the `bootstrap-k8s.sh` for the master node `srv-k8s-m1` and wait till it is finished.
```shell
cat bootstrap-kube.sh | lxc exec srv-k8s-m1 bash
```

After the bootstrap successfully completed on the master node, run the `bootstrap-k8s.sh` script for each worker node as well.
```
cat bootstrap-kube.sh | lxc exec srv-k8s-w1 bash
cat bootstrap-kube.sh | lxc exec srv-k8s-w2 bash
cat bootstrap-kube.sh | lxc exec srv-k8s-w3 bash
```

### Verify the Kubernetes Cluster
 Login to the master node
```shell
lxc exec srv-k8s-m1 bash 
```

Check the cluster nodes status
```shell
kubectl get nodes
NAME         STATUS   ROLES           AGE     VERSION
srv-k8s-m1   Ready    control-plane   6m39s   v1.34.1
srv-k8s-w1   Ready    <none>          4m9s    v1.34.1
srv-k8s-w2   Ready    <none>          2m51s   v1.34.1
srv-k8s-w3   Ready    <none>          77s     v1.34.1
```
