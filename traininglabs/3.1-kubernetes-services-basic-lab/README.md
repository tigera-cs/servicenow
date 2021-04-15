## 3.1. Kubernetes Services - Basic Lab

This is the 1st lab in a series of labs about k8s services. This lab explores the concepts related to k8s services and the drills down into the iptables chains that enable Kube Proxy to deliver the service.
In this lab, you will:
3.1.1. Examine kubernetes service
3.1.2. Explore ClusterIP iptables rules
3.1.3. Explore NodePort iptables rules

### 3.1.0. Before you begin

This lab builds on the setup in Lab1. If you haven't completed Lab1, please go back and ensure that your cluster is setup with k8s CNI and that Yaobank application is deployed.

### 3.1.1. Examine kubernetes services

We will run this lab from `worker1` so we can explore the iptables rules that kube-proxy has set up.

```
worker1
```

Let's take a look at the services and pods in the `yaobank` kubernetes namespace.

#### 3.1.1.1. List services

```
kubectl get svc -n yaobank
```
```
ubuntu@worker1:~/calico/lab-manifests$ kubectl get svc -n yaobank
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
customer   NodePort    10.49.209.8     <none>        80:30180/TCP   31m
database   ClusterIP   10.49.25.37     <none>        2379/TCP       31m
summary    ClusterIP   10.49.111.226   <none>        80/TCP         31m
```

We should have three services deployed. One `NodePort` service and two `ClusterIP` services.

#### 3.1.1.2. List service endpoints
Find the endpoints for each of the services.
```
kubectl get endpoints -n yaobank
```
```
ubuntu@worker1:~/calico/lab-manifests$ kubectl get endpoints -n yaobank
NAME       ENDPOINTS                      AGE
customer   10.46.0.225:80                 32m
database   10.46.1.114:2379               32m
summary    10.46.1.115:80,10.46.1.97:80   32m
```

#### 3.1.1.3 List pods
```
kubectl get pods -n yaobank -o wide
```
```
ubuntu@worker1:~/calico/lab-manifests$ kubectl get pods -n yaobank -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP            NODE      NOMINATED NODE   READINESS GATES
customer-746bdb9f6b-2srx5   1/1     Running   0          6m23s   10.46.0.225   worker1   <none>           <none>
database-64c799fdd7-h2mh7   1/1     Running   0          5m49s   10.46.1.114   worker2   <none>           <none>
summary-c75c4c64c-h7bdw     1/1     Running   0          5m47s   10.46.1.115   worker2   <none>           <none>
summary-c75c4c64c-nls7v     1/1     Running   0          5m3s    10.46.1.97    master    <none>           <none>
```
You can see that the IP addresses listed as the service endpoints in the previous step map to the backing pods, as expected. Each service is backed by one or more pods spread across the worker nodes in our cluster.

### 3.1.2. Explore ClusterIP iptables rules

Let's explore the iptables rules that implement the `summary` service.

#### 3.1.2.1. Get service endpoints
Find the service endpoints for `summary` `ClusterIP` service.
```
kubectl get endpoints -n yaobank summary
```
```
ubuntu@worker1:~/calico/lab-manifests$ kubectl get endpoints -n yaobank summary
NAME      ENDPOINTS                      AGE
summary   10.46.1.115:80,10.46.1.97:80   33m
```

The `summary` service has two endpoints (`10.49.130.128` on port `80` AND `10.49.130.67` on port `80` in this example output). Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to these endpoint IP addresses.

