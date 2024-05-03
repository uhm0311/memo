# install docker
https://docs.docker.com/engine/install/centos/

# reset /etc/resolv.conf : remove kube-dns svc cluster address(maybe "nameserver 10.96.0.10") from /etc/resolv.conf.
sudo vim /etc/resolv.conf

# remove deprecated k8s repository
sudo rm -rf /etc/yum.repos.d/kubernetes.repo

# remove installed k8s
yum erase -y kubeadm
yum erase -y kubelet
yum erase -y kubectl
yum erase -y cri-tools

# install cni plugins
CNI_PLUGINS_VERSION="v0.8.7"
ARCH="amd64"
DEST="/opt/cni/bin"
sudo mkdir -p "$DEST"
curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGINS_VERSION}/cni-plugins-linux-${ARCH}-${CNI_PLUGINS_VERSION}.tgz" | sudo tar -C "$DEST" -xz

# mkdir
DOWNLOAD_DIR="/usr/local/bin"
sudo mkdir -p "$DOWNLOAD_DIR"

# install kubeadm, kubelet, kubectl
RELEASE="v1.15.0"
ARCH="amd64"
cd $DOWNLOAD_DIR
sudo curl -L --remote-name-all https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet,kubectl}
sudo chmod +x {kubeadm,kubelet,kubectl}

RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
sudo mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# node reset
sudo rm -f /run/flannel/subnet.env
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
sudo ip link set cni0 down && sudo ip link set flannel.1 down
sudo ip link delete cni0 && sudo ip link delete flannel.1
sudo kubeadm reset

export K8S_VERSION=1.15.10
sudo yum remove -y kubectl kubelet kubeadm
sudo yum install -y kubernetes-cni-0.8.7 kubectl-$K8S_VERSION kubelet-$K8S_VERSION kubeadm-$K8S_VERSION

# master node
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

rm -rf $HOME/.kube
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=$HOME/.kube/config

sudo mkdir -p /run/flannel
cat <<EOF | sudo tee /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
EOF

kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.13.0/Documentation/kube-flannel.yml

watch -n 1 kubectl get pods -A
watch -n 1 kubectl get nodes

# worker node
sudo mkdir -p /run/flannel
cat <<EOF | sudo tee /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
EOF

sudo kubeadm token create --print-join-command # master
sudo kubeadm join ... # worker

mkdir -p ~/.kube
vim ~/.kube/config # copy and paste kube config.

export KUBECONFIG=$HOME/.kube/config

# all node
sudo vim /etc/resolv.conf #insert kube-dns svc cluster address(maybe "nameserver 10.96.0.10") to /etc/resolv.conf.
