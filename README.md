Hello everyone, I'm Arsen. You can follow this guide to setup a simple, 4-node(1 master and 3 workers) Kubernetes cluster and
install Rook-Ceph on top of it.

This was tested on Ubuntu 22.04.1 LTS.
* Requirements
```
vagrant
virtualbox
at least 7GB of RAM and 50GB of disk space
```

--KUBERNETES CLUSTER SETUP--

Thanks to Mehmet A. Baykara, I followed his post(https://baykara.medium.com/setup-own-kubernetes-cluster-via-virtualbox-99a82605bfcc)
to setup my own K8s cluster but I had to make some changes in order to setup Rook-Ceph on top of it. So, if you want to follow
his guide and make your own configurations, you can visit his guide to setup your K8s cluster or you can directly follow this guide.

First clone the repo and cd into it using the following command:
```
git clone https://github.com/arsenguler/Rook-Ceph-Setup-w-Vagrant.git
cd Rook-Ceph-Setup-w-Vagrant
```
Start VMs using the following command:
``` 
VAGRANT_EXPERIMENTAL=disks vagrant up
```
By default, each VM gets 50GB of storage. You can change the size if you want by modifying the Vagrantfile.

To access the master node, use the following command:
```
vagrant ssh master
```

To access the worker node "n"(1,2 or 3), use the following command:
```
vagrant ssh worker-n
```

Ssh into the master node and switch to root user with the following command:
```
vagrant shh master
Sudo su
```

Initialize kubeadm on the master node using the following command:
```
kubeadm init --apiserver-advertise-address 192.168.56.13 --pod-network-cidr=192.168.0.0/16
```
I used Calico as the CNI but you can change the cidr if you want to use flannel or any other CNI.

After you successfully initialized your control-plane, save the command following with
"kubeadm join 192.168.56.13" from the response of the previous command. It will be used to make
worker nodes to join the cluster.


Exit from the root and run the following command as an arbitrary user:
```
exit
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Download the Calico networking manifest for the Kubernetes API datastore with the following:
```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.4/manifests/calico.yaml -O
```

If you've used any other cidr than 192.168.0.0/16, uncomment the CALICO_IPV4POOL_CIDR variable 
in the manifest and set it to the same value as your chosen cidr. Else, skip this step.

Apply the manifest using the following command.
```
kubectl apply -f calico.yaml
```

Add workers to the cluster by running the following inside each worker:
```
vagrant ssh worker-n
sudo <Command starting with "kubeadm join 192.168.56.13" you saved after running kubeadm init>
```

Now the k8s cluster should be functional. You can test it by running the following and checking if
status' are healthy.
```
kubectl get cs
```

--ROOK-CEPH SETUP--

If the k8s cluster setup was successful, setting up rook-ceph on top of it should be quite straight-forward.
You can follow the official guide(https://rook.io/docs/rook/latest/Getting-Started/quickstart/) if you want
since I will be pretty much copying it.

First, clone the repo and cd into the examples folder by the following:
```
git clone --single-branch --branch master https://github.com/rook/rook.git
cd rook/deploy/examples
```

Apply crds.yaml, common.yaml and operator.yaml manifests using the following:
```
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```

Make sure the rook-ceph-operator is in the `Running` state before proceeding. Suggestion:
```
kubectl -n rook-ceph get pod --watch
```

Create the cluster using the following:
```
kubectl create -f cluster.yaml
```

Check if the necessary pods are running(might take some time) using the following:
```
kubectl -n rook-ceph get pod
```
By default, there should be 3mon pods, 2 mgr pods and 3 osd pods.

To verify that the cluster is in a healthy state, connect to the Rook toolbox
and run the ceph status command by the following:
```
kubectl create -f deploy/examples/toolbox.yaml
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
ceph status
```

If something goes wrong, it might be helpful to correct the following:

*Check if VM hostnames are ok in each VM:
```
sudo vim /etc/hosts
```
It should be as the following:
192.168.56.13  master master
192.168.56.14  worker-1 worker-1
192.168.56.15  worker-2 worker-2
192.168.56.16  worker-3 worker-3

*Make sure you can ssh into localhost in your physical mahcine.
```
ssh root@localhost
```
If that isn't possible please check your /etc/ssh/sshd_config file and change the 
following option:
```
PermitRootLogin yes
```
Afterwards copy your pub-key by running:
```
ssh-copy-id root@localhost
```

*Make sure you have a separate 10GB raw disk(probably named "sdc) by running:
```
lsblk
```
and check if it is NOT labelled by running:
```
lsblk -f
```