#### 3.1.2.2. Examine the KUBE-SERVICE chain
```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  *      *       0.0.0.0/0            10.49.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  *      *       0.0.0.0/0            10.49.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.183.70         /* calico-system/calico-node-metrics:calico-bgp-metrics-port cluster IP */ tcp dpt:9900
    0     0 KUBE-SVC-ZMPNACNGKBKCFXCW  tcp  --  *      *       0.0.0.0/0            10.49.183.70         /* calico-system/calico-node-metrics:calico-bgp-metrics-port cluster IP */ tcp dpt:9900
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.78.27          /* tigera-kibana/tigera-secure-kb-http:https cluster IP */ tcp dpt:5601
    0     0 KUBE-SVC-3XH7Z4WONXPKZ54M  tcp  --  *      *       0.0.0.0/0            10.49.78.27          /* tigera-kibana/tigera-secure-kb-http:https cluster IP */ tcp dpt:5601
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.57.60          /* calico-system/calico-kube-controllers-metrics:metrics-port cluster IP */ tcp dpt:9094
    0     0 KUBE-SVC-KQVGIOWQAVNMB2ZL  tcp  --  *      *       0.0.0.0/0            10.49.57.60          /* calico-system/calico-kube-controllers-metrics:metrics-port cluster IP */ tcp dpt:9094
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.249.146        /* tigera-manager/tigera-manager-nodeport-svc: cluster IP */ tcp dpt:9443
    0     0 KUBE-SVC-44KOMUFU34RACUOG  tcp  --  *      *       0.0.0.0/0            10.49.249.146        /* tigera-manager/tigera-manager-nodeport-svc: cluster IP */ tcp dpt:9443
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.25.37          /* yaobank/database:http cluster IP */ tcp dpt:2379
    0     0 KUBE-SVC-AE2X4VPDA5SRYCA6  tcp  --  *      *       0.0.0.0/0            10.49.25.37          /* yaobank/database:http cluster IP */ tcp dpt:2379
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  *      *       0.0.0.0/0            10.49.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.67.80          /* tigera-prometheus/calico-node-prometheus:web cluster IP */ tcp dpt:9090
    0     0 KUBE-SVC-SRS3TRYGFCUTQSH7  tcp  --  *      *       0.0.0.0/0            10.49.67.80          /* tigera-prometheus/calico-node-prometheus:web cluster IP */ tcp dpt:9090
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.183.70         /* calico-system/calico-node-metrics:calico-metrics-port cluster IP */ tcp dpt:9081
    0     0 KUBE-SVC-BPJNZGPODTH4UZQI  tcp  --  *      *       0.0.0.0/0            10.49.183.70         /* calico-system/calico-node-metrics:calico-metrics-port cluster IP */ tcp dpt:9081
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.245.202        /* tigera-elasticsearch/tigera-elasticsearch-metrics:metrics-port cluster IP */ tcp dpt:9081
    0     0 KUBE-SVC-YFF642K22PZPYCSR  tcp  --  *      *       0.0.0.0/0            10.49.245.202        /* tigera-elasticsearch/tigera-elasticsearch-metrics:metrics-port cluster IP */ tcp dpt:9081
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.117.28         /* tigera-compliance/compliance:compliance-api cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-VWTJDZUOIAKKPMUV  tcp  --  *      *       0.0.0.0/0            10.49.117.28         /* tigera-compliance/compliance:compliance-api cluster IP */ tcp dpt:443
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.167.38         /* tigera-manager/tigera-manager: cluster IP */ tcp dpt:9443
    0     0 KUBE-SVC-YIFKN7U6GLH6BT7Z  tcp  --  *      *       0.0.0.0/0            10.49.167.38         /* tigera-manager/tigera-manager: cluster IP */ tcp dpt:9443
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.111.226        /* yaobank/summary:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-OIQIZJVJK6E34BR4  tcp  --  *      *       0.0.0.0/0            10.49.111.226        /* yaobank/summary:http cluster IP */ tcp dpt:80
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.209.8          /* yaobank/customer:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-PX5FENG4GZJTCELT  tcp  --  *      *       0.0.0.0/0            10.49.209.8          /* yaobank/customer:http cluster IP */ tcp dpt:80
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.156.42         /* tigera-system/tigera-api:apiserver cluster IP */ tcp dpt:443
    0     0 KUBE-SVC-5YT3S4Q5ZQB7MXPI  tcp  --  *      *       0.0.0.0/0            10.49.156.42         /* tigera-system/tigera-api:apiserver cluster IP */ tcp dpt:443
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.255.234        /* tigera-elasticsearch/tigera-secure-es-http:https cluster IP */ tcp dpt:9200
  213 12780 KUBE-SVC-27GAF2D4QQPTKE7C  tcp  --  *      *       0.0.0.0/0            10.49.255.234        /* tigera-elasticsearch/tigera-secure-es-http:https cluster IP */ tcp dpt:9200
    0     0 KUBE-MARK-MASQ  udp  --  *      *      !10.48.0.0/16         10.49.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
  843  112K KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  *      *       0.0.0.0/0            10.49.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.91.82          /* tigera-prometheus/calico-node-alertmanager:web cluster IP */ tcp dpt:9093
    0     0 KUBE-SVC-P7GI5BPQ3GTNT5FK  tcp  --  *      *       0.0.0.0/0            10.49.91.82          /* tigera-prometheus/calico-node-alertmanager:web cluster IP */ tcp dpt:9093
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.149.199        /* calico-system/calico-typha:calico-typha cluster IP */ tcp dpt:5473
    0     0 KUBE-SVC-RK657RLKDNVNU64O  tcp  --  *      *       0.0.0.0/0            10.49.149.199        /* calico-system/calico-typha:calico-typha cluster IP */ tcp dpt:5473
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.156.42         /* tigera-system/tigera-api:queryserver cluster IP */ tcp dpt:8080
    0     0 KUBE-SVC-BXX6NV5PBDEKW23Y  tcp  --  *      *       0.0.0.0/0            10.49.156.42         /* tigera-system/tigera-api:queryserver cluster IP */ tcp dpt:8080
  176 10560 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

Each iptables chain consists of a list of rules that are executed in order until a rule matches. The key columns/elements to note in this output are:
* `target` - which chain iptables will jump to if the rule matches
* `prot` - the protocol match criteria
* `source`, and `destination` - the source and destination IP address match criteria
* the comments that kube-proxy inculdes
* the additional match criteria at the end of each rule - e.g `dpt:80` that specifies the destination port match

#### 3.1.2.3. KUBE-SERVICES -> KUBE-SVC-XXXXXXXXXXXXXXXX

Now let's look more closely at the rules for the `summary` service.
```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep -E summary
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep -E summary
    0     0 KUBE-MARK-MASQ  tcp  --  *      *      !10.48.0.0/16         10.49.111.226        /* yaobank/summary:http cluster IP */ tcp dpt:80
    0     0 KUBE-SVC-OIQIZJVJK6E34BR4  tcp  --  *      *       0.0.0.0/0            10.49.111.226        /* yaobank/summary:http cluster IP */ tcp dpt:80
