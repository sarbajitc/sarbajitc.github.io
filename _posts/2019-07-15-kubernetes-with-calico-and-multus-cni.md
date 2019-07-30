---
layout: post
title:  "Kubernetes setup with Calico and Multus Macvlan CNI plugins in VirtualBox"
date:   2019-07-15 14:40:00
categories: kubernetes calico multus macvlan networking
---

Here I'll list down steps to setup a 2 node kubernetes cluster in couple of VirtualBox VMs with calico and Macvlan as CNI networking plugins (using Multus).

### Setup
* VirtualBox
* 2 VMs with Ubuntu 16.04
* Each VM has 2 CPU, 4GB RAM, 40GB disk
* Each VM has 3 network interfaces. First a NAT adatper, second and third are two different host-only network adapters (different subnet)
* Both host-only networks are DHCP enabled. Ubuntu VMs have required DHCP settings in `/etc/network/interfaces` file.
* For Macvlan IPs assigned to containers to be pingable, we need to set Promiscous mode to **Allow all** in third adapter (host-only)
* Set promisc mode for 2nd host-only adapter in each VM `sudo ip link set enp0s9 promisc on` for macvlan to work

### Pre-requisites to install kubernetes cluster
* Install latest package updates `apt-get update; apt-get upgrade`
* Set unique hostname to every VM by editing `/etc/hostname` file. e.g. kube-1, kube-2
* Update `/etc/hosts` file with hostnames of both VMs resolving to their 1st host-only adapter IP
* Remove 127.0.0.1 IP resolution to unique hostname from `/etc/hosts` file. 127.0.0.1 resolving to localhost - is ok.
* Turn off swap partition `sudo swapoff -a` and comment out the swap partition line from `/etc/fstab` file
* Install docker engine `apt-get install docker.io`
* Add your user to docker group e.g. `usermod -a -G docker sarbajit`

### Install kubernetes
Follow below steps ([using kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)) -

```bash
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

### Configure master node
* Initialize kubernetes master `kubeadm init --apiserver-advertise-address=192.168.56.112 --pod-network-cidr=192.168.0.0/16` 
  here **apiserver-advertise-address** is the IP of first host-only adapter and **pod-network-cidr** is a free subnet for calico to use
* Once the command is successful, note down the `kubeadm join` command from the output for future use
* Setup kubeconfig to run kubectl
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
* Enable container creation in master node `kubectl taint nodes --all node-role.kubernetes.io/master-`

### Configure worker nodes
* Run the `kubeadm join` command that you noted down before `kubeadm join 192.168.56.112:6443 --token rp2epw.i3omz7ionz94r7ya --discovery-token-ca-cert-hash sha256:bc737f0afc605bff5f7aed2310d6afadc835cf3e848d62b6cfc5f4a536e9bc26`
* Check in master if both nodes are listed `kubectl get nodes` and in Ready state

### Install primary CNI plugin (calico)
* On master node execute `kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml` which will setup [calico 3.8](https://docs.projectcalico.org/v3.8/getting-started/kubernetes/)
* Verify calico pods are in running state using command `kubectl get pods -o wide --all-namespaces`
* Optional: setup `calicoctl` tool for debugging calico
  * Downlload calicoctl binary `curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.8.1/calicoctl`
  * Set executable permission `chmod +x calicoctl`

### Install Multus CNI plugin
* On master node execute `kubectl apply -f https://raw.githubusercontent.com/intel/multus-cni/master/images/multus-daemonset.yml` which will setup [Multus](https://github.com/intel/multus-cni/). Multus takes over as main CNI plugin and uses calico as default network plugin.
* Verify multus pods are in running state using command `kubectl get pods -o wide --all-namespaces`

### Create Network Attacment definition for Multus
* We will add a Macvlan interface profile with multus (so that pods get two interfaces - one from calico and another from macvlan)
* Execute following command in master
```bash
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf-1
spec:
  config: '{
            "cniVersion": "0.3.0",
            "type": "macvlan",
            "master": "enp0s9",
            "mode": "bridge",
            "ipam": {
                "type": "host-local",
                "ranges": [
                    [ {
                         "subnet": "192.168.48.0/24",
                         "rangeStart": "192.168.48.20",
                         "rangeEnd": "192.168.48.50",
                         "gateway": "192.168.48.1"
                    } ]
                ]
            }
        }'
EOF
```
here the subnet range is matching the VirtualBox 2nd host-only adapter subnet.

### Launch a pod with multus macvlam profile
* Execute following command in master -

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-case-01
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf-1
spec:
  containers:
  - name: pod-case-01
    image: docker.io/centos/tools:latest
    command:
    - /sbin/init
EOF
```
* Verify the pod interfaces with `kubectl exec -it -n default pod-case-01 -- ip a`
