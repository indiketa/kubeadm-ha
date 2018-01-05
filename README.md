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
                 |NODE 1        | k8s
                 |10.11.12.2    |
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
   | MASTER 1    | k8s         |MASTER 2      | k8s
   | 10.11.12.3  | etcd        |10.11.12.4    | etcd
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



