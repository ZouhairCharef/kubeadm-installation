# k8s-installation-kubeadm

## Pre-requisite

* Master node - 2 cpu x 2 GB memory
* Worker node - 1 cpu x 2 GB memory

## Pre-installation steps

### The below steps will be performed on both master and worker node

*  1. Turn of Swap
	` apt-get update`
	` swapoff -a `

*  2. Comment swap FS from /etc/fstab
	` vi /etc/fstab`
	` Comment any line that has swap written`

*  3. Edit /etc/hostname and edit hostname to match the host of your choice

*  4. Get private ip address of all hosts
	` ip addr `

*  5. Edit /etc/hosts to add hostname and IP address on all nodes
	` vi /etc/hosts `
	`kmaster 192.168.0.2 -- private IP address from previous step`
	`knode1 192.168.0.3 -- provate IP address from previous step`

### Forwarding IPv4 and letting iptables see bridged traffic
* Execute the below mentioned instructions:
```
{
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
}
```

* Verify that the br_netfilter, overlay modules are loaded by running the following commands:
```
{
lsmod | grep br_netfilter
lsmod | grep overlay
}
```

* Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
```
{
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
}
```

### Installing containerd
* To install containerd on your system, follow the instructions
```
{
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install containerd.io
}
```

### Cgroup drivers
* To verify which init system you have
`ps -p 1`

* To use the `systemd` cgroup driver in `/etc/containerd/config.toml` with `runc`, set
```
#Delete all the default configuration, and add this configuration
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

* After that restart the containerd service
`sudo systemctl restart containerd`


## Installation Procedure

### Install kubelet, kubeadm and kubectl

*  **Ubuntu**

```
{
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
}
```

### Configure Cluster using kubeadm

**The below steps will be performed**  __**ONLY ON MASTER NODE**__

* Get the IP address of master

```
ip addr
```

* Initialize the cluster

```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address={{IP_ADDR_MASTER}}
```

* Your output should look like below -

```
#To start using your cluster, you need to run the following as a regular user:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#You should now deploy a pod network to the cluster.
#Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
https://kubernetes.io/docs/concepts/cluster-administration/addons/

#Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 10.138.0.2:6443 --token 3ccgnq.2owa1scoiqqoqhdq \
--discovery-token-ca-cert-hash sha256:04ff7a9148ae02db8a4be70f016fe66d6d5870ea09641e48f6a7db54747e1acd
```

Preserve the above output as it contains the token required for node configuration.

* Copy over the configuration files

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* Install Networking component (CNI)

```
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
```

* Join the worker node

The below steps are to be run __**only on the worker node**__

The output of the kubeadm init command will provide the kubeadm join as its output. Run the kubeadm join command on the worker nodes.

```
kubeadm join 10.138.0.2:6443 --token 3ccgnq.2owa1scoiqqoqhdq \
--discovery-token-ca-cert-hash sha256:04ff7a9148ae02db8a4be70f016fe66d6d5870ea09641e48f6a7db54747e1acd
```

The output will be as below

```
#This node has joined the cluster:
#* Certificate signing request was sent to apiserver and a response was received.
#* The Kubelet was informed of the new secure connection details.
#Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```