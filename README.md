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
yum install -y postgresql10-contrib
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
# su postgres

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
curl $(rpm --eval "https://download.konghq.com/gateway-2.x-centos-%{centos_ver}/config.repo") | sudo tee /etc/yum.repos.d/kong.repo

sudo yum install kong-2.7.1
```
or
```
curl -Lo kong-2.7.1.rpm $(rpm --eval "https://download.konghq.com/gateway-2.x-centos-%{centos_ver}/Packages/k/kong-2.7.1.el%{centos_ver}.amd64.rpm")

sudo yum install kong-2.7.1.rpm
```
* Prepare config file
```
# cp /etc/kong/kong.conf.default /etc/kong/kong.conf
# vi /etc/kong/kong.conf
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
_or add ssl_
```
docker run -p 1337:1337 \
             --network {{kong-network}} \ // optional
             --name konga \
             -e "NODE_ENV=production" \ // or "development" | defaults to 'development'
             -e "TOKEN_SECRET={{somerandomstring}}" \
             pantsel/konga
```
* Connect Konga to Master node

```
using internal IP of internal IP
x.x.x.x:8001
```




### Setup Nginx

https://phoenixnap.com/kb/how-to-install-nginx-on-centos-7
https://serverspace.io/support/help/install-and-configure-nginx-on-centos-7/
https://www.cyberciti.biz/faq/how-to-install-and-use-nginx-on-centos-7-rhel-7/

* add the CentOS 7 EPEL repository
```
sudo yum install epel-release
```
* Create the file named /etc/yum.repos.d/nginx.repo
```
sudo vi /etc/yum.repos.d/nginx.repo
```
* Append following for CentOS 7.x:
```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/
gpgcheck=0
enabled=1
```
or Append following for RHEL (Red Hat Enterprise Linux) version 7.x
```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/mainline/rhel/7/$basearch/
gpgcheck=0
enabled=1
```

* Install nginx package
```
sudo yum install nginx
```

* Not sure
```
vi /etc/nginx/nginx.conf
# listen [::]:80 default_server;
nginx -t
```

* Start/stop/restart nginx server
```
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```
command:
```
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl status nginx

```
* Open port 80 and 443 using firewall-cmd
```
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload
```

* Verify that port 80 or 443 opened using ss command:
```
sudo ss -tulpn
```

* If you do not know your server IP address run the following
```
ip a
or
ip addr
```


* Configure Nginx server

- Config dir – /etc/nginx/
- Master/Global config file – /etc/nginx/nginx.conf
- Port 80 http config file – /etc/nginx/conf.d/default
- TCP ports opened by Nginx – 80 (HTTP), 443 (HTTPS)
- Document root directory – /usr/share/nginx/html

```
vi /etc/nginx/conf.d/default.conf
sudo nginx -t
systemctl restart nginx
```

```
#upstream BackendSever {
#    server 10.148.0.7;
#}

#upstream web {
#  ip_hash;
#  server 10.148.0.7:8000;
#}

server {
    listen       0.0.0.0:80;
    server_name  localhost;

    listen 443 ssl;

    #ssl on;
    ssl_certificate agilis-platform.com.pem;
    ssl_certificate_key agilis-platform.com.key;

    client_max_body_size 10M;
    autoindex off;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    location /kubex {
        proxy_pass http://127.0.0.1:8000; #30191
        proxy_http_version  1.1;
        #proxy_set_header Upgrade $http_upgrade;
        #proxy_set_header Connection 'upgrade';
        #proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        #proxy_redirect off;
        #try_files $uri $uri/ =404;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #
  }
```

* Make SSL
```
vi /etc/nginx/dayone.fun.pem
vi /etc/nginx/dayone.fun.key

sudo nginx -t
systemctl restart nginx
```

