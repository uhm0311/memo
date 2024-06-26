# reset /etc/resolv.conf
- remove kube-dns svc cluster address(maybe "nameserver 10.96.0.10") from /etc/resolv.conf.

```bash
sudo vim /etc/resolv.conf
```

# reset kubernetes

```bash
sudo rm -f /run/flannel/subnet.env
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
sudo ip link set cni0 down && sudo ip link set flannel.1 down
sudo ip link delete cni0 && sudo ip link delete flannel.1

sudo systemctl stop kubelet
sudo kubeadm reset --force
```

## reset docker

```bash
sudo yum remove -y docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

sudo yum remove -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

export DOCKER_VERSION="18.09.9"
sudo yum install -y docker-ce-$DOCKER_VERSION docker-ce-cli-$DOCKER_VERSION containerd.io docker-compose-plugin

sudo systemctl daemon-reload
sudo systemctl restart docker

docker --version
```

## remove all docker files

```bash
sudo docker kill $(docker ps -q)
sudo docker rm -f $(docker ps -a -q)
sudo docker volume rm $(docker volume ls -q)
sudo docker rmi -f $(docker images -a -q)
sudo systemctl stop docker
sudo rm -rf /var/lib/docker
sudo systemctl start docker
```

## remove deprecated kubernetes repository

```bash
sudo rm -rf /etc/yum.repos.d/kubernetes.repo
```

## remove installed k8s

```bash
sudo yum erase -y kubeadm
sudo yum erase -y kubelet
sudo yum erase -y kubectl
sudo yum erase -y cri-tools

sudo rm -rf /usr/local/bin/kubeadm
sudo rm -rf /usr/local/bin/kubelet
sudo rm -rf /usr/local/bin/kubectl
sudo rm -rf /opt/cni/bin

sudo rm -rf /usr/bin/kubeadm
sudo rm -rf /usr/bin/kubelet
sudo rm -rf /usr/bin/kubectl
```

# install kubernetes

## install cni plugins

```bash
CNI_PLUGINS_VERSION="v0.8.7"
ARCH="amd64"
DEST="/opt/cni/bin"
sudo mkdir -p "$DEST"
curl -L "https://github.com/containernetworking/plugins/releases/download/${CNI_PLUGINS_VERSION}/cni-plugins-linux-${ARCH}-${CNI_PLUGINS_VERSION}.tgz" | sudo tar -C "$DEST" -xz
```

## install kubeadm, kubelet, kubectl

```bash
DOWNLOAD_DIR="/usr/local/bin"
sudo mkdir -p "$DOWNLOAD_DIR"

RELEASE="v1.15.12"
ARCH="amd64"
cd $DOWNLOAD_DIR
sudo curl -L --remote-name-all https://dl.k8s.io/release/${RELEASE}/bin/linux/${ARCH}/{kubeadm,kubelet,kubectl}
sudo chmod +x {kubeadm,kubelet,kubectl}

sudo ln -s $DOWNLOAD_DIR/kubeadm /usr/bin/kubeadm
sudo ln -s $DOWNLOAD_DIR/kubelet /usr/bin/kubelet
sudo ln -s $DOWNLOAD_DIR/kubectl /usr/bin/kubectl

RELEASE_VERSION="v0.4.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service
sudo mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${DOWNLOAD_DIR}:g" | sudo tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
sudo systemctl daemon-reload

sudo modprobe br_netfilter
sudo bash -c "echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables"
sudo rm -rf /var/lib/etcd
```

# master node

## kubeadm init

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

## edit kubelet config

```bash
sudo vim /var/lib/kubelet/config.yaml
```

### before

```conf
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
```

### after

```conf
evictionHard:
  imagefs.available: 1%
  memory.available: 1Mi
  nodefs.available: 1%
  nodefs.inodesFree: 1%
```

```bash
sudo systemctl restart kubelet
```

## copy kubectl config

```bash
rm -rf $HOME/.kube
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=$HOME/.kube/config
```

## config flannel

```bash
sudo mkdir -p /run/flannel
cat <<EOF | sudo tee /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
EOF
```

## install flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.13.0/Documentation/kube-flannel.yml

watch -n 1 kubectl get pods -A
watch -n 1 kubectl get nodes
```

## create join token

```bash
sudo kubeadm token create --print-join-command
```

# worker node

## config flannel

```bash
sudo mkdir -p /run/flannel
cat <<EOF | sudo tee /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
EOF
```

## join cluster

```bash
sudo kubeadm join ...
```

## copy kubectl config

```bash
mkdir -p ~/.kube
vim ~/.kube/config
```

```bash
export KUBECONFIG=$HOME/.kube/config
```

# config /etc/resolv.conf

- insert kube-dns svc cluster address(maybe "nameserver 10.96.0.10") to /etc/resolv.conf.

```bash
sudo vim /etc/resolv.conf
```

# config /etc/yum.conf

- insert value `docker* containerd* kube*` to key `exclude`.
- packages of key `exclude` will be excluded when command `yum update`.

```bash
sudo vim /etc/yum.conf
```
