# Build a k8s cluster (Flannel & MetalLB & Rook/Ceph)

kubernetes clusters are used as applications for users.

The kubernetes cluster is built in the vshpere environment. Create a template vm and save it in the content library. You can clone master node and worker nodes from template vm with __vsphere rest api__ and build a cluster with __ansible__.

## The OS on which this software was tested

- VERSION="22.04.2 LTS (Jammy Jellyfish)"

## Build Vsphere template VM for k8s cluster

### Install container runtime

1. Update the `apt` package index and install packages to allow `apt` to use a repository over HTTPS:

```
sudo apt-get update
sudo apt-get install -y vim git ca-certificates curl gnupg lsb-release software-properties-common apt-transport-https open-vm-tools containerd
```

2. Configure containerd.

```
sudo mkdir /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Edit config.toml

```
(snip)
  [plugins."io.containerd.grpc.v1.cri"]
(snip)
   sandbox_image = "registry.k8s.io/pause:3.9"
(snip)
    [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
(snip)
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."10.2.23.129:5000"]
          endpoint = ["http://10.2.23.129:5000"]
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."http://10.2.23.129:5000".tls]
(snip)
```

```
sudo systemctl restart containerd
```

### Install kubelet, kubeadm and kubectl

1. Add hostname of k8s master node and worker nodes to hosts file

```
sudo vim /etc/hosts
```

```
10.127.209.251 ub2204-k8s-master
10.127.209.252 ub2204-k8s-worker1
10.127.209.253 ub2204-k8s-worker2
10.127.209.254 ub2204-k8s-worker3
```

```
sudo systemctl reboot
```

2. Install kubelet, kubeadm and kubectl

Google Cloud public key download.

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add kubernetes repository to apt repository.

```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install.

```
sudo apt-get update
sudo apt-get -y install kubeadm kubelet kubectl kubernetes-cni
sudo apt-mark hold kubelet kubeadm kubectl
```

> Confirm installation by checking the version of kubectl.
> `kubectl version --client && kubeadm version`

3. Disable Swap

If you want to temporarily turn off swap.

```
sudo swapoff -a
```

Comment out swap from fstab so that it is not used.

```
sudo vim /etc/fstab
```

```
- /swap.img      none    swap    sw      0       0\
+ #/swap.img      none    swap    sw      0       0\
```

4. Enable kernel modules

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

5. Enable IPv4 forwarding.
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

6. Reload sysctl

```
sudo sysctl --system
```

7. make sure that the br_netfilter module is loaded:

```
lsmod | grep br_netfilter
```

8. Enable kubelet service

```
sudo systemctl enable kubelet
```

9. Pull container images:

```
sudo kubeadm config images pull
```

## Build k8s cluster (Manually)

__If you build cluster manually instead of automatically using vshpere api.__

### Build k8s master node

#### Initialize master node

1. Here is the output of my initialization command

```
sudo kubeadm init --apiserver-advertise-address=< IP address of master node > --pod-network-cidr=10.244.0.0/16
```

2. Configure kubectl using commands in the output:

```
mkdir -p $HOME/.kube
sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

3. Check cluster status:

```
kubectl cluster-info
```

#### Install network plugin on Master

1. Enable deploying pods on master node

```
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

2. Install flannel

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

> If your environment is under proxy, you may need copy all images of flannel from local resource.

3. Install MetalLB

Layer 2 Configuration metallb_configmap.yaml.

