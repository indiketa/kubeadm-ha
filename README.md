# kubeadm-ha

The intention of this manual is to describe the steps I followed to create an on-premise kubernetes cluster:

+ kubernetes 1.9 (using kubeadm)
+ centOS 7
+ etcd outside k8s
+ ha-proxy master load balancer

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


