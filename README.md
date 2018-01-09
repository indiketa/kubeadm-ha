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
firewall-cmd --permanent --zone=public --add-port=10250/tcp
firewall-cmd --zone=public --add-port=10250/tcp
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

Docker install (in production environments configure docker to use lvm volume directly, check [this](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#prerequisites))
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
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.d/99-sysctl.conf
sysctl -p /etc/sysctl.conf
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


## Dashboard UI + Heapster + Grafana
Dashboard installation. Run on a master node (with kubectl configured)
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

We want to execute Heapster & Grafana in master nodes. We need to modify influxDb definitions adding tolerations to permit this (we need to donwload and modify specs):

```
yum install -y wget unzip
wget https://github.com/kubernetes/heapster/archive/master.zip
unzip master.zip
cd heapster-master/deploy/kube-config/influxdb
```

Edit all yaml files (grafana.yaml, heapster.yaml, influxdb.yaml) inserting a toleration to allow execute on master nodes:

<pre>
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: monitoring-grafana
  namespace: kube-system
spec:
  replicas: 1
  template:  
    metadata:
      labels:
        task: monitoring
        k8s-app: grafana
    spec:
<b>      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"  
        effect: "NoSchedule" </b>
      containers:
      - name: grafana
        image: k8s.gcr.io/heapster-grafana-amd64:v4.4.3
        ports:
        - containerPort: 3000
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ca-certificates
</pre>

And then create RBAC auth, and heapster, grafana & influxdb services from your edited files:
```
kubectl create -f heapster-master/deploy/kube-config/rbac/heapster-rbac.yaml
kubectl create -f heapster-master/deploy/kube-config/influxdb/
```

Create a user for accessing dashboard-ui:
```
cat <<EOF > master_account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF

kubectl apply -f master_account.yaml
```

Get a bearer token to login to dashboard-ui:
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
(copy the token)
```

Open a proxy from your desktop:
```
kubectl proxy
```

Access [dashboard-ui](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy) and paste the token obtained above.

Now, in order to CANAL (flanneld) to work correctly i had to specify which network interface flanneld should use. This is done through config maps. Select kube-system namespace, then config maps. Open and edit canal-config. Find "canal_iface" key on data field and put your interface name, in my case "tun0".

<pre>
\\\"Backend\\\": {\\n    \\\"Type\\\": \\\"vxlan\\\"\\n  }\\n}\\n\"},\"kind\":\"ConfigMap\",\"metadata\":{\"annotations\":{},\"name\":\"canal-config\",\"namespace\":\"kube-system\"}}\n"
    }
  },
  "data": {
    <b>"canal_iface": "tun0",</b>
    "cni_network_config": "{\n    \"name\": \"k8s-pod-network\",\n    \"cniVersion\": \"0.3.0\",\n    \"plugins\": [\n        {\n            \"type\": \"calico\",\n            \"log_level\": \"info\",\n            \"datastore_type\": \"kubernetes\",\n            \"nodename\": \"__KUBERNETES_NODE_NAME__\",\n            \"ipam\": {\n                \"type\": \"host-local\",\n                \"subnet\": \"usePodC
</pre>


## Set up other master nodes
First install docker, kubelet, kubeadm as shown on first master node chapter and remember to turn swap off. Create /etc/kubernetes/pki and copy all files from first master to this directory:

```
mkdir -p /etc/kubernetes/pki
scp -r root@10.11.12.3:/etc/kubernetes/pki /etc/kubernetes
```

Copy kubeadm config file too:
```
scp -r root@10.11.12.3:kube_adm.yaml .
```

Init master node:
```
kubeadm init --config kube_adm.yaml
```
> Ensure that kubeadm finds the certificates and uses them in this master instance:
```
...
[preflight] Starting the kubelet service
[certificates] Using the existing ca certificate and key.
[certificates] Using the existing apiserver certificate and key.
[certificates] Using the existing apiserver-kubelet-client certificate and key.
[certificates] Using the existing sa key.
[certificates] Using the existing front-proxy-ca certificate and key.
[certificates] Using the existing front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
...
```

Repeat this process for each master node (2 nodes in this example) and check that all is running:
```
kubectl get nodes
```

## Setup worker node
First install docker, kubelet, kubeadm as shown on first master node chapter and remember to turn swap off. Create services for kubelet and docker. And then join to cluster with the command we obtained previously with master nodes (notice we are registering with load balancer ip):

```
kubeadm join --token 9aeb42.99b7540a5833866a 10.11.12.9:6443 --discovery-token-ca-cert-hash sha256:5ea4d794cbd1285d486a70bb37d2d6250908fa33cbfc4103b2d1f957ca204476
```

If join command was lost, you can retrieve the token with:
```
kubeadm token list
```

On a master node to obtain token value. You can regenerate CA sha256 digest with:
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

With this 2 values, you can regenerate the join command:
```
kubeadm join --token <token from kubeadm token list> --discovery-token-ca-cert-hash sha256:<value from openssl command> 10.11.12.9:6443 
```

Label this node (from master):
```
kubectl label node node.name.com name=piolin
```

## Enable log rotation on all nodes

Clear container logs
```
cat << EOF > /etc/logrotate.d/containers
/var/lib/docker/containers/*/*-json.log {
    rotate 5
    copytruncate
    missingok
    notifempty
    compress
    maxsize 10M
    daily
    create 0644 root root
}
EOF
```