```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 10.127.209.221-10.127.209.249
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default
```

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
kubectl apply -f ./metallb_configmap.yaml 
```

4. Confirm that all of the pods are running:

```
kubectl get nodes
kubectl get pods --all-namespaces -o wide
```

5. Confirm master node is ready:

```
kubectl get nodes -o wide
```

### Build k8s worker nodes

1. Add worker nodes

The join command that was given is used to add a worker node to the cluster

```
sudo kubeadm join <Master node address>:6443 --token < Token > --discovery-token-ca-cert-hash sha256:< hash >
kubectl get nodes
```

> if you can not add worker nodes, try command as follow:<br>
> On Master node side<br>
>  `sudo kubeadm reset`<br>
>  `sudo kubeadm token create --print-join-command`<br>
> On worker node side<br>
>  `sudo kubeadm reset`<br>
>  `sudo kubeadm join <Master node address>:6443 --token < Token > --discovery-token-ca-cert-hash sha256:< hash >`<br>

### Install Rook/Cepf

1. Add SCSI HDD with Nutanix Prism

To configure the Ceph storage cluster, at least one of these local storage types is required:<br>
- Raw devices (no partitions or formatted filesystems)<br>
- Raw partitions (no formatted filesystem)<br>
- LVM Logical Volumes (no formatted filesystem)<br>
- Persistent Volumes available from a storage class in block mode<br>

2. Deploy the Rook Operator

A simple Rook cluster is created for Kubernetes with the following kubectl commands and example manifests.

```
git clone --single-branch --branch v1.11.11 https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

Verify.

```
nail@ub2204-k8s-master:~/rook/deploy/examples$ kubectl get all -n rook-ceph
NAME                                     READY   STATUS    RESTARTS   AGE
pod/rook-ceph-operator-684bfdb79-dvjpr   1/1     Running   0          2m14s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rook-ceph-operator   1/1     1            1           2m14s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/rook-ceph-operator-684bfdb79   1         1         1       2m14s
nail@ub2204-k8s-master:~/rook/deploy/examples$
```

3. Create a Ceph Cluster

```
kubectl create -f cluster.yaml
```

Verify.

