---
title:  "Home Kubernetes cluster with GlusterFS"
date:   2018-04-20 03:00:00
categories: [software]
tags: [kubernetes, glusterfs]
---

I have been experimenting with my own kubernetes cluster at home, it is
a simple set up of five headless servers running Debian with three 2 TB
HDDs attached to them for storage. I am sharing my notes to help others
save time and effort.

![My home cluster]({{ "/assets/home_cluster.jpg" | absolute_url }})

For what it's worth I use docker for local development, this is just my own
"production environment" to experiment and run services at home.

## Router configuration.

Some things to consider:

1. Enforce security.
2. Private WiFi, no guests allowed.
3. Set an admin user and password.
4. Block all incoming connections by default.
5. Disable UPnP and port forwarding.
6. Assign predefined local IP addresses to servers.

## Kubernetes cluster installation instructions.

### Server installation.

I personally like [Debian](https://www.debian.org/) and that is what I use in all the machines at home.

The installation is straightforward:

**(1)** Download the small installation image from https://www.debian.org/distrib/ .

**(2)** Prepare USB drive and format it with FAT filesystem, OS X users can use the DiskUtil application. [1]

Find the device, unmount and write the image to it.

Example:

```
# umount /dev/disk2
# diskutil unmountDisk /dev/disk2
# dd if=debian-9.2.0-amd64-netinst.iso of=/dev/disk2 bs=1m
```

**(3)** Install Debian on each server by following the documentation [2].

*Note:* Environments with strict security requirements should consider the use of full disk encryption, IDS and secure configuration settings that are beyond the scope of this document.

Remember to disable swap partitions.

Here is a list of suggested packages to install in each server:

1. vim
2. hdparm
3. smartmontools
4. memtest86
5. net-tools
6. dnsutils
7. unzip
8. golang


Set up automatic installation of security updates by following instructions in [5].
Check the following file to confirm:

```
# cat /etc/apt/apt.conf.d/20auto-upgrades

# Value = "1" means that it will check every day.
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";

# cat https://www.debian.org/distrib/

(...)

```

Files to copy to each server:

1. /etc/motd    # Login message.
2. /etc/modules # Kernel modules to load.

Content of */etc/modules:*:

```
br_netfilter
dm_snapshot
dm_mirror
dm_thin_pool
```

**(4)** Install SSH server.

Follow instructions in [3] and [4],  after the installation is complete connect to each server, upload your public key, edit the SSH server configuration file and make sure to disable password logins and require keys for authentication.

Tip: Add hosts to your personal computer's SSH config file in ~/.ssh/config for quick access.

### Kubernetes installation.

Read the official documentation at https://kubernetes.io to understand
general concepts and for general instructions.

Use *kubeadm*, follow instructions in [6] (Installing kubeadm) and [7]
(Using kubeadm to create a cluster).

Disable swap [8] in each server and apply settings for CIN plugins
(network).
```
# cat /proc/swaps
# swapoff -a
# vim /etc/fstab # Remove references.
# vim /etc/initramfs-tools/conf.d/resume # Comment out swap partition
UUID.
# update-initramfs -u
```
```
# grub-install --version # Find grub version being utilized.
grub-install (GRUB) 2.02~beta3-5
# update-grub2

(reboot and confirm swap is off)
```

If for any reason it is not possible to disable swap, then turn it off
during system start time:

```
# vim /etc/systemd/system/swapoff.service

[Unit]
Description=Turn off swap
After=multi-user.target

[Service]
ExecStart=/sbin/swapoff -a

[Install]
WantedBy=default.target
```

```
# systemctl enable swapoff.service
# systemctl start swapoff.service

(reboot and confirm swap is off)
```

Network related settings:

```
# sysctl net.bridge.bridge-nf-call-iptables=1 # In each server.
```

If the variable does not exist then search for the kernel module and
load it:

```
# grep -rl bridge-nf-call-iptables /lib/modules/`uname -r`
# modprobe br_netfilter
# lsmod | grep  br_netfilter
br_netfilter           24576  0
bridge                135168  1 br_netfilter
```

Then ensure it loads every time the system starts:

```
# cat /etc/modules | grep br_netfilter
br_netfilter

( Execute the sysctl command again, reboot and check)
```

Ensure that each server has unique MAC address and *product_uuid* [6].

MAC address:

```
# ip link
```

*product_uuid:*

```
# cat /sys/class/dmi/id/product_uuid
```


Install docker on each server, see instructions in [9] (Docker CE for
Debian).

```
# apt-get install \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common
```

```
# curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key
add -
```

```
# add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
```

```
# apt-get update
# apt-get install docker-ce
```

Install specific version of docker that is known to work:
```
# apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
```

Install kubeadm, kubelet and kubectl:

```
# apt-get update && apt-get install -y apt-transport-https curl
# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg |
apt-key add -
# apt-get update
```

```
# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

```
# apt-get update
# apt-get install -y kubelet kubeadm kubectl
```

*kubeadmin:* Utility to bootstrap cluster.

*kubelet:* Primary node agent that runs in each server of the cluster.

*kubectl:* Command line utility for running commands in the cluster.


Configure cgroup driver used by kubelet on master node:

```
# docker info | grep -i cgroup
Cgroup Driver: cgroupfs
```
```
# cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS
```

There is no cgroup driver, therefore there is no flag to change and the step in the instruction does not apply.

Intialize the master server:

```
# kubeadmin init # In the master server.

[init] Using Kubernetes version: v1.10.1
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [sigant001 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.2.108]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [localhost] and IPs [127.0.0.1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [sigant001] and IPs [192.168.2.108]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
[init] This might take a minute or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 73.008616 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node sigant001 as master by adding a label and a taint
[markmaster] Master sigant001 tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: avmvib.5c7s9dcio1nxuzk6
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.2.108:6443 --token [FILTERED] --discovery-token-ca-cert-hash [FILTERED]
```

```
# mkdir -p $HOME/.kube
# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# sudo chown $(id -u):$(id -g) $HOME/.kube/config
(or, as root)
# export KUBECONFIG=/etc/kubernetes/admin.conf
```

Install a pod network:

Seleted third-party Pod Network Provider: *Weave Net*.


```
# export kubever=$(kubectl version | base64 | tr -d '\n')
[FILTERED]

# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"
serviceaccount "weave-net" created
clusterrole.rbac.authorization.k8s.io "weave-net" created
clusterrolebinding.rbac.authorization.k8s.io "weave-net" created
role.rbac.authorization.k8s.io "weave-net" created
rolebinding.rbac.authorization.k8s.io "weave-net" created
daemonset.extensions "weave-net" created
```

Check that *kube-dns* is running:

```
# kubectl get pods --all-namespaces
NAMESPACE     NAME                                READY     STATUS    RESTARTS   AGE
kube-system   etcd-sigant001                      1/1       Running   0          10m
kube-system   kube-apiserver-sigant001            1/1       Running   0          10m
kube-system   kube-controller-manager-sigant001   1/1       Running   0          10m
kube-system   kube-dns-86f4d74b45-2pdrq           2/3       Running   0          11m
kube-system   kube-proxy-bj4lw                    1/1       Running   0          11m
kube-system   kube-scheduler-sigant001            1/1       Running   0          10m
kube-system   weave-net-8wntq                     2/2       Running   0          40s
```

Master isolation, if you want to schedule pods on the master server (not recommended for security reasons but it depends on your environment):

```
# kubectl taint nodes --all node-role.kubernetes.io/master-
node "sigant001" untainted
```

```
(On each server)
# mkdir ~/go
# export GOPATH=$HOME/go
# go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
```

Then join all other servers (nodes):

```
(In each server, use the command to join nodes:)
kubeadm join 192.168.2.108:6443 --token [FILTERED] --discovery-token-ca-cert-hash sha256:[FILTERED]

[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
[discovery] Trying to connect to API Server "192.168.2.108:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.2.108:6443"
[discovery] Requesting info from "https://192.168.2.108:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "192.168.2.108:6443"
[discovery] Successfully established connection with API Server "192.168.2.108:6443"

This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

The pre-flight check warning can be ignored (see github issues in the kubernetes project).

Check to confirm that all servers are ready:

```
# kubectl get nodes

NAME        STATUS    ROLES     AGE       VERSION
sigant001   Ready     master    50m       v1.10.1
sigant002   Ready     <none>    33m       v1.10.1
sigant003   Ready     <none>    6m        v1.10.1
sigant004   Ready     <none>    1m        v1.10.1
sigant005   Ready     <none>    52s       v1.10.1
```

Controlling cluster from your personal computer.

```
# scp -i ~/.ssh/<key> fede@192.168.2.108:/etc/kubernetes/admin.conf . # You may need to grant temporary read permissions.
# kubectl --kubeconfig ./admin.conf proxy
```

Set up the dashboard https://github.com/kubernetes/dashboard#kubernetes-dashboard , this is not meant for a production system, please see official documentation for more details.

Follow instructions [11] https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/ .

Run the following in the master server:

```
# kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
rolebinding.rbac.authorization.k8s.io "kubernetes-dashboard-minimal" created
deployment.apps "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

From https://github.com/kubernetes/dashboard/wiki/Access-control

```
(Content of ./dashboard-admin.yaml)

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

```
# kubectl create -f ./dashboard-admin.yaml 
clusterrolebinding.rbac.authorization.k8s.io "kubernetes-dashboard" created
```

Start the proxy in your personal computer:

```
# kubectl --kubeconfig ./admin.conf proxy
```

Then connect to:

```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/.
```

And click the *Skip* button to access the dashboard.

#### Upgrading kubernetes.

Read release notes instructions [17].

```
(Master server)

# export VERSION=$(curl -sSL https://dl.k8s.io/release/stable.txt)
# export ARCH=amd64
# curl -sSL https://dl.k8s.io/release/${VERSION}/bin/linux/${ARCH}/kubeadm > /usr/bin/kubeadm
# chmod a+rx /usr/bin/kubeadm
# kubeadm version
# kubeadm upgrade plan
# kubeadm upgrade apply v1.9.1
```

Check Container Network Interface (CNI) provider instructions for upgrade (if any).

In the master server, for each node:

```
# kubectl drain sigant002 --ignore-daemonsets --delete-local-data --force
# kubectl drain sigant003 --ignore-daemonsets --delete-local-data --force
# kubectl drain sigant004 --ignore-daemonsets --delete-local-data --force
# kubectl drain sigant005 --ignore-daemonsets --delete-local-data --force
# kubectl drain sigant001 --ignore-daemonsets --delete-local-data --force
```

In each server:

```
# apt-get update
# apt-get upgrade
# systemctl status kubelet
```

In the master server:

```
# kubectl uncordon sigant001
# kubectl uncordon sigant002
# kubectl uncordon sigant003
# kubectl uncordon sigant004
# kubectl uncordon sigant005
# kubectl get nodes
```

#### Tearing down nodes (servers).

Follow kubernetes instructions [6].


#### Setting up GlusterFS.

GlusterFS (https://www.gluster.org/) is an open source distributed file system, it can provide a storage solution for persistent volumes. In this example three 2 TB HDDs are present in the cluster.

*Note:* You can also use loop devices to experiment but for simplicity it will not be included in this document.

Ensure that each server has the  */etc/modules* file with the following content:

```
dm_thin_pool
dm_snapshot
dm_mirror
```

Install *nfs-common* in all servers:

```
# apt-get install nfs-common
```

Use *modprobe* to load each module if necessary.

Ensure TDP and UDP ports are open:

```
(In each server)

iptables -A INPUT -p tcp --dport 24007:24008 -j ACCEPT
iptables -A INPUT -p udp --dport 24007:24008 -j ACCEPT
iptables -A INPUT -p tcp --dport 49152:49156 -j ACCEPT
iptables -A INPUT -p udp --dport 49152:49156 -j ACCEPT
```

Download https://github.com/gluster/gluster-kubernetes as a zip file to get necessary files and samples.

Erase the filesystem on each HDD that will be used for network storage, be careful and make sure you run this command with the right device:

```
( Server 1 )
# wipefs -a /dev/sdb
/dev/sdb: 8 bytes were erased at offset 0x00000218 (LVM2_member): 4c 56 4d 32 20 30 30 31

( Server 2 )
# wipefs -a /dev/sdc
/dev/sdc: 8 bytes were erased at offset 0x00000218 (LVM2_member): 4c 56 4d 32 20 30 30 31

( Server 3 )
# wipefs -a /dev/sdb
/dev/sdb: 8 bytes were erased at offset 0x00000218 (LVM2_member): 4c 56 4d 32 20 30 30 31
```

My *topology.json* file:

```
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "sigant001"
              ],
              "storage": [
                "192.168.2.108"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "sigant003"
              ],
              "storage": [
                "192.168.2.102"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdc"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "sigant005"
              ],
              "storage": [
                "192.168.2.116"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        }
      ]
    }
  ]
}
```

Example *lsblk* command output:

```
# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  1.8T  0 disk 
├─sda1   8:1    0  512M  0 part /boot/efi
├─sda2   8:2    0  1.8T  0 part /
└─sda3   8:3    0  1.9G  0 part 
sdb      8:16   0  3.7T  0 disk 
```

*Note:* *dmsetup* is a Command utility to manage volumes, see man page for instructions it could be useful if you need to make changes or remove volumes.

Install *glusterfs* in all servers:

```
# apt-get install glusterfs-common
# apt-get install glusterfs-client

```

Install (run in master server):

```
# cd path/to/glusterfs/files # Downloaded in a previous step.
# cd deploy/
# mv path/to/topology.json .
# ./gk-deploy -g 

Welcome to the deployment tool for GlusterFS on Kubernetes and OpenShift.

Before getting started, this script has some requirements of the execution
environment and of the container platform that you should verify.

The client machine that will run this script must have:
 * Administrative access to an existing Kubernetes or OpenShift cluster
 * Access to a python interpreter 'python'

Each of the nodes that will host GlusterFS must also have appropriate firewall
rules for the required GlusterFS ports:
 * 2222  - sshd (if running GlusterFS in a pod)
 * 24007 - GlusterFS Management
 * 24008 - GlusterFS RDMA
 * 49152 to 49251 - Each brick for every volume on the host requires its own
   port. For every new brick, one new port will be used starting at 49152. We
   recommend a default range of 49152-49251 on each host, though you can adjust
   this to fit your needs.

The following kernel modules must be loaded:
 * dm_snapshot
 * dm_mirror
 * dm_thin_pool

For systems with SELinux, the following settings need to be considered:
 * virt_sandbox_use_fusefs should be enabled on each node to allow writing to
   remote GlusterFS volumes

In addition, for an OpenShift deployment you must:
 * Have 'cluster_admin' role on the administrative account doing the deployment
 * Add the 'default' and 'router' Service Accounts to the 'privileged' SCC
 * Have a router deployed that is configured to allow apps to access services
   running in the cluster

Do you wish to proceed with deployment?

[Y]es, [N]o? [Default: Y]: Y
Using Kubernetes CLI.
Using namespace "default".
Checking for pre-existing resources...
  GlusterFS pods ... not found.
  deploy-heketi pod ... not found.
  heketi pod ... not found.
  gluster-s3 pod ... not found.
Creating initial resources ... serviceaccount "heketi-service-account" created
clusterrolebinding.rbac.authorization.k8s.io "heketi-sa-view" created
clusterrolebinding.rbac.authorization.k8s.io "heketi-sa-view" labeled
OK
node "sigant001" labeled
node "sigant003" labeled
node "sigant005" labeled
daemonset.extensions "glusterfs" created
Waiting for GlusterFS pods to start ... OK
secret "heketi-config-secret" created
secret "heketi-config-secret" labeled
service "deploy-heketi" created
deployment.extensions "deploy-heketi" created
Waiting for deploy-heketi pod to start ... OK
Flag --show-all has been deprecated, will be removed in an upcoming release
Creating cluster ... ID: 96fe7dad50d8970929f82a1cdaf08fb3
Allowing file volumes on cluster.
Allowing block volumes on cluster.
Creating node sigant001 ... ID: 7a5d7cdcc597e21e55eef2851f238ec1
Adding device /dev/sdb ... OK
Creating node sigant003 ... ID: 956b3f62c55d295452d5305a44311ca7
Adding device /dev/sdc ... OK
Creating node sigant005 ... ID: 58c398b16661ee91444094ebf055cb85
Adding device /dev/sdb ... OK
heketi topology loaded.
Saving /tmp/heketi-storage.json
secret "heketi-storage-secret" created
endpoints "heketi-storage-endpoints" created
service "heketi-storage-endpoints" created
job.batch "heketi-storage-copy-job" created
service "heketi-storage-endpoints" labeled
pod "deploy-heketi-76784cd777-bgfw8" deleted
service "deploy-heketi" deleted
deployment.apps "deploy-heketi" deleted
job.batch "heketi-storage-copy-job" deleted
secret "heketi-storage-secret" deleted
service "heketi" created
deployment.extensions "heketi" created
Waiting for heketi pod to start ... OK

heketi is now running and accessible via http://10.45.0.0:8080 . To run
administrative commands you can install 'heketi-cli' and use it as follows:

  # heketi-cli -s http://10.45.0.0:8080 --user admin --secret '<ADMIN_KEY>' cluster list

You can find it at https://github.com/heketi/heketi/releases . Alternatively,
use it from within the heketi pod:

  # /usr/bin/kubectl -n default exec -i <HEKETI_POD> -- heketi-cli -s http://localhost:8080 --user admin --secret '<ADMIN_KEY>' cluster list

For dynamic provisioning, create a StorageClass similar to this:

---
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.45.0.0:8080"


Deployment complete!

```

Create a file *gluster_storage_class.yaml* for the storage class:

```
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.45.0.0:8080"
```

```
# kubectl create -f ./gluster_storage_class.yaml
storageclass "glusterfs-storage" created
```

Test heketi endpoint:

```
# curl http://10.45.0.0:8080/hello
Hello from Heketi
```

Download heketi-client from https://github.com/heketi/heketi and extract with *tar -xvzf <filename>.tar.gz*.

```
# ./heketi-client/bin/heketi-cli -s  http://10.45.0.0:8080 cluster list
Clusters:
Id:96fe7dad50d8970929f82a1cdaf08fb3
```
```
# ./heketi-client/bin/heketi-cli -s  http://10.45.0.0:8080 volume list
Id:f27f5dc7e274fe08e06cffa68835efa0    Cluster:96fe7dad50d8970929f82a1cdaf08fb3    Name:heketidbstorage
```

Example *lsblk* output:

```
# lsblk
NAME                                                                              MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                                                                                 8:0    0 298.1G  0 disk 
├─sda1                                                                              8:1    0 294.2G  0 part /
├─sda2                                                                              8:2    0     1K  0 part 
└─sda5                                                                              8:5    0   3.9G  0 part 
sdb                                                                                 8:16   0   3.7T  0 disk 
├─vg_83638125faedb097f12f9ade57759de4-tp_d40f41e976890cbe98ee51cf135fee13_tmeta   254:0    0    12M  0 lvm  
│ └─vg_83638125faedb097f12f9ade57759de4-tp_d40f41e976890cbe98ee51cf135fee13-tpool 254:2    0     2G  0 lvm  
│   ├─vg_83638125faedb097f12f9ade57759de4-tp_d40f41e976890cbe98ee51cf135fee13     254:3    0     2G  0 lvm  
│   └─vg_83638125faedb097f12f9ade57759de4-brick_d40f41e976890cbe98ee51cf135fee13  254:4    0     2G  0 lvm  
└─vg_83638125faedb097f12f9ade57759de4-tp_d40f41e976890cbe98ee51cf135fee13_tdata   254:1    0     2G  0 lvm  
  └─vg_83638125faedb097f12f9ade57759de4-tp_d40f41e976890cbe98ee51cf135fee13-tpool 254:2    0     2G  0 lvm  
    ├─vg_83638125faedb097f12f9ade57759de4-tp_d40f41e976890cbe98ee51cf135fee13     254:3    0     2G  0 lvm  
    └─vg_83638125faedb097f12f9ade57759de4-brick_d40f41e976890cbe98ee51cf135fee13  254:4    0     2G  0 lvm  
```

And here is a screenshot of the dashboard:

![Kubernetes dashboard]({{ "/assets/kubernetes_dashboard.png" | absolute_url }})

The end.

## References

1. http://osxdaily.com/2015/06/05/copy-iso-to-usb-drive-mac-os-x-command/
2. https://www.debian.org/releases/stable/amd64/
3. https://wiki.debian.org/SSH#Installation_of_the_server
4. https://www.debian.org/doc/manuals/securing-debian-howto/ch-sec-services.en.html
5. https://wiki.debian.org/UnattendedUpgrades
6. https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
7. https://kubernetes.io/docs/setup/independent/install-kubeadm/
8. https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux
9. https://docs.docker.com/engine/installation/linux/docker-ce/debian/#set-up-the-repository
10. https://github.com/kubernetes/kubernetes/issues/53333
11. https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
12. https://github.com/kubernetes/dashboard/wiki/Access-control
13. https://www.kevinhooke.com/2017/10/20/deploying-kubernetes-dashboard-to-a-kubeadm-created-cluster/
