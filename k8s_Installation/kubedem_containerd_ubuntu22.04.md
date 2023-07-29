Kubernetes v1.26 Cluster Setup with Kubeadm and Containerd on AWS Ubuntu 22.04
!! Kubernetes Cluster Setup:
•	To run containers in Pods, Kubernetes uses a container runtime.
•	Kubernetes works with all container runtimes that implement a standard known as the container runtime interface (CRI). This is essentially a standard way to communicate between Kubernetes and the container runtime, and any runtime that supports this standard automatically works with Kubernetes. 
•	Docker does not implement CRI, so Kubernetes implemented Docker shim, an additional layer to serve as an interface between Kubernetes and Docker. Now, however, there are plenty of runtimes available that implement the CRI, and it no longer makes sense for Kubernetes to maintain special support for Docker. 
•	Mirantis and Docker have agreed to partner to maintain shim code standalone outside Kubernetes, as a conformant CRI interface for the Docker Engine API. They call it cri-dockered.
•	Docker is not actually a container runtime! It’s actually a collection of tools that sits on top of a container runtime called containerd. 
•	If you use Docker, it means you already use containerd as runtime. So in a production environment no need for Docker, as we only need the runtime, i.e. containerd lightweight runtimes. 
•	So we can deploy directly containerd in a production environment.
 
!! Before we deploy the Docker.
Kubelet uses to talk with CRI to deploy containers. As Docker doesn’t support CRI, we are using Docker shim which use to be supported and provided by Kubernetes. 
Now Kubernetes doesn’t provide docker shim as the container runtime which is provided by containerd can use directly as it supports the CRI. Now kubelet talks directly with the CRI for deploys containers. 

Cluster Setup on AWS using Kubeadm:
In Kubeadm the control plane component like API server, scheduler, and ETCD will run as a POD, so we should have containerd or container runtime supported by Kubernetes. 
Prerequisites:
•	A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
•	2 GB or more of RAM per machine (any less will leave little room for your apps).
•	2 CPUs or more.
•	Full network connectivity between all machines in the cluster (public or private network is fine).
•	Unique hostname, MAC address, and product_uuid for every node.
•	Certain ports are open on your machines.
•	Swap disabled. You MUST disable swap in order for the kubelet to work properly.
o	For example, sudo swapoff -a will disable swapping temporarily. To make this change persistent across reboots, make sure swap is disabled in config files like /etc/fstab, systemd.swap, depending how it was configured on your system.

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ 
The above link is for the installation guide for Kubeadm.
1.	As per the instruction, the certain port we need to open, 
https://kubernetes.io/docs/reference/networking/ports-and-protocols/ 
•	On Master Node: Open the below ports in the security group
6443/tcp        for Kubernetes API Server
2379-2380       for etcd server client API
6783/tcp,6784/  udp for Weavenet CNI
10248-10260     for Kubelet API, Kube-scheduler, Kube-controller-manager, Read-Only Kubelet API, Kubelet health
80,8080,443     Generic Ports
30000-32767 for NodePort Services
•	On Slave Node: Open the below ports in the security group
6783/tcp,6784/  udp for Weavenet CNI
10248-10260 for Kubelet API etc
30000-32767 for NodePort Services
2.	Now add the unique name for the nodes as master, slave1, and slave2. 
hostnamectl set-hostname master
hostnamectl set-hostname slave1
hostnamectl set-hostname slave2
Or
Sudo hostname master
Exec bash
3.	Run on all nodes of the cluster as a root user.
sudo su
4.	Disable SWAP
sudo swapoff –a
sudo sed -i -e '/swap/d' /etc/fstab
5.	Installing a containerd 
From the Official binaries: 
-	Installing Containerd
https://github.com/containerd/containerd/blob/main/docs/getting-started.md Link for the GitHub Repo Of containerd. 
Download the binaries of containerd
wget https://github.com/containerd/containerd/releases/download/v1.7.3/containerd-1.7.3-linux-amd64.tar.gz 
tar Cxzvf /usr/local containerd-1.6.2-linux-amd64.tar.gz
-	Installing Control Groups:
             In Kubernetes, a "system" generally refers to the collection of components and processes that work together to manage and maintain the overall health and operation of the Kubernetes cluster itself.
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service 
mkdir /usr/local/lib/systemd/system/
mv containerd.service  /usr/local/lib/systemd/system/containerd.service
systemctl daemon-reload
systemctl enable --now containerd
-	Installing runc
wget https://github.com/opencontainers/runc/releases/download/v1.1.8/runc.amd64 
install -m 755 runc.amd64 /usr/local/sbin/run
-	Installing CNI
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz 
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
When we install docker, the docker CLI and docker daemon come in a package, so docker use to install both CLI and daemon. But here we need to install separately a CLI which can interact with containerd.
Detailed instruction is given on the containerd GitHub page. https://github.com/containerd/containerd/blob/main/docs/getting-started.md 
Here I use crictl to interact with containerd (UNIX socket of the containerd).
-	Install CRICTL
VERSION="v1.26.0" # check latest version in /releases page
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
CRICTL is a universal tool to interact with containerd, not specific for containerd. So after installation, we need to attach the CRICTL with the UNIX socket of the containerd. Else CRICTL won’t interact with containerd. 
How to add CRICTL with the UNIX socket of containerd is given in the document of CRI.
https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md 
    CRICTL by default connects on Unix to:
•	unix:///var/run/dockershim.sock or
•	unix:///run/containerd/containerd.sock or
•	unix:///run/crio/crio.sock or
•	unix:///var/run/cri-dockerd.sock

             Now create a file name as /etc/crictl.yaml
vi /etc/crictl.yaml
cat /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false  
pull-image-on-create: false

Or

cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false
pull-image-on-create: false
EOF
Make debug false, else it will generate a lot of debug file. 

Here the containerd installation completed. 
            Have to enable some Network Forwarding. And the details taken from the official website of container Runtime. https://kubernetes.io/docs/setup/production-environment/container-runtimes/           
            
            Forwarding IPv4 and letting iptables see bridged traffic 
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
6.	Install Kubectl, kubelet and kubeadm
apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt update -y
apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm Kubectl
7.	Run the below commands on the master Node to make the Node master.
kubeadm config images pull
kubeadm init

This will pull the kubeadm component such as apiserver, etcd, etc. 
Kubeadm init will initialize the cluster. 
If more than one master node in available then use network flag with kubeadm init,
e.g., sudo kubeadm init --pod-network-cidr=192.168.0.0/16
The pod network CIDR you set during kubeadm init must match the CIDR range supported by the chosen CNI plugin. If the CIDR ranges are not compatible, the networking might not work as expected.
               Install the CNI (Container Network Interface)
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml 
CNI plugins are crucial components in Kubernetes that provide the networking infrastructure necessary for pods and containers to communicate with each other and the outside world. They ensure proper network connectivity, isolation, and security, enabling the smooth functioning of distributed applications within a Kubernetes cluster.
Once the installation completed, generate the joining token from master node and execute the token in slave nodes. 
