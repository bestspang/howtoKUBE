## Getting Started to [KUBE](https://kubernetes.io/docs/home/)    

_วิธีติดตั้ง Kubernetes, Kong และ Konga_

*****************************************************************************************

## Contents
- [**Prerequisites**](#prerequisites)
- [**Setup instances**](#Setting-up-instances)
- [**Install Docker**](#Install-Docker)
- [**Master node**](#Master-node)
- [**Kong and Konga**](#Kong-and-Konga*)
- [**Running Konga**](#running-konga)
- [**Upgrading**](#upgrading)
- [**Tips**](#Tips)
- [**FAQ**](#faq)
- [**More Kong related stuff**](#more-kong-related-stuff)
- [**License**](#license)

*****************************************************************************************

## Tips
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

docker ps
```

* LoadBalancer
```
kubectl expose deployment/nginx-deployment --name=nginx-api1 --port=80 --type=LoadBalancer
```
## Kong and Konga

### Install database

* Install environment
```
yum install -y gcc gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel git
```

* Install postgresql 10
```
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install -y postgresql10-server
yum install -y postgresql10-server
/usr/pgsql-10/bin/postgresql-10-setup initdb
systemctl enable postgresql-10
systemctl start postgresql-10
```

* Change password postgres and add user kong
```
# passwd postgres
# adduser kong
# passwd kong
```
* Create DB for Kong
```
psql
create user kong with password 'your_password';
create database kong owner kong;
grant all privileges on database kong to kong;
\q
exit;
```

* Config Postgres 1
_trust, peer, md5_
```
# vi /var/lib/pgsql/10/data/pg_hba.conf

add this

host    all   all  127.0.0.1/32  trust
```

* Config Postgres 2
```
vi /var/lib/pgsql/10/data/postgresql.conf

listen_addresses = 'localhost'

systemctl restart postgresql-10.service
```


### Install Kong

* Install Kong
_more information [here](https://docs.konghq.com/gateway/2.7.x/install-and-run/centos/)_
```
sudo yum install kong-2.7.1
```

* Prepare config file
```
# cp /etc/kong/kong.conf.default /etc/kong/kong.conf
# vim /etc/kong/kong.conf
```

* Edit file
```
admin_listen =0.0.0.0:8001

database = postgres
pg_host = 127.0.0.1
pg_port 5432
pg_user = kong
pg_password = your_password #PASSWORD POSTGRESQL USER KONG
pg_database = kong
```

* Start kong
```
# kong migrations bootstrap -c/etc/kong/kong.conf
# kong start -c/etc/kong/kong.conf --vv
# curl 127.0.0.1:8001
```

### Install Konga
* Install Kong
_more [here](https://hub.docker.com/r/pantsel/konga/#prerequisites)_
konga [repo](https://github.com/pantsel/konga)

```
docker pull pantsel/konga

docker run -d -p 1337:1337 --name konga pantsel/konga
```
_or_
```
docker run -p 1337:1337 \
             --network {{kong-network}} \ // optional
             --name konga \
             -e "NODE_ENV=production" \ // or "development" | defaults to 'development'
             -e "TOKEN_SECRET={{somerandomstring}}" \
             pantsel/konga
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

## FAQ
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

Nginx:
```
service nginx restart
systemctl restart nginx
```
```
setsebool -P httpd_can_network_connect 1
or
setsebool -P httpd_can_network_relay 1

firewall-cmd --set-default-zone=trusted
```
https://stackoverflow.com/questions/35177177/nginx-error-13-permission-denied-while-connecting-to-upstream
https://stackoverflow.com/questions/23948527/13-permission-denied-while-connecting-to-upstreamnginx
https://stackoverflow.com/questions/52039898/error-while-kubernetes-kubeadm-initialization
https://serverfault.com/questions/1044211/docker-nginx-php-fpm-error-502-bad-gateway
https://stackoverflow.com/questions/21524373/nginx-connect-failed-111-connection-refused-while-connecting-to-upstream

## Documentation
### Nginx
https://shouts.dev/install-nginx-on-centos-7
### Learning
### เตรียมสอบ
[nopnithi](https://nopnithi.medium.com/%E0%B9%81%E0%B8%8A%E0%B8%A3%E0%B9%8C%E0%B8%9B%E0%B8%A3%E0%B8%B0%E0%B8%AA%E0%B8%9A%E0%B8%81%E0%B8%B2%E0%B8%A3%E0%B8%93%E0%B9%8C%E0%B9%80%E0%B8%95%E0%B8%A3%E0%B8%B5%E0%B8%A2%E0%B8%A1%E0%B8%AA%E0%B8%AD%E0%B8%9A-cka-%E0%B9%81%E0%B8%A5%E0%B8%B0-ckad-kubernetes-certification-6e4575de8320)
[openlandscape](https://blog.openlandscape.cloud/certified_kubernetes_administrator)

#### Kong and Konga
[wisdomgoody](https://wisdomgoody.medium.com/%E0%B8%AA%E0%B8%AD%E0%B8%99%E0%B8%95%E0%B8%B4%E0%B8%94%E0%B8%95%E0%B8%B1%E0%B9%89%E0%B8%87-kong-api-gateway-with-kubernetes-k8s-and-nginx-ingress-konga-on-google-cloud-gke-d94cc6b2e965)
https://blog.unixdev.co.th/install-kong-and-konga-on-centos7/
https://www.programmerall.com/article/40022225267/
#### Flask
https://github.com/bestspang/kubernetes-the-hard-way
https://medium.com/@towfeeqpandith/dockerizing-python-flask-application-and-deploying-on-kubernetes-a33de0614f67
https://medium.com/swlh/deploying-flask-app-with-kubernetes-aba8166e5f2c

### Reverse Proxy
[Reverse proxy](https://linuxize.com/post/nginx-reverse-proxy/)
[Avoid nginx decoding query parameters on proxy_pass](https://stackoverflow.com/questions/20496963/avoid-nginx-decoding-query-parameters-on-proxy-pass-equivalent-to-allowencodeds)
[How can query string parameters be forwarded through a proxy_pass](https://stackoverflow.com/questions/8130692/how-can-query-string-parameters-be-forwarded-through-a-proxy-pass-with-nginx)
[How to remove the path with an nginx proxy_pass](https://serverfault.com/questions/562756/how-to-remove-the-path-with-an-nginx-proxy-pass)
https://serverfault.com/questions/598202/make-nginx-to-pass-hostname-of-the-upstream-when-reverseproxying


### Docker
[Install Docker](https://github.com/bestspang/howtoKUBE/blob/main/README.md#faq)

## More Kong related stuff
- [**Kong Admin proxy**](https://github.com/pantsel/kong-admin-proxy)
- [**Kong Middleman plugin**](https://github.com/pantsel/kong-middleman-plugin)

## Author

Best Suriyawanakul

Documentation is available at [dayone.fun](https://www.dayone.fun/).
