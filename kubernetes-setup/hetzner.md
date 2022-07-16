# Hetzner

### Buy server

Navigate to [https://hetzner.cloud/](https://hetzner.cloud/?ref=TMLy7z7pZnk9)

I recommend Dedicated AX-Line

![](<../.gitbook/assets/image (5).png>)

I do run serveral AX101 servers - AMD Ryzen™ 9 5950X , 128 GB DDR4 ECC, 2 x 3.84 TB NVME, unlimitted traffic @ 111.86 EUR (including VAT)

![](<../.gitbook/assets/image (6).png>)![](<../.gitbook/assets/image (10).png>)![](<../.gitbook/assets/image (7).png>)![](<../.gitbook/assets/image (12).png>)

In rescue mode format drives to be in sw raid.

After boot up, you have linux up and ready.

### Setup Kubernetes

This is raw install notes. Other setup such as dns might be required as well.

```
passwd

apt update && apt -y dist-upgrade && apt install -y ca-certificates curl gnupg lsb-release apt-transport-https parted mc git vim nfs-kernel-server

parted /dev/nvme0n1 mklabel gpt
parted /dev/nvme0n1 mkpart primary ext4 0% 100%

parted /dev/nvme1n1 mklabel gpt
parted /dev/nvme1n1 mkpart primary ext4 0% 100%
 
mkfs.ext4 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme1n1p1


mkdir /mnt/nvme1
mkdir /mnt/nvme2

mount /dev/nvme0n1p1 /mnt/nvme1
mount /dev/nvme1n1p1 /mnt/nvme2

adduser scholtz
usermod -aG sudo scholtz

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt update && apt dist-upgrade -y && apt install -y kubelet kubeadm kubectl docker-ce docker-ce-cli containerd.io && apt-mark hold kubelet kubeadm kubectl


containerd config default | sudo tee /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep systemd_cgroup
sed -i -e "s?systemd_cgroup = false?systemd_cgroup = true?g" /etc/containerd/config.toml

cat /etc/containerd/config.toml | grep SystemdCgroup
/etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

sed -i -e "s?SystemdCgroup = false?SystemdCgroup = true?g" /etc/containerd/config.toml

sed -i -e "s?cgroupDriver: systemd?SystemdCgroup = true?g" /var/lib/kubelet/config.yaml

cat /etc/systemd/system/multi-user.target.wants/docker.service | grep ExecStart
sed -i -e "s?ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock?ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock  --exec-opt native.cgroupdriver=systemd?g" /etc/systemd/system/multi-user.target.wants/docker.service


cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF




sudo sysctl --system
sudo swapoff -a
vim /etc/fstab # remove swap

UUID=9bcd1871-ed32-497d-b8bd-945c711b85xx /mnt/nvme2  ext4    defaults    0   0
UUID=4b532733-49dd-4086-b80c-679239b8bcxx /mnt/nvme1  ext4    defaults    0   0

vim /run/systemd/resolve/resolve.conf

systemctl enable docker
systemctl enable kubelet
reboot

# reset
sudo kubeadm reset
sudo rm /root/.kube/ -rf
sudo rm /etc/kubernetes/ -rf
sudo rm /etc/cni/net.d -rf
sudo rm /home/pod/.kube/ -rf

kubeadm token create --print-join-command # to allow other node to join
kubeadm init phase upload-certs --upload-certs # to allow other node to join control planes
kubectl taint nodes --all node-role.kubernetes.io/master- # to allow schedule on all nodes


sudo kubeadm init --control-plane-endpoint "k8s.h2.domain.com:6443" --upload-certs

mkdir -p /home/scholtz/.kube
sudo cp -i /etc/kubernetes/admin.conf /home/scholtz/.kube/config
sudo chown scholtz:scholtz /home/scholtz/.kube/config
sudo mkdir -p /root/.kube
sudo cp -i /etc/kubernetes/admin.conf /root/.kube/config
sudo chown root:root /root/.kube/config


curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml

#  add to /etc/kubernetes/manifests/kube-apiserver.yaml
    - --service-node-port-range=25-60000
kubectl apply -f /etc/kubernetes/manifests/kube-apiserver.yaml

kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.2/deploy/static/provider/aws/deploy.yaml

#kubectl edit cm ingress-nginx-controller -n ingress-nginx

kubectl edit service ingress-nginx-controller -n ingress-nginx # Type=NodePort, port 80,443

kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.1/cert-manager.yaml
kubectl apply -f acme.yaml



kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.1/aio/deploy/recommended.yaml
kubectl apply -f dashboard.yaml
kubectl apply -f admin.yaml

wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml


apt install nfs-kernel-server
mkdir -p /mnt/k8s
echo "/mnt/k8s     {ip}(rw,sync,no_subtree_check)" >> /etc/exports
echo "/mnt/k8s     65.21.233.177(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -ar
exportfs -v
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash


kubectl create namespace nfs-provisioner
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
helm repo update
helm template nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner     --set nfs.server={ip}     --set nfs.path=/mnt/k8s --set storageClass.reclaimPolicy=Retain --set storageClass.allowVolumeExpansion=true --set storageClass.name=nfs-fast-retain --set replicaCount=2 --namespace=nfs-provisioner > tmpl.nfs-provisioner.yaml
kubectl apply -f tmpl.nfs-provisioner.yaml  --namespace=nfs-provisioner

chmod 0777 /home/data

```