```

The second rule directs traffic destined for the summary service clusterIP (`10.49.163.205` in the example output) to the chain that load balances the service (KUBE-SVC-XXXXXXXXXXXXXXXX).

#### 3.1.2.4. KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX

`kube-proxy` in `iptables` mode uses a randomized equal cost selection algorithm to load balance traffic between pods.  We currently have two `summary` pods.

Let's examine how this loadbalancing works. (Remember your chain name may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SVC-OIQIZJVJK6E34BR4
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SVC-OIQIZJVJK6E34BR4
Chain KUBE-SVC-OIQIZJVJK6E34BR4 (1 references)
    0     0 KUBE-SEP-NOAT3GMKE6BDT7P4  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/summary:http */ statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-PDEV4VWRXC4LHR3U  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/summary:http */
```

Notice that `kube-proxy` is using the `iptables` `statistic` module to set the probability for a packet to be randomly matched.  Make sure you scroll all the way to the right to see the full output.

The first rule directs traffic destined for the `summary` service to the chain that delivers packets to the first service endpoint (KUBE-SEP-QBQ2ZLCSJLFEASMX) with a probability of 0.50000000000. The second rule unconditionally directs to the second service endpoint chain (KUBE-SEP-BS2BXDAXX5ZBJNP2). The result is that traffic is load balanced across the service endpoints equally (on average).

If there were 3 service endpoints then the first chain matches would be probability 0.33333333, the second probability 0.5, and the last unconditional. The result is each service endpoint receives a third of the traffic (on average).