```
nail@ub2204-k8s-master:~/rook/deploy/examples$ kubectl get all -n rook-ceph
NAME                                                               READY   STATUS      RESTARTS   AGE
pod/csi-cephfsplugin-jh8mq                                         2/2     Running     0          5m11s
pod/csi-cephfsplugin-l4hdw                                         2/2     Running     0          5m11s
pod/csi-cephfsplugin-provisioner-859b9d7776-hw5wb                  5/5     Running     0          5m11s
pod/csi-cephfsplugin-provisioner-859b9d7776-nz8l4                  5/5     Running     0          5m11s
pod/csi-cephfsplugin-v5jvh                                         2/2     Running     0          5m11s
pod/csi-cephfsplugin-whlzn                                         2/2     Running     0          5m11s
pod/csi-rbdplugin-4sc47                                            2/2     Running     0          5m11s
pod/csi-rbdplugin-5hrt6                                            2/2     Running     0          5m11s
pod/csi-rbdplugin-97p4f                                            2/2     Running     0          5m11s
pod/csi-rbdplugin-hf5l4                                            2/2     Running     0          5m11s
pod/csi-rbdplugin-provisioner-5d6595bf68-8g77x                     5/5     Running     0          5m11s
pod/csi-rbdplugin-provisioner-5d6595bf68-92smp                     5/5     Running     0          5m11s
pod/rook-ceph-crashcollector-ub2204-k8s-master-9465b6ddd-vtc2h     1/1     Running     0          2m30s
pod/rook-ceph-crashcollector-ub2204-k8s-worker1-59d7868b5c-8jj9d   1/1     Running     0          3m35s
pod/rook-ceph-crashcollector-ub2204-k8s-worker2-554d58b55b-gsd97   1/1     Running     0          3m19s
pod/rook-ceph-crashcollector-ub2204-k8s-worker3-6f8bb86fbc-qktfk   1/1     Running     0          3m19s
pod/rook-ceph-mgr-a-5777997c9d-ms2t9                               3/3     Running     0          3m52s
pod/rook-ceph-mgr-b-5d989c5b4c-dqg86                               3/3     Running     0          3m50s
pod/rook-ceph-mon-a-7d686468c4-tpvsn                               2/2     Running     0          5m9s
pod/rook-ceph-mon-b-7756f774cb-xkg9k                               2/2     Running     0          4m17s
pod/rook-ceph-mon-c-586bbfbdb8-f4cfz                               2/2     Running     0          4m5s
pod/rook-ceph-operator-684bfdb79-dvjpr                             1/1     Running     0          9m37s
pod/rook-ceph-osd-0-d445d57bc-8dskc                                2/2     Running     0          3m20s
pod/rook-ceph-osd-1-6d8d64fbc7-gj6nt                               2/2     Running     0          3m19s
pod/rook-ceph-osd-2-cbc568b4b-82w7n                                2/2     Running     0          3m19s
pod/rook-ceph-osd-3-d5b5958db-wq4jt                                2/2     Running     0          2m31s
pod/rook-ceph-osd-prepare-ub2204-k8s-master-rjszn                  0/1     Completed   0          2m9s
pod/rook-ceph-osd-prepare-ub2204-k8s-worker1-kpmfl                 0/1     Completed   0          2m6s
pod/rook-ceph-osd-prepare-ub2204-k8s-worker2-7ktq4                 0/1     Completed   0          2m3s
pod/rook-ceph-osd-prepare-ub2204-k8s-worker3-xhzdl                 0/1     Completed   0          2m

NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/rook-ceph-mgr             ClusterIP   10.100.12.149    <none>        9283/TCP            3m34s
service/rook-ceph-mgr-dashboard   ClusterIP   10.103.249.175   <none>        8443/TCP            3m34s
service/rook-ceph-mon-a           ClusterIP   10.103.185.131   <none>        6789/TCP,3300/TCP   5m11s
service/rook-ceph-mon-b           ClusterIP   10.100.140.165   <none>        6789/TCP,3300/TCP   4m18s
service/rook-ceph-mon-c           ClusterIP   10.97.25.12      <none>        6789/TCP,3300/TCP   4m7s

NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/csi-cephfsplugin   4         4         4       4            4           <none>          5m11s
daemonset.apps/csi-rbdplugin      4         4         4       4            4           <none>          5m11s

NAME                                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/csi-cephfsplugin-provisioner                  2/2     2            2           5m11s
deployment.apps/csi-rbdplugin-provisioner                     2/2     2            2           5m11s
deployment.apps/rook-ceph-crashcollector-ub2204-k8s-master    1/1     1            1           2m31s
deployment.apps/rook-ceph-crashcollector-ub2204-k8s-worker1   1/1     1            1           3m35s
deployment.apps/rook-ceph-crashcollector-ub2204-k8s-worker2   1/1     1            1           3m50s
deployment.apps/rook-ceph-crashcollector-ub2204-k8s-worker3   1/1     1            1           3m52s
deployment.apps/rook-ceph-mgr-a                               1/1     1            1           3m52s
deployment.apps/rook-ceph-mgr-b                               1/1     1            1           3m50s
deployment.apps/rook-ceph-mon-a                               1/1     1            1           5m9s
deployment.apps/rook-ceph-mon-b                               1/1     1            1           4m17s
deployment.apps/rook-ceph-mon-c                               1/1     1            1           4m5s
deployment.apps/rook-ceph-operator                            1/1     1            1           9m37s
deployment.apps/rook-ceph-osd-0                               1/1     1            1           3m20s
deployment.apps/rook-ceph-osd-1                               1/1     1            1           3m20s
deployment.apps/rook-ceph-osd-2                               1/1     1            1           3m19s
deployment.apps/rook-ceph-osd-3                               1/1     1            1           2m31s

NAME                                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/csi-cephfsplugin-provisioner-859b9d7776                  2         2         2       5m11s
replicaset.apps/csi-rbdplugin-provisioner-5d6595bf68                     2         2         2       5m11s
replicaset.apps/rook-ceph-crashcollector-ub2204-k8s-master-9465b6ddd     1         1         1       2m30s
replicaset.apps/rook-ceph-crashcollector-ub2204-k8s-worker1-59d7868b5c   1         1         1       3m35s
replicaset.apps/rook-ceph-crashcollector-ub2204-k8s-worker2-554d58b55b   1         1         1       3m19s
replicaset.apps/rook-ceph-crashcollector-ub2204-k8s-worker2-7b86456f4    0         0         0       3m50s
replicaset.apps/rook-ceph-crashcollector-ub2204-k8s-worker3-6f778666d9   0         0         0       3m52s
replicaset.apps/rook-ceph-crashcollector-ub2204-k8s-worker3-6f8bb86fbc   1         1         1       3m19s
replicaset.apps/rook-ceph-mgr-a-5777997c9d                               1         1         1       3m52s
replicaset.apps/rook-ceph-mgr-b-5d989c5b4c                               1         1         1       3m50s
replicaset.apps/rook-ceph-mon-a-7d686468c4                               1         1         1       5m9s
replicaset.apps/rook-ceph-mon-b-7756f774cb                               1         1         1       4m17s
replicaset.apps/rook-ceph-mon-c-586bbfbdb8                               1         1         1       4m5s
replicaset.apps/rook-ceph-operator-684bfdb79                             1         1         1       9m37s
replicaset.apps/rook-ceph-osd-0-d445d57bc                                1         1         1       3m20s
replicaset.apps/rook-ceph-osd-1-6d8d64fbc7                               1         1         1       3m20s
replicaset.apps/rook-ceph-osd-2-cbc568b4b                                1         1         1       3m19s
replicaset.apps/rook-ceph-osd-3-d5b5958db                                1         1         1       2m31s

NAME                                                 COMPLETIONS   DURATION   AGE
job.batch/rook-ceph-osd-prepare-ub2204-k8s-master    1/1           6s         2m9s
job.batch/rook-ceph-osd-prepare-ub2204-k8s-worker1   1/1           6s         2m6s
job.batch/rook-ceph-osd-prepare-ub2204-k8s-worker2   1/1           6s         2m3s
job.batch/rook-ceph-osd-prepare-ub2204-k8s-worker3   1/1           6s         2m
nail@ub2204-k8s-master:~/rook/deploy/examples$```
```

3. Ceph Dashboard

Now create the service.

```
kubectl apply -f dashboard-loadbalancer.yaml
```

Output login password.

```
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo 
```

4. Shared Filesystem

Ceph filesystem (CephFS) allows the user to mount a shared posix-compliant folder into one or more application pods. <br>

Deploy Filesystem.

```
kubectl apply -f filesystem.yaml
```

Deploy storageclass.

```
cd ~/rook/deploy/examples/csi/cephfs/
kubectl apply -f storageclass.yaml
```

> If you want to keep the data permanently, change reclaimPolicy to retain.

Deploy PVC.

```
kubectl apply -f pvc.yaml
```

> Change the storage volume you want accordingly.

### Only k8s client nodes

1. Add Docker’s official GPG key and k8s's apt-key:

```
sudo apt update
sudo apt install -y curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
2. Add apt repository

```
echo \
  "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

3. Install kubectl

```
sudo apt update
sudo apt install -y kubectl
```

4. Copy kube config from master node to clinet node

```
sudo mkdir ~/.kube
sudo scp nail@172.16.1.21:~/.kube/config ~/.kube
sudo chown -R root:root ~/.kube/
sudo chmod -R 755 ~/.kube/
```

5. Verify

```
kubectl get nodes
```

## Create application pods and Services

### Copy a app-deployment file to k8s-master and Create app deployments and services

__NSO__

```
kubectl apply -f ./nso-kubernetes/nik-deployment.yaml
```

> when you want to delete deployment and service.
> kubectl apply -f ./nso-kubernetes/nik-deployment.yaml

### If you use kind

Installing From Release Binaries.

```
curl --insecure -Lo ./kind "https://github.com/kubernetes-sigs/kind/releases/download/v0.11.1/kind-linux-amd64"
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```