## Install Helm Kubernetes
* Installing [here](https://www.cyberithub.com/steps-to-install-helm-kubernetes-package-manager-on-linux/)
check release https://github.com/helm/helm/releases
```
wget https://get.helm.sh/helm-v3.8.3-linux-amd64.tar.gz
tar -xvf helm-v3.6.3-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm repo add stable https://charts.helm.sh/stable
```
command:
```
helm list
helm3 del <release-name> --namespace <namespace>
helm repo update

```
## Prometheus and Grafana
```
kubectl create namespace prometheus

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack --version 33.1.0 --namespace prometheus
```

*****************************************************************************************
## Tips
_ก่อนเริ่มใส่คำสั่ง shortcut_
```
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
source ~/.bashrc
```

### Short key
> **Note**: คำสั่งย่อต่างๆ

**Args**

shortnames  | description | shortnames | description
----------|-------------|---------|---------
cs      | componentstatuses | sa |  serviceaccounts
cm      | configmaps        | svc |  services
ev      | events            | crd, crds |  customresourcedefinitions
ep      | endpoints         | ds |  daemonsets
limits  | limitranges       | deploy |  deployments
ns      | namespaces        | rs |  replicasets
no      | nodes             | sts |  statefulsets
pvc     | persistentvolumeclaims | hpa |  horizontalpodautoscalers
pv      | persistentvolumes | cj |  cronjobs
po      | pods              | cr,crs |  certificiaterequests
rc      | replicationcontrollers | cert,certs |  certificates
ing     | ingresses         | csr |  certificatesigningrequests
netpol  | networkpolicies   | rs |  replicasets
ss      | scheduledscalers  | pc |  priorityclasses
sc      | storageclasses    | quota |  resourcequotas
psp     | podsecuritypolicies |  -  |  -

*****************************************************************************************

## FAQ
Clear ssh key:
```
ssh-keygen -R 10.101.108.101
```
```
cat bintray-kong-kong-community-edition-rpm.repo \
[bintraybintray-kong-kong-community-edition-rpm] \
name=bintray--kong-kong-community-edition-rpm \
baseurl=https://kong.bintray.com/kong-community-edition-rpm/centos/7
gpgcheck=0
repo_gpgcheck=0
enabled=1
```

check firewall:

[for more](https://www.liquidweb.com/kb/how-to-stop-and-disable-firewalld-on-centos-7/)
```
systemctl status firewalld
sudo firewall-cmd --zone=work --list-all
```
Can't start firewall:
```
pkill -f firewalld

firewall-cmd --state
systemctl start firewalld
firewall-cmd --state
```

Docker Got permission denied while trying to connect to the Docker:
https://stackoverflow.com/questions/47854463/docker-got-permission-denied-while-trying-to-connect-to-the-docker-daemon-socke
```
or 664
sudo chmod 666 /var/run/docker.sock
```

M1:
```
docker build --platform linux/amd64 -t lakhansamani/docker-demo .
```

How to Fix "Read-only file system" error:
```
sudo fsck -n -f

reboot
```
or
```
# this  working
sudo mount -o rw,remount /
```
https://askubuntu.com/questions/287021/how-to-fix-read-only-file-system-error-when-i-run-something-as-sudo-and-try-to
or
```
# not work
sudo chmod -R 755 yourfoldername
```
https://superuser.com/questions/591108/how-can-i-restore-the-permissions-of-my-usr-folder

Upgrade Kernel:
```
grub2-mkconfig -o /boot/grub2/grub.cfg
or
mv /boot/grub/grub.conf /boot/grub/bk_grub.conf
yum -y update && yum -y reinstall kernel
```

grubby fatal error: unable to find a suitable template:
```
grub2-mkconfig -o /boot/grub2/grub.cfg
```


Search:
```
find / -name file.xxx
```
Remove Nginx:
https://otodiginet.com/software/how-to-unistall-nginx-from-centos-7/

The connection to the server <host> was refused:
```
# not working
sudo -i
swapoff -a
exit
strace -eopenat kubectl version
```
https://discuss.kubernetes.io/t/the-connection-to-the-server-host-6443-was-refused-did-you-specify-the-right-host-or-port/552
https://discuss.kubernetes.io/t/the-connection-to-the-server-localhost-8080-was-refused-did-you-specify-the-right-host-or-port/1464


Nginx:
```
service nginx restart

vi /etc/nginx/conf.d/default.conf
sudo nginx -t
systemctl restart nginx
tail -30 /var/log/nginx/error.log
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

*****************************************************************************************

## Documentation
### Nginx
https://otodiginet.com/software/how-to-unistall-nginx-from-centos-7/
https://www.cyberciti.biz/faq/how-to-install-and-use-nginx-on-centos-7-rhel-7/
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
Proxy:
https://docs.docker.com/config/daemon/systemd/

### More Kong related stuff
- [**Kong Admin proxy**](https://github.com/pantsel/kong-admin-proxy)
- [**Kong Middleman plugin**](https://github.com/pantsel/kong-middleman-plugin)

*****************************************************************************************

## Author

Best Suriyawanakul

Documentation is available at [dayone.fun](https://www.dayone.fun/).
