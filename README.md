# kubernetes-install

This repository contains step by step instructions for setting up Kubernetes cluster using `containerd` as container runtime.

## CoreDNS

`systemd` interferes with `resolv.conf`. This causes CoreDNS going into a loop, querying itself for every request, since 127.0.0.53 is a loopback IP. So we have to use an alternate resolv.conf file. Luckily, systemd ships with an alternate resolv.conf file that actually contains a list of valid nameservers, and doesn't reference localhost. This file is very useful for scenarios like these where you do not want to use systemd's DNS server.

As root, edit /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

```
Environment="KUBELET_EXTRA_ARGS=--resolv-conf=/run/systemd/resolve/resolv.conf"
```

Then execute:

```
systemctl daemon-reload && service kubelet restart
```

Lastly, redeploy your CoreDNS pods and test Kubernetes DNS, everything should be working correctly.

## Metal LB

MetalLB is a load-balancer implementation for bare metal Kubernetes clusters, using standard routing protocols.

Kubernetes does not offer an implementation of network load-balancers (Services of type LoadBalancer) for bare metal clusters. The implementations of Network LB that Kubernetes does ship with are all glue code that calls out to various IaaS platforms (GCP, AWS, Azure…). If you’re not running on a supported IaaS platform (GCP, AWS, Azure…), LoadBalancers will remain in the “pending” state indefinitely when created.

MetalLB aims to redress this imbalance by offering a Network LB implementation that integrates with standard network equipment, so that external services on bare metal clusters also “just work” as much as possible.

### Requirements

- A Kubernetes cluster, running Kubernetes 1.13.0 or later, that does not already have network load-balancing functionality.
- A cluster network configuration that can coexist with MetalLB.
- Traffic on port 7946 (TCP & UDP) must be allowed between nodes, as required by memberlist.

### Installation

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/namespace.yaml

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/metallb.yaml

# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

### Layer 2 Configuration

Layer 2 mode is the simplest to configure: in many cases, you don’t need any protocol-specific configuration, only IP addresses.

Layer 2 mode does not require the IPs to be bound to the network interfaces of your worker nodes. It works by responding to ARP requests on your local network directly, to give the machine’s MAC address to clients.

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
```
