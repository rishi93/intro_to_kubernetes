# Kubernetes

A Kubernetes cluster, is a set of nodes working together. You can deploy
containerized applications on this cluster.

There are worker nodes, and a control plane (also consisting of nodes) that
control these worker nodes.

The nodes in the control plane can be called master nodes.

Ideally, you should have 3 nodes in the control plane, to have high tolerance
against failures.

For learning purposes, and to keep the costs low, we will run one master node
in the control plane, and one worker node.

## Setting up your own Kubernetes cluster

* Hetzner cloud has cheap VMs available. Get two VMs there, 1 master
node and 1 worker node.

* We'll setup the cluster with `kubeadm`. We also have to configure the
Container Runtime. 

* `containerd` is the Container Runtime, we'll be using.

### iptables and bridged traffic

We'll use iptables in our setup, and we need to ensure that it can see
bridged traffic.

Add this to the `/etc/modules-load.d/k8s.conf` file:
```
overlay
br_netfilter
EOF
```

Add this to the `/etc/sysctl.d/k8s.conf` file:
```
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the sysctl params without reboot
```
sudo sysctl --system
```

### cgroup drivers
We will configure a `control group` driver so that the `kubelet` and the 
container runtime can interact with the underlying resources.

Although the `cgroupfs driver` is the default in the kubelet, we'll choose the
`systemd cgroup driver` as this is the recommended driver when running on a
systemd init system.

### Install `containerd`
See installation instructions here: https://containerd.io/

Extract it to `/usr/local`

Before configuring the `systemd` service, we'll generate the config file
for `containerd`
```
sudo mkdir /etc/containerd

containerd config default > config.toml

sudo cp config.toml /etc/containerd
```

Now download the service manifest for containerd and enable the service.
```
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

sudo cp containerd.service /etc/systemd/system/

sudo systemctl daemon-reload

sudo systemctl enable --now containerd
```

### Install `runc`
There is a different component that is the actual runtime. This is called `runc`.

This is the website: https://github.com/opencontainers/runc

```
wget https://github.com/opencontainers/runc/releases/download/v1.1.10/runc.amd64

sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Install cni plugin for containerd
containerd needs a cni plugin for interacting with the network.

Links:

https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/

https://github.com/containernetworking/cni

Installation instructions
```
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz

sudo mkdir -p /opt/cni/bin

sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.4.0.tgz
```

### Configure systemd cgroup driver
We'll also need to configure the systemd cgroup driver for `containerd` which
is done in the `/etc/containerd/config.toml` file. We'll change the
`SystemdCgroup` parameter to `true`.

After updating the configuration, restart the `containerd` service and make
sure it's working.
```
sudo systemctl restart containerd
sudo systemctl status containerd
```

## Install Kubernetes packages
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

```
# Add the public signing key for the Kubernetes package repo
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes `apt` repository
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

## Disable swap
Previously disabling `swap` was a requirement for `kubelet` to run. From v1.22,
`swap` has been supported, but with some extra configuration needed for kubelet.
We've chosen to disable swap.

To temporarily disable swap,
```
sudo swapoff -a
```

To make it persistent however, we need to edit the `/etc/fstab` file:
```
# Comment out the swap line
```

## Kubernetes network model
From Calico website: https://www.tigera.io/learn/guides/kubernetes-networking/ 

The kubernetes network model specifies:
* Each pod gets its own IP address
* Containers within a pod share the pod IP address, and communicate freely
with each other
* Pods can communicate with all other pods in the cluster using pod IP address
(without NAT)
* Isolation (restricting what each pod can communicate with) is defined using
network policies.

As a result, pods can be treated much like VMs or hosts (they all have unique
IP addresses), and the containers within pods can be treated like processes
running within a VM or host (they run in the same network namespace and share
an IP address).

Kubernetes built-in network support `kubelet` can provide some basic network
connectivity. However, it is more common to use third-party network implementations
that plug into Kubernetes using the CNI (Container Network Interface) API.

There are lots of different kinds of CNI plugins, but the two main ones are:
* Network plugins, which are responsible for connecting pods to the network
* IPAM (IP Address Management) plugins, which are responsible for allocating
pod IP addresses.

Calico provides both network and IPAM plugins.

### Kubernetes Services
Kubernetes services provide a way of abstracting access to a group of pods as a
network service. The group of pods is usually defined using a label selector.

Within the cluster, the network service is usually represented as a virtual IP
address, and kube-proxy load balances connections to the virtual IP address
across the group of pods backing the service.

The virtual IP is discovereable through Kubernetes DNS. The DNS name and virtual
IP address remain constant for the lifetime of the service, even though the pods
backing the service may be created or destroyed, and the number of pods backing
the service may change over time.

Kubernetes service, can also define how a service is accessed from outside of
the cluster, using one of the following:
* A node port, where the service can be accessed via a specific port on every node
* A load balancer, where a network load balancer provides a virtual IP address
that the service can be accessed via from outside the cluster.

### Kubernetes DNS
Each Kubernetes cluster provides a DNS service. Each pod and every service is
discoverable through the Kubernetes DNS service.

