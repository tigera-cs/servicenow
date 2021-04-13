
# Install Calico in IPVS mode


Calico has support for kube-proxy’s ipvs proxy mode. Calico ipvs support is activated automatically if Calico detects that kube-proxy is running in that mode.

ipvs mode provides greater scale and performance vs iptables mode. However, it comes with some limitations. In IPVS mode
– Its a good kick-start activity to understand Calico Enterprise capabilities.

## Requirements

1. A cluster running Kubernetes v1.11+
2. Load the below required kernel modules and install `ipvsadm` and `ipset` on all nodes.

```
sudo apt install -y ipvsadm ipset

# load module <module_name>
sudo modprobe ip_vs 
sudo modprobe ip_vs_rr
sudo modprobe ip_vs_wrr 
sudo modprobe ip_vs_sh
sudo modprobe nf_conntrack
sudo sysctl --system
sudo sysctl -p

# to check loaded modules, use
lsmod | grep -e ip_vs -e nf_conntrack
# or
cut -f1 -d " " /proc/modules | grep -e ip_vs -e nf_conntrack
```


## Steps to enable IPVS mode 

1. Change the configMap of kube-proxy, modify "mode" from "" to "ipvs"

```
kubectl -n kube-system edit cm kube-proxy
```

2. Delete all the active proxy pods

```
kubectl get pods -n kube-system

kubectl delete pod --namespace=kube-system kube-proxy-<name>
```

3. Check the logs of new kube-proxy pods

```
kubectl -n kube-system logs kube-proxy-<name>

For easy understanding
kubectl logs kube-proxy-<name> -n kube-system | grep "Using ipvs Proxier"
```
If you are able to find the mentioned String in the logs, IPVS mode is being used by the cluster. You can always see detailed logs for more depth about the IPVS mode.

## Verify and Debug IPVS

Users can use ipvsadm tool to check whether kube-proxy are maintaining IPVS rules correctly. For example, we have the following services in the cluster.

```
root@ip-10-0-0-10:~# kubectl get svc -A
NAMESPACE     NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                  23h
kube-system   kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   23h

``` 


Following are the IPVS proxy rules for above services

```
root@ip-10-0-0-10:~# sudo ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.1:443 rr
  -> 10.0.0.10:6443               Masq    1      0          0
TCP  10.96.0.10:53 rr
TCP  10.96.0.10:9153 rr
UDP  10.96.0.10:53 rr


```

## Why kube-proxy can't start IPVS mode
Use the following check list to help you solve the problems:

1. Specify `mode=ipvs` 

Check whether the kube-proxy mode has been set to ipvs in the `kube-proxy` configmap.

3. Install required kernel modules and packages

Check whether the IPVS required kernel modules have been compiled into the kernel and packages installed. (see Requirements)


## Demo

Considering that you have the cluster running in `ipvs` mode and Calico is now configured, lets us create a `nginx-deployment` and a `service` and observe how `ipvs` loadbalancing works.


```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
EOF
```

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: service-nginx
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
EOF

```
Examine the ClusterIP of `service-nginx` service

```
kubectl get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP   27h
service-nginx   ClusterIP   10.99.170.70   <none>        80/TCP    115m

```

Now let us list the ipvs table and check how are service maps to the pods created using deployment. Blow output is trimmed for better understanding.

```
sudo ipvsadm -l
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  ip-10-99-170-70.us-west-2.co rr
-> ip-192-168-180-201.us-west-2 Masq    1      0          0
-> ip-192-168-180-202.us-west-2 Masq    1      0          0
-> ip-192-168-180-203.us-west-2 Masq    1      0          0
```
Here `ip-10-99-170-70.us-west-2.co` and the below is list of pods/endpoints. Here `rr` means the `loadbalancing used is round-robin`

If you are on the same host i.e. Master you can run the following script to generate traffic for the service and check the output stats in other tab by doing the following.

```
for i in {1..10}; do  curl <service-ip>:80 ; done

In the next tab analyze the packet flow per second using

watch sudo ipvsadm -L -n --rate

```
Here you will see the traffic getting distributed among the pods using `rr algorithm`
