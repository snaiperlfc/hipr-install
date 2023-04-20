## Install anydesk to Rocky Linux 9

````bash
sudo tee /etc/yum.repos.d/anydesk.repo<<EOF [anydesk] name=AnyDesk Rocky Linux baseurl=http://rpm.anydesk.com/centos/x86_64/ gpgcheck=1 repo_gpgcheck=1 gpgkey=https://keys.anydesk.com/repos/RPM-GPG-KEY EOF

dnf repolist | grep anydesk

sudo dnf install anydesk -y
````

````bash
sudo dnf update --refresh
````

````bash
sudo nano /etc/gdm/custom.conf
````

add following:

````
WaylandEnable=false
AutomaticLoginEnable = true
AutomaticLogin = <my_user_name>
````

## Install Kubernetes Master Node on Rocky Linux 9

Set a proper FQDN (Fully Qualified Domain Name) for your Kubernetes server. Also use the Local DNS resolver (/etc/hosts)
for name resolution of your hostname.

````bash
hostnamectl set-hostname kubemaster-01.centlinux.com
echo 192.168.116.131 kubemaster-01.centlinux.com kubemaster-01 >> /etc/hosts
````

Refresh your cache of enabled yum repositories.

````bash
dnf makecache --refresh
````

Execute following dnf command to update your Rocky Linux server.

````bash
dnf update -y
````

Linux Kernel packages may be updated by the above command. Therefore, reboot your Linux server before moving forward.

````bash
reboot
````

After reboot, check the Linux Kernel and operating system versions that are being used in this configuration guide.

````bash
cat /etc/rocky-release
uname -r
````

Execute following commands at Linux bash to set SELinux permissive mode.

````bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
````

Kubernetes requires "overlay" and "br_netfilter" Kernel modules. Therefore, you can use following group of commands to
permanently enable them.

````bash
modprobe overlay
modprobe br_netfilter
cat > /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF
````

Set following Kernel parameter as required by Kubernetes software.

````bash
cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
````

Reload Kernel parameter configuration files with above changes.

````bash
sysctl --system
````

Execute following commands to do the same.

````bash
swapoff -a
sed -e '/swap/s/^/#/g' -i /etc/fstab
````

Verify the usage of Swap storage on your Linux server.

````bash
free -m
````

Containerd is not available in standard yum repositories, therefore, you may need to install Docker Official Yum
Repository to install Container runtime.

````bash
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
````

Build yum cache for Docker yum repository.

````bash
dnf makecache
````

Now, you can easily install Containerd runtime by using dnf command.

````bash
mv /etc/containerd/config.toml /etc/containerd/config.toml.orig
containerd config default > /etc/containerd/config.toml
````

Edit Containerd configuration file by using vim text editor.

````bash
nano /etc/containerd/config.toml
````

Locate and set SystemdCgroup parameter in this file, to enable the systemd cgroup driver for Containerd runtime.

````
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
````

Enable and start Containerd service.

````bash
systemctl enable --now containerd.service
````

Check the status of Containerd service for any errors.

````bash
systemctl status containerd.service
````

Therefore, you need to allow these service ports in Linux firewall.

````bash
firewall-cmd --permanent --add-port={6443,2379,2380,10250,10251,10252}/tcp
firewall-cmd --reload
````

The following command will add the Kubernetes repository in your Linux server.

````bash
cat > /etc/yum.repos.d/k8s.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
````

Build the yum cache for Kubernetes yum repository.

````bash
dnf makecache
````

Now, you can install Kubernetes packages by using dnf command.

````bash
dnf install -y {kubelet,kubeadm,kubectl} --disableexcludes=kubernetes
````

Enable and start kubelet.service. It is the main Kubernetes service that waits for any events when you initialize the
cluster or join a node to this cluster.

````bash
systemctl enable --now kubelet.service
````

Verify the status of kubelet.service.

````bash
systemctl status kubelet
````

### Enable Bash Completion for Kubernetes Commands:

To enable automatic completion of kubectl commands, you have to execute the script provided by kubectl command itself.
You must ensure that bash-completion package is already installed on your Rocky Linux server.

````bash
source <(kubectl completion bash)
kubectl completion bash > /etc/bash_completion.d/kubectl
````

### Installing Flannel CNI Plugin:

In this configuration guide, we are using Flannel CNI plugin. Ensure that this plugin must be installed on each
Kubernetes node.

Create a directory and download flanneld file therein.

````bash
mkdir /opt/bin
curl -fsSLo /opt/bin/flanneld https://github.com/flannel-io/flannel/releases/download/v0.20.1/flannel-v0.20.1-linux-amd64.tar.gz
````

Grant execution permissions to flanneld file to make it an executable.

````bash
chmod +x /opt/bin/flanneld
````

### Initializing Kubernetes Control Plane:

````bash
kubeadm config images pull
````

After successful downloading of images, execute the following command to initialize the Kubernetes cluster on
kubemaster-01.centlinux.com server. This nod will be selected as the Kubernetes control plane because it is the first
node in the cluster.

````bash
kubeadm init
````

Execute following command to set KUBECONFIG variable for all sessions.

````bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile.d/k8s.sh
````

Execute following commands as a user, that is being used to manage your Kubernetes cluster.

````bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
````

Execute kubectl commands to check the status of your Kubernetes cluster.

````bash
kubectl get nodes
kubectl cluster-info
````

After Kubernetes Control Plane is started, run the following command to install the Flannel Pod network plugin. This
command will automatically run the "flanneld" binary file and start some flannel pods.

````bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
````

Check the list of running pods on your Kubernetes cluster.

````bash
kubectl get pods --all-namespaces

````

Your Kubernetes master node has been installed successfully.

### Config

Create namespace

````bash
kubectl create namespace dcs
````

Set default namespace

````bash
kubectl config set-context --current --namespace=dcs
````

### Config ceph

````bash
mkdir /mnt/cephfs

ceph-authtool --name client.admin /etc/ceph/ceph.client.admin.keyring --print-key | tee /etc/ceph/admin.secret

mount -t ceph master-node:6789,worker01:6789,worker02:6789:/ /mnt/cephfs -o
name=admin,secretfile=/etc/ceph/admin.secret,noatime
````

### Calico config

````bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O
kubectl apply -f calico.yaml
````

### Install ingress

````bash
helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true --set
controller.enableSnippets=true

````
