## Getting Started to [KUBE](https://kubernetes.io/docs/home/)    

_วิธีติดตั้ง Kubernetes, Kong และ Konga_

*****************************************************************************************

## Contents

- [**Initial setup**](#Setup-auto-completion-and-alias-on-terminal)
- [**Requirements**](#Prerequisites)
- [**Compatibility**](#compatibility)
- [**Prerequisites**](#prerequisites)
- [**Used libraries**](#used-libraries)
- [**Installation**](#installation)
- [**Configuration**](#configuration)
- [**Environment variables**](#environment-variables)
- [**Running Konga**](#running-konga)
- [**Upgrading**](#upgrading)
- [**FAQ**](#faq)
- [**More Kong related stuff**](#more-kong-related-stuff)
- [**License**](#license)

*****************************************************************************************

## Setup auto completion and alias on terminal
_ก่อนเริ่มใส่คำสั่ง shortcut_
```
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```
คำสั่งย่อต่างๆ
```
Pods = po
ReplicaSets = rs
Deployments = deploy
Services = svc
Namespaces = ns
Network Policies = netpol
Persistent Volumes = pv
PersistentVolumeClaims = pvc
Service Accounts = sa
```

## Prerequisites
- Cloud สำหรับติดตั้ง [Nipa](https://www.nipa.cloud/)
- CentOS 7 >= 8, <= 12.x (12.16 LTS is recommended)
- PostgresSQL < 14
- 3-4 ready instances

*****************************************************************************************

## Setting up instances
```
ssh nc-user@x.x.x.x
```
* Add `nameserver`
```
vi /etc/resolv.conf

nameserver 8.8.8.8
nameserver 9.9.9.9
```
* Setup
```
yum update
yum install yum-utils -y
yum install firewalld

setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
modprobe br_netfilter
modprobe overlay
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
swapoff -a

sudo vi /etc/fstab
#/dev/mapper/centos-swap swap swap defaults 0 0
```
[Fixed] Failed to set locale, defaulting to C
_test edit vi /etc/environment using en_US:_
```
sudo vi /etc/environment

LC_ALL="en_US.UTF-8" 
LC_CTYPE="en_US.UTF-8"
 LANGUAGE="en_US.UTF-8"
```
_or_
```
vi /etc/bashrc
export LANG=en_US.UTF-8
export LANGUAGE=en_US.UTF-8
export LC_COLLATE=C
export LC_CTYPE=en_US.UTF-8
export LC_ALL="en_US.UTF-8"
source /etc/bashrc
```

* Install Firewall
```
yum install firewalld

systemctl start firewalld
systemctl enable firewalld
systemctl status firewalld
```
* Setup Firewall
```
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10251/tcp
firewall-cmd --permanent --add-port=10252/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --permanent --add-port=8472/udp
firewall-cmd --permanent --add-port=8001/tcp
firewall-cmd --permanent --add-port=9090/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
```

## Install Docker
* Setup Docker
_Download and Install_
```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo -y
yum install -y docker-ce
```
_make new a directory_
```
sudo mkdir /etc/docker
```
_edit file_
```
sudo vi /etc/docker/daemon.json
```
_depend what you are running on_
```
{
 "exec-opts": ["native.cgroupdriver=systemd"],
 "log-driver": "json-file",
 "log-opts": {
 "max-size": "100m"
 },
 "storage-driver": "overlay2"
}
--- vmware
{
 "exec-opts": ["native.cgroupdriver=systemd"],
 "log-driver": "json-file",
 "log-opts": {
 "max-size": "100m"
 },
 "storage-driver": "devicemapper"
}
--end vmware
```
Reload and start up
```
  systemctl daemon-reload
  systemctl start docker && systemctl enable docker
```
## Kubernetes
* EOF [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
* Check Version
```
yum list --showduplicates kubelet --disableexcludes=kubernetes -y
yum list --showduplicates kubeadm --disableexcludes=kubernetes -y
yum list --showduplicates kubectl --disableexcludes=kubernetes -y
```
* Install and Start
```
yum install -y kubelet-1.23.3 kubectl-1.23.3 kubeadm-1.23.3
#!!! yum install -y kubelet kubeadm kubectl
systemctl start kubelet && systemctl enable kubelet
```
_or_
```
# Download
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Check
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

# Validate
echo "$(<kubectl.sha256)  kubectl" | sha256sum --check

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
* DONE

*****************************************************************************************

## Master node
**For Master node only**

* 0

[for more](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
```
ifconfig
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.148.0.6
```
* 1 / run as a regular user
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
* 2
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
-- sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
* 3
```
kubectl get pods --all-namespaces
 --!! wait for ready all
```
* 4
```
sudo kubectl get nodes
```
* 5
```
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
```
* Generate Tokens / 6.16

```
kubeadm token create kube.dev001 --print-join-command

kubeadm token create hp9b0k.1g9tqz8vkf78ucwf --print-join-command
```

## Final Touch

```
docker pull devapi47/kube1:v1.0
```

* LoadBalancer
```
kubectl expose deployment/nginx-deployment --name=nginx-api1 --port=80 --type=LoadBalancer
```
## Kong and Konga

### Kong

* Install Kong
```
--- install docker konga
docker pull pantsel/konga
docker run -d -p 1337:1337 --name konga pantsel/konga
```


*****************************************************************************************

## Short key

1. ##### Prepare the database
> **Note**: You can skip this step if using the `mongo` adapter.

You can prepare the database using an ephemeral container that runs the prepare command.

**Args**

argument  | description | default
----------|-------------|--------
-c      | command | -
-a      | adapter (can be `postgres` or `mysql`) | -
-u     | full database connection url | -

```
$ docker run --rm pantsel/konga:latest -c prepare -a {{adapter}} -u {{connection-uri}}
```

[It is possible to seed default users on first install.](./docs/SEED_DEFAULT_DATA.md)

You may also configure Konga to authenticate via [LDAP](./docs/LDAP.md).

*****************************************************************************************

## Problems
Clear ssh key:
```
ssh-keygen -R 10.101.108.101
```

check firewall:

[for more](https://www.liquidweb.com/kb/how-to-stop-and-disable-firewalld-on-centos-7/)
```
systemctl status firewalld
sudo firewall-cmd --zone=work --list-all
```
Can't start firewall:
```
```
Upgrade Kernel:
```
grub2-mkconfig -o /boot/grub2/grub.cfg
or
mv /boot/grub/grub.conf /boot/grub/bk_grub.conf
yum -y update && yum -y reinstall kernel
```
## More Kong related stuff
- [**Kong Admin proxy**](https://github.com/pantsel/kong-admin-proxy)
- [**Kong Middleman plugin**](https://github.com/pantsel/kong-middleman-plugin)

## Author

Panagis Tselentis

## Documentation

Documentation is available at [abbok.net](https://www.abbok.net/).