#### 3.1.2.5. KUBE-SEP-XXXXXXXXXXXXXXXX -> `summary` pod
Let's look at one of the service endpoint chains. (Remember your chain names may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SEP-NOAT3GMKE6BDT7P4
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SEP-NOAT3GMKE6BDT7P4
Chain KUBE-SEP-NOAT3GMKE6BDT7P4 (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.46.1.115          0.0.0.0/0            /* yaobank/summary:http */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/summary:http */ tcp to:10.46.1.115:80
```

The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (`10.48.0.65` in this example). After this, standard Linux routing can handle forwarding the packet like it would for any other packet.

### 3.1.2.6. Recap
You've just traced the kube-proxy iptables rules used to load balance traffic to `summary` pods exposed as a service of type `ClusterIP`.

In summary, for a packet being sent to a clusterIP:
* The KUBE-SERVICES chain matches on the clusterIP and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

### 3.1.3. Explore NodePort iptables rules

Let's explore the iptables rules that implement the `customer` service.

#### 3.1.3.1. Get service endpoints
Find the service endpoints for `customer` `NodePort` service.
```
kubectl get endpoints -n yaobank customer
```
```
ubuntu@worker1:~$ kubectl get endpoints -n yaobank customer
NAME       ENDPOINTS        AGE
customer   10.46.0.225:80   37m
```

The `customer` service has one endpoint (`10.49.2.65` on port `80` in this example output). Starting from the `KUBE-SERVICES` iptables chain, we will traverse each chain until you get to the rule directing traffic to this endpoint IP address.

### 3.1.3.2. KUBE-SERVICES -> KUBE-NODEPORTS
The `KUBE-SERVICE` chain handles the matching for service types `ClusterIP` and `LoadBalancer`. At the end of `KUBE-SERVICE` chain, another custom chain `KUBE-NODEPORTS` will handle traffic for service type `NodePort`.
```
sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep KUBE-NODEPORTS
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SERVICES | grep KUBE-NODEPORTS
323 19380 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL
```

`match dst-type LOCAL` matches any packet with a local host IP as the destination. I.e. any address that is assigned to one of the host's interfaces.

### 3.1.3.3. KUBE-NODEPORTS -> KUBE-SVC-XXXXXXXXXXXXXXXX
```
sudo iptables -v --numeric --table nat --list KUBE-NODEPORTS
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-NODEPORTS
Chain KUBE-NODEPORTS (1 references)
    0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */ tcp dpt:30180
    0     0 KUBE-SVC-PX5FENG4GZJTCELT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */ tcp dpt:30180
```

The second rule directs traffic destined for the `customer` service to the chain that load balances the service (KUBE-SVC-PX5FENG4GZJTCELT). `tcp dpt:30180` matches any packet with the destination port of tcp 30180 (the node port of the `customer` service).

#### 3.1.3.4. KUBE-SVC-XXXXXXXXXXXXXXXX -> KUBE-SEP-XXXXXXXXXXXXXXXX
(Remember your chain name may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SVC-PX5FENG4GZJTCELT
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SVC-PX5FENG4GZJTCELT
Chain KUBE-SVC-PX5FENG4GZJTCELT (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-SEP-E6S7I7GXEBEAGHWS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */
```

As we only have a single backing pod for the `customer` service, there is no loadbalancing to do, so there is a single rule that directs all traffic to the chain that delivers the packet to the service endpoint (KUBE-SEP-XXXXXXXXXXXXXXXX).

### 3.1.3.5. KUBE-SEP-XXXXXXXXXXXXXXXX -> `customer` endpoint
(Remember your chain name may be different than this example.)
```
sudo iptables -v --numeric --table nat --list KUBE-SEP-E6S7I7GXEBEAGHWS
```
```
ubuntu@worker1:~$ sudo iptables -v --numeric --table nat --list KUBE-SEP-DES65NIG7LUKIP6J
Chain KUBE-SEP-E6S7I7GXEBEAGHWS (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 KUBE-MARK-MASQ  all  --  *      *       10.46.0.225          0.0.0.0/0            /* yaobank/customer:http */
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* yaobank/customer:http */ tcp to:10.46.0.225:80
```

This rule delivers the packet to the `customer` service endpoint.

The second rule performs the DNAT that changes the destination IP from the service's clusterIP to the IP address of the service endpoint backing pod (`10.49.2.65` in this example). After this, standard Linux routing can handle forwarding the packet like it would for any other packet.

### 3.1.3.6. Recap
You've just traced the kube-proxy iptables rules used to load balance traffic to `customer` pods exposed as a service of type `NodePort`.

In summary, for a packet being sent to a NodePort:
* The end of the KUBE-SERVICES chain jumps to the KUBE-NODEPORTS chain
* The KUBE-NODEPORTS chaing matches on the NodePort and jumps to the corresponding KUBE-SVC-XXXXXXXXXXXXXXXX chain.
* The KUBE-SVC-XXXXXXXXXXXXXXXX chain load balances the packet to a random service endpoint KUBE-SEP-XXXXXXXXXXXXXXXX chain.
* The KUBE-SEP-XXXXXXXXXXXXXXXX chain DNATs the packet so it will get routed to the service endpoint (backing pod).

In the end make sure you exit from the worker node and come back to master node.

```
exit
```

> __Congratulations! You have completed this la and you have by now a basic understanding of kubernetes services.__
