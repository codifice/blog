---
title: "RPi Cluster (Part 4) - Installing Kubernetes"
date: 2020-02-20T20:47:08Z
draft: false
authors: ["martincjarvis"]
comments: true
thumbnail: "/devops-gallery"
series: ["rpi"]
---

Now the RPi Cluster is assembled, set up and has its network correctly configured we can install Kubernetes!

# On the Master RPi

```bash
sudo kubeadm config images pull -v3
sudo kubeadm init --token-ttl=0
```

At the end of the output, there's the command that must be used to join worker nodes to the master, it will resemble:

```bash
kubeadm join 10.0.1.2:6443 --token hqrwwd.zia49kiu5096aq0p \
    --discovery-token-ca-cert-hash sha256:2b22b8c62774bdfd051720326ccb49970457140a19ec9f425c28727ef4b4dae9
```

Initialise kubeadm:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install the pod network (Weave Net):

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

Check the pods in `kube-system` and confirm all is runing:

```bash
pi@master:~ $ kubectl get pods --namespace kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-6955765f44-h4nbv         1/1     Running   0          10m
coredns-6955765f44-nsk45         1/1     Running   0          10m
etcd-master                      1/1     Running   0          10m
kube-apiserver-master            1/1     Running   0          10m
kube-controller-manager-master   1/1     Running   0          10m
kube-proxy-njfp4                 1/1     Running   0          10m
kube-scheduler-master            1/1     Running   0          10m
weave-net-h6d7s                  2/2     Running   0          7m21s
```

>  You can also double check that Kubernetes is using the docker cache, by checking the `weaveworks/*` images are stored in the cache:
> ```bash
> pi@master:~ $ curl http://cache:5000/v2/_catalog
> {"repositories":["library/hello-world","library/redis","weaveworks/weave-kube","weaveworks/weave-npc"]}
> ```


## Join Worker Nodes

SSH onto each node and execute the join command provided previously:

```bash
sudo kubeadm join 10.0.1.2:6443 --token hqrwwd.zia49kiu5096aq0p \
    --discovery-token-ca-cert-hash sha256:2b22b8c62774bdfd051720326ccb49970457140a19ec9f425c28727ef4b4dae9
sudo sysctl net.bridge.bridge-nf-call-iptables=1    
```

You can monitor the status of the node via:

```bash
pi@master:~ $ kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
master   Ready      master   13m   v1.17.3
node1    NotReady   <none>   27s   v1.17.3
```

When all are added thi command's output should resemble:

```bash
pi@master:~ $ kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   18m     v1.17.3
node1    Ready    <none>   5m47s   v1.17.3
node2    Ready    <none>   102s    v1.17.3
node3    Ready    <none>   63s     v1.17.3
```

# Accessing Kubernetes from Laptop

As it stands the Kubernetes cluster is only available on the clusters network `10.0.1.0` and only `master` has the config to do so, so to access it from a machine on the host network we need to do some additional config.

## Gateway

First we want to makesure the gateway has the tools to admin the K8S cluster, as this will be the easiest box to SSH onto from a new host network.  To do this we need to copy the config from `master` on to RPi and check that connectivity works as expected.  Fortunately, this should be as simple as copying the `.kube` folder between the machines:


From the gateway RPi:
```bash
scp -r pi@master:~/.kube ~/
kubectl get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   37m   v1.17.3
node1    Ready    <none>   36m   v1.17.3
node2    Ready    <none>   33m   v1.17.3
node3    Ready    <none>   33m   v1.17.3
```

All Good :)

## Laptop

Using WSL, we can copy the `.kube` folder in a similar way to the gateway:

```bash
scp -r pi@<gateway-ip>:~/.kube ~/
```
 
> If you've configured `~/.ssh/config` with the SSH Proxy settings you can use `scp -r pi@master:~/.kube ~/` the same as on the Gateway RPi

To use windows based tooling copy the `.kube/config` into the appropriate profile location:

```bash
mkdir /mnt/c/users/<username>/.kube
cp ~/.kube/config /mnt/c/users/<username>/.kube/config
```

You will also need to add a routing rule so that all the `10.0.1.x` traffic is routed through your Gateway RPi.

On Windows (will be picked up by WSL):
```powershell
route ADD 10.0.1.0 MASK 255.255.255.0 <gateway-ip>
```

You can install the kubernetes tools for WSL using:
```bash
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
wget -qO - https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install -qy kubeadm
```

or for windows (using chocolatey):
```powershell
choco install kubernetes-cli
```

However, executing `kubectl get nodes` will result in an error as the Gateway RPi is not forwarding the appropriate traffic.

To enable the kubernetes tools access from host network we need to enable port forwarding off `6443`, `ssh` on the the Gateway RPi and configure the following:  

```bash
# Prepare port forwarding of eth1 traffic on 6443 to master (10.0.1.2)
sudo iptables -t nat -A PREROUTING -p tcp -i eth1 --dport 6443 -j DNAT --to-destination 10.0.1.2:6443
sudo iptables -A FORWARD -p tcp -i eth1 -d 10.0.1.2 --dport 6443 -j ACCEPT
# Prepare port forwarding of wlan0 traffic on 6443 to master (10.0.1.2)
sudo iptables -t nat -A PREROUTING -p tcp -i wlan0 --dport 6443 -j DNAT --to-destination 10.0.1.2:6443
sudo iptables -A FORWARD -p tcp -i wlan0 -d 10.0.1.2 --dport 6443 -j ACCEPT
# Do the routing, send the pack out on 10.0.1.1
sudo iptables  -t nat -A POSTROUTING -o 10.0.1.1 -j SNAT --to-source 10.0.1.2
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

You machine on the host network should now be able to access kubernetes via the tooling.  For example, the VS Code Kubernetes extension should automatically pick up the configuration and allow you to browse the RPi Kubernetes cluster:

{{< imgproc vscode-kubernetes Fill "300x600" >}}
VS Code Kubernetes Extension
{{< / imgproc>}}
