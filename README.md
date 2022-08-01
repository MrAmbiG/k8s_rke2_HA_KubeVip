# k8s_rke2_HA_KubeVip
Sample implementation of k8s with rke2, HA with floating ip from kube-vip
# Overview of the k8s Cluster
- k8s cluster with multiple master nodes always in odd numbers
- floating ip is used to register all master nodes
- same floating ip is used to communicate with the k8s cluster
- Install nfs-client (nfs-common) on all nodes if using NFS storage

### common tasks on all nodes
Perform all these tasks on all master nodes before anything else.
1. `sudo su -` # switch to root user
2. `cd /home/<useraccount>` # user account is usually k8sprod or master, wherever you have copied the config.yaml
3. `apt-get remove docker docker-engine docker.io containerd runc -y` # remove all docker, containerd
4. `echo 'export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock' >> ~/.bashrc ; echo 'export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock' >> ~/.bashrc ; echo 'export PATH=${PATH}:/var/lib/rancher/rke2/bin' >> ~/.bashrc ; echo 'alias kl=kubectl' >> ~/.bashrc ; source ~/.bashrc ;`

# FQDN
- sudo hostnamectl set-hostname <name>
- /etc/hosts should reflect all master nodes including kube-vip name if FQDN is not implemented via DNS. 
- /etc/hostname should reflect the full name
- set `preserve_hostname: true` at /etc/cloud/cloud.cfg. `sed -i 's/preserve_hostname: false/preserve_hostname: true/g' /etc/cloud/cloud.cfg`

*sample /etc/hosts file entries* if DNS is not used for FQDN <br>
10.1.1.1 node1.ambi.com node1 # master1 <br>
10.1.1.2 node2.ambi.com node2 # master2 <br>
10.1.1.3 node3.ambi.com node3 # master3 <br>
10.1.1.4 myk8s.ambi.com myk8s # k8s master api - kube-vip <br>
<br>

# common tasks on all master nodes
Perform all these tasks on all master nodes before anything else.
1. sudo su - # switch to root user
2. cd /home/<useraccound> # user account is usually k8sprod or master, wherever you have copied the config.yaml
3. mkdir -p /etc/rancher/rke2
4. ensure the names of the primary network interface of the master nodes is the same. it cannot eth0 on master1 and eth1 or eno1 on master2 or master3

# private insecure registry
_Be careful with the registries.yaml file. The official documentation as of today july2022 is misleading. If you want your private insecure registry to work, then mimic the file below_
```
# registries.yaml    
mirrors:
  "my-artifactory.company.com:5000":
    endpoint:
      - "http://my-artifactory.company.com:5000"
configs:
  "my-artifactory.company.com:5000":
    auth:
      username: xxxxxxxxx # this is the registry username
      password: yyyyyyyyy # this is the registry
```

# master1 setup
*config.master1.yaml*
```
tls-san:
  - node1
  - node1.amd.com
  - myk8s.amd.com
  - 10.1.1.4 # kube-vip
disable: rke2-ingress-nginx
```

```
sudo su -
mkdir -p /var/lib/rancher/rke2/server/manifests/
export RKE2_VIP_IP=10.1.1.4 # IMPORTANT: Update this with the IP that you chose.
export INTERFACE=eth1 # The name of the interface on all master nodes should be the same
export TAG="v0.3.8"
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/k3s/containerd/containerd.sock
export CONTAINERD_ADDRESS=/run/k3s/containerd/containerd.sock
export PATH=/var/lib/rancher/rke2/bin:$PATH
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
alias kl=kubectl


# Install the kube-vip deployment into rke2's self-installing manifest folder
curl -s https://kube-vip.io/manifests/rbac.yaml > /var/lib/rancher/rke2/server/manifests/kube-vip-rbac.yaml
curl -sL kube-vip.io/k3s |  vipAddress=${RKE2_VIP_IP} vipInterface=$INTERFACE sh | sudo tee vip.yaml


# Find/Replace all k3s entries to represent rke2
sed -i 's/k3s/rke2/g' vip.yaml
cp vip.yaml /var/lib/rancher/rke2/server/manifests/vip.yaml

# create the rke2 config file
mkdir -p /etc/rancher/rke2
cp config.master1.yaml /etc/rancher/rke2/config.yaml # copy the node's config.yaml file
cp registries.yaml /etc/rancher/rke2/ # copy registry file
```

### update path with rke2-binaries
```
echo 'export KUBECONFIG=/etc/rancher/rke2/rke2.yaml' >> ~/.bashrc ; echo 'export PATH=${PATH}:/var/lib/rancher/rke2/bin' >> ~/.bashrc ; echo 'alias kl=kubectl' >> ~/.bashrc ; source ~/.bashrc ;
```

### install specific version of rke2
```
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.21.14+rke2r1 sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service
sleep 90 #wait ~90 seconds for rke2 to be ready
KUBECONFIG=/etc/rancher/rke2/rke2.yaml kubectl get nodes -o wide # this should work, if not, start over.
```

### test the api on kube-vip
```
mkdir -p $HOME/.kube
export RKE2_VIP_IP=devcloud-k8s.amd.com
sudo cat /etc/rancher/rke2/rke2.yaml | sed 's/127.0.0.1/'$RKE2_VIP_IP'/g' > $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
KUBECONFIG=~/.kube/config kubectl get nodes -o wide # this should work, if not, start over.
```

# master 2
*master2 config file*
```
token: Kasdfasdfasfawerfawdfsdgwrfasrdfwerwefrsdfwerwefrserver:aswdfasdfasdfc # /var/lib/rancher/rke2/server/token from master1
server: https://10.1.1.4:9345
tls-san:
  - node2
  - node2.ambi.com
  - myk8s.ambi.com
  - 10.1.1.4 # kube-vip
disable: rke2-ingress-nginx
```

```
sudo su -
mkdir -p /etc/rancher/rke2
cp config.master2.yaml /etc/rancher/rke2/config.yaml
cp registries.yaml /etc/rancher/rke2/ # copy registry file
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.21.14+rke2r1 sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service
sleep 90 #wait ~90 seconds for rke2 to be ready
```

# master 3
*master3 config file*
```
token: Kasdfasdfasfawerfawdfsdgwrfasrdfwerwefrsdfwerwefrserver:aswdfasdfasdfc # /var/lib/rancher/rke2/server/token from master1
server: https://10.1.1.4:9345
tls-san:
  - node3
  - node3.ambi.com
  - myk8s.ambi.com
  - 10.1.1.4 # kube-vip
disable: rke2-ingress-nginx
```

```
sudo su -
mkdir -p /etc/rancher/rke2
cp config.master2.yaml /etc/rancher/rke2/config.yaml
cp registries.yaml /etc/rancher/rke2/ # copy registry file
curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.21.14+rke2r1 sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service
sleep 90 #wait ~90 seconds for rke2 to be ready
```

# worker nodes
- follow the same for all worker nodes
*worker1.config.yaml*
```
token: K101ad4d209b9d453c2de43a7aed07ca8cbf4b6effa4c13cfedc1c7b054c4c4729a::server:e20bdc7a1789d576a1334fee0d65df6b # /var/lib/rancher/rke2/server/token from master1
server: https://10.1.1.4:9345
```

```
sudo su -
cd /home/<useraccound> # user account is usually k8sprod or master, wherever you have copied the config.yaml
mkdir -p /etc/rancher/rke2/
cp config.yaml /etc/rancher/rke2/config.yaml # their respective config.yaml
cp registries.yaml /etc/rancher/rke2/ # copy registry file
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" INSTALL_RKE2_VERSION=v1.21.14+rke2r1 sh -
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```
