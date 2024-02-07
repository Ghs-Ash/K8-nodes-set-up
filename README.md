https://devopscube.com/setup-kubernetes-cluster-kubeadm/




Kubeadm is a tool to set up a minimum viable Kubernetes cluster without much complex configuration. Also, Kubeadm makes the whole process easy by running a series of prechecks to ensure that the server has all the essential components and configs to run Kubernetes.
It is developed and maintained by the official Kubernetes community. There are other options like minikube, kind, etc., that are pretty easy to set up. You can check out my minikube tutorial. Those are good options with minimum hardware requirements if you are deploying and testing applications on Kubernetes.
But if you want to play around with the cluster components or test utilities that are part of cluster administration, Kubeadm is the best option. Also, you can create a production-like cluster locally on a workstation for development and testing purposes.

Following are the prerequisites for Kubeadm Kubernetes cluster setup.

Minimum two Ubuntu nodes [One master and one worker node]. You can have more worker nodes as per your requirement.
The master node should have a minimum of 2 vCPU and 2GB RAM.
For the worker nodes, a minimum of 1vCPU and 2 GB RAM is recommended.
10.X.X.X/X network range with static IPs for master and worker nodes. We will be using the 192.x.x.x series as the pod network range that will be used by the Calico network plugin. Make sure the Node IP range and pod IP range don’t overlap.
Note: If you are setting up the cluster in the corporate network behind a proxy, ensure set the proxy variables and have access to the container registry and docker hub. Or talk to your network administrator to whitelist registry.k8s.io to pull the required images.

Execute the following commands on all the nodes for IPtables to see bridged traffic. Here we are tweaking some kernel parameters and setting them using sysctl.

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

For kubeadm to work properly, you need to disable swap on all the nodes using the following command.

sudo swapoff -a
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
The fstab entry will make sure the swap is off on system reboots.

You can also, control swap errors using the kubeadm parameter --ignore-preflight-errors Swap we will look at it in the latter part.

Note: From 1.28 kubeadm has beta support for using swap with kubeadm clusters. Read this to understand more.
Install CRI-O Runtime On All The Nodes

Note: We are using cri-o instead if containerd because, in Kubernetes certification exams, cri-o is used as the container runtime in the exam clusters.
The basic requirement for a Kubernetes cluster is a container runtime. You can have any one of the following container runtimes.

CRI-O
containerd
Docker Engine (using cri-dockerd)
We will be using CRI-O instead of Docker for this setup as Kubernetes deprecated Docker engine

As a first step, we need to install cri-o on all the nodes. Execute the following commands on all the nodes.

Enable cri-o repositories for version 1.28

OS="xUbuntu_22.04"

VERSION="1.28"

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF
Add the GPG keys for CRI-O to the system’s list of trusted keys.

curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
Update and install crio and crio-tools.

sudo apt-get update
sudo apt-get install cri-o cri-o-runc cri-tools -y
Reload the systemd configurations and enable cri-o.

sudo systemctl daemon-reload
sudo systemctl enable crio --now
The cri-tools contain crictl, a CLI utility to interact with the containers created by the container runtime. When you use container runtimes other than Docker, you can use the crictl utility to debug containers on the nodes. Also, it is useful in CKS certification where you need to debug containers.

Install Kubeadm & Kubelet & Kubectl on all Nodes

Install the required dependencies.

sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
Download the GPG key for the Kubernetes APT repository.

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://dl.k8s.io/apt/doc/apt-key.gpg
Add the Kubernetes APT repository to your system.

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
Update apt repo

sudo apt-get update -y
Note: If you are preparing for Kubernetes certification, install the specific version of kubernetes. For example, the current Kubernetes version for CKA, CKAD and CKS exams is kubernetes version 1.28
You can use the following commands to find the latest versions.

sudo apt update
apt-cache madison kubeadm | tac
Specify the version as shown below.

sudo apt-get install -y kubelet=1.28.2-00 kubectl=1.28.2-00 kubeadm=1.28.2-00
Or, to install the latest version from the repo use the following command without specifying any version.

sudo apt-get install -y kubelet kubeadm kubectl
Add hold to the packages to prevent upgrades.

