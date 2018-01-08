# kubeadm-ha

The intention of this manual is to describe the steps I followed to create an on-premise kubernetes cluster:

+ kubernetes 1.9 (using kubeadm)
+ centOS 7
+ etcd outside k8s
+ ha-proxy master load balancer

SELinux disabled in all hosts.

Network diagram:

```
                 +--------------+
                 | NODE 1       | k8s
                 | 10.11.12.2   |
                 +--------------+
                        |
                        |
                +-------v--------+
                | LOAD BALANCER  | ha-proxy
                | 10.11.12.9     |
                +----------------+
                        |
          +---------------------------+
          |                           |
   +------v------+             +------v-------+
   | MASTER 1    | k8s         | MASTER 2     | k8s
   | 10.11.12.3  | etcd        | 10.11.12.4   | etcd
   +-------------+             +--------------+
```

## Load balancer
Download and compile ha-proxy 1.8.3

```
yum install gcc pcre-static pcre-devel -y
wget http://www.haproxy.org/download/1.8/src/haproxy-1.8.3.tar.gz -O ~/haproxy.tar.gz
tar xzvf ~/haproxy.tar.gz -C ~/
cd ~/haproxy-1.8.3
make TARGET=linux2628
make install
mkdir -p /etc/haproxy
mkdir -p /var/lib/haproxy 
touch /var/lib/haproxy/stats
ln -s /usr/local/sbin/haproxy /usr/sbin/haproxy
cp ./examples/haproxy.init /etc/init.d/haproxy
chmod 755 /etc/init.d/haproxy
systemctl daemon-reload
systemctl enable haproxy
useradd -r haproxy
```

Open kubernetes API port
```
firewall-cmd --permanent --zone=public --add-port=6443/tcp
firewall-cmd --zone=public --add-port=6443/tcp
```

Open HAProxy status port (optional)
```
firewall-cmd --permanent --zone=public --add-port=8080/tcp
firewall-cmd --zone=public --add-port=8080/tcp
```


Configure and start HAProxy
```
cat << EOF > /etc/haproxy/haproxy.cfg
global
   log /dev/log local0
   log /dev/log local1 notice
   chroot /var/lib/haproxy
   stats timeout 30s
   user haproxy
   group haproxy
   daemon

defaults
    mode                    tcp
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

frontend http_stats
   bind *:8080
   mode http
   stats uri /haproxy?stats

frontend haproxy_kube
    bind *:6443
    mode tcp
    option tcplog
    timeout client  10800s
    default_backend masters

backend masters
    mode tcp
    option tcplog
    balance leastconn
    timeout server  10800s
    server master1 10.11.12.3:6443
    server master2 10.11.12.4:6443
EOF

systemctl start haproxy
```

Check  (if you opened port 8080) to get HAProxy stats:
http://10.11.12.9:8080/haproxy?stats


## etcd cluster preparation

On master 1:
```
yum -y install etcd

cat <<EOF > /etc/etcd/etcd.conf
# [member]
ETCD_NAME=etcd1
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.11.12.3:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.11.12.3:2379,http://127.0.0.1:2379"
# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.11.12.3:2380"
ETCD_INITIAL_CLUSTER="etcd1=http://10.11.12.3:2380,etcd2=http://10.11.12.4:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="KUBE-ETCD-CLUSTER"
ETCD_ADVERTISE_CLIENT_URLS="http://10.11.12.3:2379"
EOF

systemctl enable etcd
systemctl start etcd
```

On master 2:
```
yum -y install etcd
cat <<EOF > /etc/etcd/etcd.conf
# [member]
ETCD_NAME=etcd2
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.11.12.4:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.11.12.4:2379,http://127.0.0.1:2379"
# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.11.12.4:2380"
ETCD_INITIAL_CLUSTER="etcd1=http://10.11.12.3:2380,etcd2=http://10.11.12.4:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="KUBE-ETCD-CLUSTER"
ETCD_ADVERTISE_CLIENT_URLS="http://10.11.12.4:2379"
EOF

systemctl enable etcd
systemctl start etcd
```
Now the ETCD cluster is ready. My recommendation is to use at least 3 etcd servers (and kuber masters) to ensure real HA.  I've changed master1, master2 network to permit all incoming traffic (kubernetes over vpn: pritunl)

```
firewall-cmd --permanent --zone=trusted --add-source=10.11.12.0/24
firewall-cmd --zone=trusted --add-source=10.11.12.0/24
firewall-cmd --zone=trusted --add-interface=tun0 --permanent
```

## First kubernetes master installation

Docker install (in production environments configure docker to use lvm volume directly)
```
yum install docker
systemctl enable docker
systemctl start docker
```

Kubectl installation
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv ./kubectl /usr/local/bin/kubectl
```

Kubelet installation
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet
systemctl enable kubelet
```

Kubeadm installation
```
yum install -y kubeadm
```

As of kubernetes 1.9 swap space must be deactivated. Turn swap space off and remove from fstab:
```
swapoff -a
vi /etc/fstab
```

Send bridged packets to iptables:
```
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

Create kube_adm.yaml configuration file for kubeadm
> Note we are including the load balancer ip and domain name (master.yourdomain.com). The advertiseAddress is set to load balancer.

```
cat <<EOF > kube_adm.yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
etcd:
  endpoints:
  - "http://10.11.12.3:2379"
  - "http://10.11.12.4:2379"
apiServerCertSANs:
- "10.11.12.3"
- "10.11.12.4"
- "10.11.12.9"
- "127.0.0.1"
- "master.yourdomain.com"
token: "9aeb42.99b7540a5833866a"
tokenTTL: "0s"
api:
  advertiseAddress: "10.11.12.9"
networking:
  podSubnet: 10.244.0.0/16
EOF
```

Initialize the first cluster master:
```
kubeadm init --config kube_adm.yaml
```
First node started successfully. Save the join command, it will be used later. In case of loose you can retrieve this tokens directly into etcd cluster.

> Troubleshooting: If could not determine external etcd version error is thrown, ensure that etcd service is started.

Configure kubectl to use credentials to this cluster 
> **Be careful!** This command will remove access to other clusters
```
rm -rf $HOME/.kube
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
Apply this configuration to all devices (your desktop) that need root-access to cluster. Try with:

```
kubectl get nodes
```
NotReady status will be shown. CNI is missing.

Installation of CNI (we used [Canal](https://github.com/projectcalico/canal): a mix of flannel & calico).
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/canal/master/k8s-install/1.7/rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/projectcalico/canal/master/k8s-install/1.7/canal.yaml
```

Retry node status, the node is Ready.
```
kubectl get nodes
```
First kubernetes master is completed. Prior installing things like dashboard, heapster, etc... I recommend to setup the other master nodes.

## Other master deployment

First install: docker, kubelet, kubeadm as shown on previous chapter.
