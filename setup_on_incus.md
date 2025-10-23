
# Setup
**Clone The Repository:**
```shell
git clone https://github.com/chan3845/lxc-k8s-deploy.git
cd lxc-k8s-deploy/
```

**Create and edit a new custom profile for Kubernetes cluster nodes.**

Use the `lxc-profile-k8s` file as a template and adjust it according to your requirements. <br>
**Important:** Ensure that the bridge adapter name in the parent configuration and the storage pool name match your environment.


```shell
incus profile create k8s
incus profile edit k8s < lxc-profile-k8s
```

**Create Incus containers:**
```shell
incus launch images:ubuntu/noble srv-k8s-m1 --profile k8s 
incus launch images:ubuntu/noble srv-k8s-w1 --profile k8s 
incus launch images:ubuntu/noble srv-k8s-w2 --profile k8s 
incus launch images:ubuntu/noble srv-k8s-w3 --profile k8s 
```

> [!IMPORTANT]
> **Naming Convention:** 
> Master node must include the word `master` in its name and worker nodes must include `worker` to avoid issues with the bootstrap script.

**Verify containers are running**
```shell
incus ls
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
incus config device add srv-k8s-m1 eth0 nic   nictype=bridged   parent=incusbr0   name=eth0   ipv4.address=192.168.201.50
incus config device add srv-k8s-w1 eth0 nic   nictype=bridged   parent=incusbr0   name=eth0   ipv4.address=192.168.201.51
incus config device add srv-k8s-w2 eth0 nic   nictype=bridged   parent=incusbr0   name=eth0   ipv4.address=192.168.201.52
incus config device add srv-k8s-w3 eth0 nic   nictype=bridged   parent=incusbr0   name=eth0   ipv4.address=192.168.201.53
```
<br>

Restart containers to apply IP changes.
```shell
lincus restart srv-k8s-m1 srv-k8s-w1 srv-k8s-w2 srv-k8s-w3
```

Now verify the IP again.
```shell
incus ls
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
cat bootstrap-k8s.sh | incus exec srv-k8s-m1 bash
```

After the bootstrap successfully completed on the master node, run the `bootstrap-k8s.sh` script for each worker node as well.
```
cat bootstrap-k8s.sh | incus exec srv-k8s-w1 bash
cat bootstrap-k8s.sh | incus exec srv-k8s-w2 bash
cat bootstrap-k8s.sh | incus exec srv-k8s-w3 bash
```

### Verify the Kubernetes Cluster
 Login to the master node
```shell
incus exec srv-k8s-m1 bash 
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