sudo apt-mark hold kubelet kubeadm kubectl
Now we have all the required utilities and tools for configuring Kubernetes components using kubeadm.

Add the node IP to KUBELET_EXTRA_ARGS.

sudo apt-get install -y jq
local_ip="$(ip --json a s | jq -r '.[] | if .ifname == "eth1" then .addr_info[] | if .family == "inet" then .local else empty end else empty end')"
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF
Initialize Kubeadm On Master Node To Setup Control Plane

Here you need to consider two options.

Master Node with Private IP: If you have nodes with only private IP addresses the API server would be accessed over the private IP of the master node.
Master Node With Public IP: If you are setting up a Kubeadm cluster on Cloud platforms and you need master Api server access over the Public IP of the master node server.
Only the Kubeadm initialization command differs for Public and Private IPs.

Execute the commands in this section only on the master node.

If you are using a Private IP for the master Node,

Set the following environment variables. Replace 10.0.0.10 with the IP of your master node.

IPADDR="10.0.0.10"
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
If you want to use the Public IP of the master node,

Set the following environment variables. The IPADDR variable will be automatically set to the server’s public IP using ifconfig.me curl call. You can also replace it with a public IP address

IPADDR=$(curl ifconfig.me && echo "")
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
Now, initialize the master node control plane configurations using the kubeadm command.

For a Private IP address-based setup use the following init command.

sudo kubeadm init --apiserver-advertise-address=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
--ignore-preflight-errors Swap is actually not required as we disabled the swap initially.

For Public IP address-based setup use the following init command.

Here instead of --apiserver-advertise-address we use --control-plane-endpoint parameter for the API server endpoint.

sudo kubeadm init --control-plane-endpoint=$IPADDR  --apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR --node-name $NODENAME --ignore-preflight-errors Swap
All the other steps are the same as configuring the master node with private IP.

Note: You can also pass the kubeadm configs as a file when initializing the cluster. See Kubeadm Init with config file
On a successful kubeadm initialization, you should get an output with kubeconfig file location and the join command with the token as shown below. Copy that and save it to the file. we will need it for joining the worker node to the master.

 Kubeadm init command output
Use the following commands from the output to create the kubeconfig in master so that you can use kubectl to interact with cluster API.

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
Now, verify the kubeconfig by executing the following kubectl command to list all the pods in the kube-system namespace.

kubectl get po -n kube-system
You should see the following output. You will see the two Coredns pods in a pending state. It is the expected behavior. Once we install the network plugin, it will be in a running state

You verify all the cluster component health statuses using the following command.

kubectl get --raw='/readyz?verbose'
You can get the cluster info using the following command.

kubectl cluster-info 
By default, apps won’t get scheduled on the master node. If you want to use the master node for scheduling apps, taint the master node.

kubectl taint nodes --all node-role.kubernetes.io/control-plane-
Install Calico Network Plugin for Pod Networking

Kubeadm does not configure any network plugin. You need to install a network plugin of your choice for kubernetes pod networking and enable network policy.

I am using the Calico network plugin for this setup.

Note: Make sure you execute the kubectl command from where you have configured the kubeconfig file. Either from the master of your workstation with the connectivity to the kubernetes API.
Execute the following commands to install the Calico network plugin operator on the cluster.

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml -O

kubectl create -f custom-resources.yaml
After a couple of minutes, if you check the pods in kube-system namespace, you will see calico pods and running CoreDNS pods.Join Worker Nodes To Kubernetes Master Node

We have set up cri-o, kubelet, and kubeadm utilities on the worker nodes as well.

Now, let’s join the worker node to the master node using the Kubeadm join command you have got in the output while setting up the master node.

If you missed copying the join command, execute the following command in the master node to recreate the token with the join command.

kubeadm token create --print-join-command
Here is what the command looks like. Use sudo if you running as a normal user. This command performs the TLS bootstrapping for the nodes.

sudo kubeadm join 10.128.0.37:6443 --token j4eice.33vgvgyf5cxw4u8i \
    --discovery-token-ca-cert-hash sha256:37f94469b58bcc8f26a4aa44441fb17196a585b37288f85e22475b00c36f1c61
On successful execution, you will see the output saying, “This node has joined the cluster”.

