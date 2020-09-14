# Experimentation with k3s

I wanted to try k3s running on a VSI instance on IBM cloud.

See the [quickstart link](https://rancher.com/docs/k3s/latest/en/quick-start/)
from Rancher.

I created a VSI (Virtual Server Instance — i.e., Virtual Machine in IBM speak) on ibmcloud,
setup my `/etc/hosts` to have “vsi1” and the FIP used to ssh to the VSI, and setup my
own username (to avoid having to be root all the time).

## Installation, etc.

I'm not going to go into too detail as it's pretty easy and the website is
really nice.

Login and become root on the VSI:

```
sudo su
curl -sfL https://get.k3s.io | sh -
```

Here's how I installed a specific version.  I got the environment variable from
[here](https://rancher.com/docs/k3s/latest/en/upgrades/basic/)
and got a legal version string (e.g., "v1.16.14+k3s1") by looking at the
[releases page](https://github.com/rancher/k3s/releases).

```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.16.14+k3s1 sh -
```

Maybe 30 seconds later …

```
root@dperiquet-vsi1:/home/dperiquet# export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
root@dperiquet-vsi1:/home/dperiquet# kubectl get no
NAME             STATUS   ROLES    AGE   VERSION
dperiquet-vsi1   Ready    master   44s   v1.18.4+k3s1
```

Get back to being my user:

```
exit (become dperiquet)
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

dperiquet@dperiquet-vsi1:~$ kubectl get no
NAME             STATUS   ROLES    AGE   VERSION
dperiquet-vsi1   Ready    master   2m    v1.18.4+k3s1
```

Look around to see what's running by default:

```
dperiquet@dperiquet-vsi1:~$ kubectl get po -A
NAMESPACE     NAME                                     READY   STATUS      RESTARTS   AGE
kube-system   metrics-server-7566d596c8-d7t5k          1/1     Running     0          2m39s
kube-system   local-path-provisioner-6d59f47c7-qgfp4   1/1     Running     0          2m39s
kube-system   helm-install-traefik-9dtf2               0/1     Completed   0          2m39s
kube-system   svclb-traefik-dwpr7                      2/2     Running     0          2m20s
kube-system   coredns-8655855d6-gm88j                  1/1     Running     0          2m39s
kube-system   traefik-758cd5fc85-qxcfk                 1/1     Running     0          2m20s
```

Copy the kube my local machine (called "Mozart") in `~/kubes/` like I have for all my other Kubernetes
clusters.  Leave the server field as 127.0.0.1.

Setup an ssh tunnel to port 6443 to the VSI runnign k3s and confirm port 6443 connectivity:

```
Mozart:~ dperique$ ssh -i ~/.ssh/dperiquet_id_rsa -f -L 6443:127.0.0.1:6443 dperiquet@vsi1 sleep 9999
Mozart:~ dperique$ 
Mozart:~ dperique$ 
Mozart:~ dperique$ nc -vz 127.0.0.1 6443
found 0 associations
found 1 connections:
     1:	flags=82<CONNECTED,PREFERRED>
	outif lo0
	src 127.0.0.1 port 55361
	dst 127.0.0.1 port 6443
	rank info not available
	TCP aux info available

Connection to 127.0.0.1 port 6443 [tcp/sun-sr-https] succeeded!
```

Point to the Kubernetes cluster and see if it works:

```
Mozart:~ dperique$ export KUBECONFIG=~/kubes/k3s.yaml

Mozart:~ dperique$ kubectl get no
NAME             STATUS   ROLES    AGE   VERSION
dperiquet-vsi1   Ready    master   10m   v1.18.4+k3s1
```

## Try a simple container

Use this from my tool-container project:

```
$ cat dp-client.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: dp-client
spec:
  hostname: dp-client
  containers:
  - image: busybox
    imagePullPolicy: Always
    name: dp-client
    command: [ "sleep", "999999" ]

$ kubectl apply -f dp-client.yaml
pod/dp-client created

$ kubectl get po
NAME        READY   STATUS    RESTARTS   AGE
dp-client   1/1     Running   0          4s

$ kubectl exec -ti dp-client -- sh
/ # 
/ # route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.42.0.1       0.0.0.0         UG    0      0        0 eth0
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.42.0.0       10.42.0.1       255.255.0.0     UG    0      0        0 eth0
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue 
    link/ether 46:b9:42:ee:56:52 brd ff:ff:ff:ff:ff:ff
    inet 10.42.0.9/24 brd 10.42.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::44b9:42ff:feee:5652/64 scope link 
       valid_lft forever preferred_lft forever
/ # ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=118 time=1.303 ms
^C
--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.303/1.303/1.303 ms
```

Basic things work -- nice.

## Uninstall

All I did was login to my k3s host (VSI), become root and run this:

```
root@dperiquet-vsi1:# cd /usr/local/bin
root@dperiquet-vsi1:/usr/local/bin# ./k3s-uninstall.sh 
```

## Try k3s with docker as the container runtime

I'm most familiar with docker as the container runtime so let's use docker.
This time I timed it and it took 13 seconds to finish install.  But the node
took about 20s more to become ready -- nice.

```
dperiquet@dperiquet-vsi1:~/k3s_install$ sudo apt-get install docker.io
dperiquet@dperiquet-vsi1:~/k3s_install$ sudo su
root@dperiquet-vsi1:/home/dperiquet/k3s_install# curl -sfL https://get.k3s.io | sh -s - --docker
[INFO]  Finding release for channel stable
[INFO]  Using v1.18.4+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.18.4+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.18.4+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```

See the node and note it's not ready:

```
root@dperiquet-vsi1:/home/dperiquet/k3s_install# kubectl get no
NAME             STATUS     ROLES    AGE   VERSION
dperiquet-vsi1   NotReady   master   4s    v1.18.4+k3s1
root@dperiquet-vsi1:/home/dperiquet/k3s_install# kubectl get no
NAME             STATUS     ROLES    AGE   VERSION
dperiquet-vsi1   NotReady   master   7s    v1.18.4+k3s1
root@dperiquet-vsi1:/home/dperiquet/k3s_install# kubectl get no
NAME             STATUS     ROLES    AGE   VERSION
dperiquet-vsi1   NotReady   master   8s    v1.18.4+k3s1
```

Describe the node in case we need to debug:

```
root@dperiquet-vsi1:/home/dperiquet/k3s_install# kubectl describe no dperiquet-vsi1
Name:               dperiquet-vsi1
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=k3s
                    beta.kubernetes.io/os=linux
                    k3s.io/hostname=dperiquet-vsi1
                    k3s.io/internal-ip=10.240.128.4
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=dperiquet-vsi1
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=true
                    node.kubernetes.io/instance-type=k3s
Annotations:        flannel.alpha.coreos.com/backend-data: {"VtepMAC":"d6:b3:4c:9e:0f:7d"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.240.128.4
                    k3s.io/node-args: ["server","--docker"]
                    k3s.io/node-config-hash: ZIZ7LB6HJANXHCAGBJRXBOLNWSAHAML5HGK3IEE2D23AHRHY623A====
                    k3s.io/node-env: {"K3S_DATA_DIR":"/var/lib/rancher/k3s/data/3a8d3d90c0ac3531edbdbde77ce4a85062f4af8865b98cedc30ea730715d9d48"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sun, 05 Jul 2020 19:36:21 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  dperiquet-vsi1
  AcquireTime:     <unset>
  RenewTime:       Sun, 05 Jul 2020 19:36:40 +0000
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Sun, 05 Jul 2020 19:36:35 +0000   Sun, 05 Jul 2020 19:36:35 +0000   FlannelIsUp                  Flannel is running on this node
  MemoryPressure       False   Sun, 05 Jul 2020 19:36:31 +0000   Sun, 05 Jul 2020 19:36:21 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Sun, 05 Jul 2020 19:36:31 +0000   Sun, 05 Jul 2020 19:36:21 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Sun, 05 Jul 2020 19:36:31 +0000   Sun, 05 Jul 2020 19:36:21 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Sun, 05 Jul 2020 19:36:31 +0000   Sun, 05 Jul 2020 19:36:31 +0000   KubeletReady                 kubelet is posting ready status. AppArmor enabled
Addresses:
  InternalIP:  10.240.128.4
  Hostname:    dperiquet-vsi1
Capacity:
  cpu:                8
  ephemeral-storage:  102820212Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             30672584Ki
  pods:               110
Allocatable:
  cpu:                8
  ephemeral-storage:  100023502156
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             30672584Ki
  pods:               110
System Info:
  Machine ID:                 7567cf2caf11435ebdb4e0a6deb015af
  System UUID:                3E6A6FC0-BED2-11EA-9909-FEFF0B60470B
  Boot ID:                    873eb814-2657-41fe-95b3-f007afb676d9
  Kernel Version:             4.15.0-42-generic
  OS Image:                   Ubuntu 18.04.1 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://19.3.6
  Kubelet Version:            v1.18.4+k3s1
  Kube-Proxy Version:         v1.18.4+k3s1
PodCIDR:                      10.42.0.0/24
PodCIDRs:                     10.42.0.0/24
ProviderID:                   k3s://dperiquet-vsi1
Non-terminated Pods:          (4 in total)
  Namespace                   Name                                      CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                                      ------------  ----------  ---------------  -------------  ---
  kube-system                 helm-install-traefik-5p8mr                0 (0%)        0 (0%)      0 (0%)           0 (0%)         7s
  kube-system                 local-path-provisioner-6d59f47c7-fctsw    0 (0%)        0 (0%)      0 (0%)           0 (0%)         7s
  kube-system                 metrics-server-7566d596c8-2vwjf           0 (0%)        0 (0%)      0 (0%)           0 (0%)         7s
  kube-system                 coredns-8655855d6-pf2kd                   100m (1%)     0 (0%)      70Mi (0%)        170Mi (0%)     7s
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests   Limits
  --------           --------   ------
  cpu                100m (1%)  0 (0%)
  memory             70Mi (0%)  170Mi (0%)
  ephemeral-storage  0 (0%)     0 (0%)
  hugepages-1Gi      0 (0%)     0 (0%)
  hugepages-2Mi      0 (0%)     0 (0%)
Events:
  Type    Reason                   Age                From                        Message
  ----    ------                   ----               ----                        -------
  Normal  Starting                 21s                kubelet, dperiquet-vsi1     Starting kubelet.
  Normal  NodeHasSufficientMemory  20s (x2 over 20s)  kubelet, dperiquet-vsi1     Node dperiquet-vsi1 status is now: NodeHasSufficientMemory
  Normal  NodeHasSufficientPID     20s (x2 over 20s)  kubelet, dperiquet-vsi1     Node dperiquet-vsi1 status is now: NodeHasSufficientPID
  Normal  NodeHasNoDiskPressure    20s (x2 over 20s)  kubelet, dperiquet-vsi1     Node dperiquet-vsi1 status is now: NodeHasNoDiskPressure
  Normal  NodeAllocatableEnforced  20s                kubelet, dperiquet-vsi1     Updated Node Allocatable limit across pods
  Normal  Starting                 20s                kube-proxy, dperiquet-vsi1  Starting kube-proxy.
  Normal  NodeReady                10s                kubelet, dperiquet-vsi1     Node dperiquet-vsi1 status is now: NodeReady
```

The node is up:

```
root@dperiquet-vsi1:/home/dperiquet/k3s_install# kubectl get no
NAME             STATUS   ROLES    AGE   VERSION
dperiquet-vsi1   Ready    master   30s   v1.18.4+k3s1
```

you can see the all the k3s images are docker images (the busybox one is for my dp-client.yaml):

```
dperiquet@dperiquet-vsi1:~/k3s_install$ sudo docker images
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
busybox                          latest              c7c37e472d31        5 days ago          1.22MB
rancher/klipper-helm             v0.2.5              6207e2a3f522        2 months ago        136MB
rancher/library-traefik          1.7.19              aa764f7db305        8 months ago        85.7MB
rancher/metrics-server           v0.3.6              9dd718864ce6        8 months ago        39.9MB
rancher/local-path-provisioner   v0.0.11             9d12f9848b99        9 months ago        36.2MB
rancher/coredns-coredns          1.6.3               c4d3d16fe508        10 months ago       44.3MB
rancher/klipper-lb               v0.1.2              897ce3c5fc8f        13 months ago       6.1MB
rancher/pause                    3.1                 da86e6ba6ca1        2 years ago         742kB
```

Running k3s using docker is nice because it uses the local docker instance automatically.
This means that you can build your own docker images and use them in your Kubernetes manifests
and they will load; i.e., you don't need a separate docker registry.  This is nice especially
if you're doing local development and/or testing.

## Comparison with minikube

I've been using minikube for a while and use `--vm-driver=none` mode so I can use the local
host to install minikube (as opposed to having to run libvirt or some other hypervisor so I can
create a VM to install the minikube software).  Using `--vm-driver=none` also makes it so I can
install docker images locally without having to have a separate docker registry.

K3s (using the docker installation mode) works this way by default, does not need a hypervisor,
installs in well under a minute, and apparently uses a small resource "footprint".

I'm going to be using k3s more and will update this with more experiences.
