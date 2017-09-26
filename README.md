# Kubernetes-Cluster
Kubernetes is an open-source container orchestration framework which was built upon the learnings of Google. It enables you to run applications using containers in a production ready-cluster. Kubernetes has many moving parts and there are countless ways to configure its pieces - from the various system components, network transport drivers, CLI utilities not to mention applications and workloads.

In this blog post we'll install Kubernetes 1.6 on a bare-metal machine with Ubuntu 16.04 in about 10 minutes. At the end you'll be able to start learning how to interact with Kubernetes via its CLI kubectl.

Kubernetes overview:

Pre-reqs
I suggest using Packet for running through the tutorial which will offer a bare-metal host - you can also run through this on a VM or your own PC if you're running Ubuntu 16.04 as your OS.

Head over to Packet.net and create a new project. For this example we can take advantage of the Type 0 host which gives you 4x Atom cores and 8GB of RAM for $0.05/hour.

When you provision the host make sure you pick Ubuntu 16.04 as the OS. Unlike Docker Swarm - Kubernetes is best paired with older versions of Docker. Fortunately the Ubuntu apt repository contains Docker 1.12.6.

Install Docker
$ apt-get update && apt-get install -qy docker.io
Don't upgrade the Docker version on this host. You can still build images in your CI pipe-line or on your laptop with newer versions

Installation
Install Kubernetes apt repo
$ apt-get update && apt-get install -y apt-transport-https
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
OK

$ echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
Now update your packages list with apt-get update.

Install kubelet, kubeadm and kubernetes-cni
The kubelet is responsible for running containers on your hosts. kubeadm is a convenience utility to configure the various components that make up a working cluster and kubernetes-cni represents the networking components.

CNI stands for Container Networking Interface which is a spec that defines how network drivers should interact with Kubernetes

$ apt-get update
$ apt-get install -y kubelet kubeadm kubernetes-cni
Initialize your cluster with kubeadm
From the docs:

kubeadm aims to create a secure cluster out of the box via mechanisms such as RBAC.

Docker Swarm provides an overlay networking driver by default - but with kubeadm this decision is left to us. The team are still working on updating their instructions - so I'll show you how to use the most similar driver to Docker's overlay driver (flannel by CoreOS).

Flannel

Flannel provides a software defined network (SDN) using the Linux kernel's overlay and ipvlan modules.

Packet provides two networks for its machines - the first is a datacenter link which goes between your hosts in a specific region and project and the second faces the public Internet. There is no default firewall - if you want to lock things down you'll have to configure iptables or ufw rules manually.

You can find your private/datacenter IP address through ifconfig:

root@kubeadm:~# ifconfig bond0:0  
bond0:0   Link encap:Ethernet  HWaddr 0c:c4:7a:e5:48:d4  
          inet addr:10.80.75.9  Bcast:255.255.255.255  Mask:255.255.255.254
          UP BROADCAST RUNNING MASTER MULTICAST  MTU:1500  Metric:1
We'll now use the internal IP address to broadcast the Kubernetes API - rather than the Internet-facing address.

$ kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.80.75.9 --skip-preflight-checks --kubernetes-version stable-1.6
--pod-network-cidr is needed for the flannel driver and specifies an address space for containers
--apiserver-advertise-address determines which IP address Kubernetes should advertise its API server on
--skip-preflight-checks allows kubeadm to check the host kernel for required features. We cannot run this since Packet hosts have had the kernel meta-data removed already
--kubernetes-version stable-1.6 this pins the version of the cluster to 1.6, but if you want to use Kubernetes 1.7 for example - then just leave off this final flag.
Here's the output we got:

[init] Using Kubernetes version: v1.6.6
[init] Using Authorization mode: RBAC
[preflight] Skipping pre-flight checks
[certificates] Generated CA certificate and key.
[certificates] Generated API server certificate and key.
[certificates] API Server serving cert is signed for DNS names [kubeadm kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.80.75.9]
[certificates] Generated API server kubelet client certificate and key.
[certificates] Generated service account token signing key and public key.
[certificates] Generated front-proxy CA certificate and key.
[certificates] Generated front-proxy client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 36.795038 seconds
[apiclient] Waiting for at least one node to register
[apiclient] First node has registered after 3.508700 seconds
[token] Using token: 02d204.3998037a42ac8108
[apiconfig] Created RBAC rules
[addons] Created essential addon: kube-proxy
[addons] Created essential addon: kube-dns

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  sudo cp /etc/kubernetes/admin.conf $HOME/
  sudo chown $(id -u):$(id -g) $HOME/admin.conf
  export KUBECONFIG=$HOME/admin.conf

You should now deploy a pod network to the cluster.  
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:  
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node  
as root:

  kubeadm join --token 02d204.3998037a42ac8108 10.80.75.9:6443
Configure an unprivileged user-account
Packet's Ubuntu installation ships without an unprivileged user-account, so let's add one.

# useradd packet -G sudo -m -s /bin/bash
# passwd packet
Configure environmental variables as the new user
You can now configure your environment with the instructions at the end of the init message above.

Switch into the new user account with: sudo su packet.

$ cd $HOME
$ sudo whoami

$ sudo cp /etc/kubernetes/admin.conf $HOME/
$ sudo chown $(id -u):$(id -g) $HOME/admin.conf
$ export KUBECONFIG=$HOME/admin.conf

$ echo "export KUBECONFIG=$HOME/admin.conf" | tee -a ~/.bashrc
Apply your pod network (flannel)
We will now apply configuration to the cluster using kubectl and two entries from the flannel docs:

$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
clusterrole "flannel" created  
clusterrolebinding "flannel" created

$ kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-legacy.yml

serviceaccount "flannel" created  
configmap "kube-flannel-cfg" created  
daemonset "kube-flannel-ds" created  
We've now configured networking for pods.

Allow a single-host cluster
Kubernetes is about multi-host clustering - so by default containers cannot run on master nodes in the cluster. Since we only have one node - we'll taint it so that it can run containers for us.

$ kubectl taint nodes --all node-role.kubernetes.io/master-
An alternative at this point would be to provision a second machine and use the join token from the output of kubeadm.

Check it's working
Many of the Kubernetes components run as containers on your cluster in a hidden namespace called kube-system. You can see whether they are working like this:

$ kubectl get all --namespace=kube-system

NAME                                 READY     STATUS    RESTARTS   AGE  
po/etcd-kubeadm                      1/1       Running   0          12m  
po/kube-apiserver-kubeadm            1/1       Running   0          12m  
po/kube-controller-manager-kubeadm   1/1       Running   0          13m  
po/kube-dns-692378583-kqvdd          3/3       Running   0          13m  
po/kube-flannel-ds-w9xvp             2/2       Running   0          1m  
po/kube-proxy-4vgwp                  1/1       Running   0          13m  
po/kube-scheduler-kubeadm            1/1       Running   0          13m

NAME           CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE  
svc/kube-dns   10.96.0.10   <none>        53/UDP,53/TCP   14m

NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE  
deploy/kube-dns   1         1         1            1           14m

NAME                    DESIRED   CURRENT   READY     AGE  
rs/kube-dns-692378583   1         1         1         13m  
As you can see all of the services are in a state of Running which indicates a healthy cluster. If these components are still being downloaded from the Internet they may appear as not started.

Run a container
You can now run a container on your cluster. Kubernetes organises containers into Pods which share a common IP address, are always scheduled on the same node (host) and can share storage volumes.

First check you have no pods (containers) running with:

$ kubectl get pods
Now use kubectl run to deploy a container. We'll deploy a Node.js and Express.js microservice that generates GUIDs over HTTP.

This code was originally written for a Docker Swarm tutorial and you can find the source-code there - Scale a real microservice with Docker 1.12 Swarm Mode

$ kubectl run guids --image=alexellis2/guid-service:latest --port 9000
deployment "guids" created  
You'll now be able to see the Name assigned to the new Pod and when it gets started up:

$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE  
guids-2617315942-lzwdh   0/1       Pending   0          11s  
Use the Name to check on the pod:

$ kubectl describe pod guids-2617315942-lzwdh
...
Pulling            pulling image "alexellis2/guid-service:latest"  
...
Once running you can get the IP address and use curl to generate GUIDs:

$ kubectl describe pod guids-2617315942-lzwdh | grep IP:
IP:        10.244.0.3

$ curl http://10.244.0.3:9000/guid ; echo
{"guid":"4659819e-cf00-4b45-99d1a9f81bdcf6ae","container":"guids-2617315942-lzwdh"}

$ curl http://10.244.0.3:9000/guid ; echo
{"guid":"1604b4cb-88d2-49e2-bd38-73b589da0469","container":"guids-2617315942-lzwdh"}
If you want to see the logs for your Pod type in:

$ kubectl logs guids-2617315942-lzwdh
listening on port 9000  
A very useful feature for debugging containers is the ability to attach to the console via a shell to execute ad-hoc commands in the container:

$ kubectl exec -t -i guids-2617315942-lzwdh sh
/ # head -n3 /etc/os-release
NAME="Alpine Linux"  
ID=alpine  
VERSION_ID=3.5.2  
/ # exit
View the Dashboard UI
The Kubernetes dashboard can be deployed as another Pod, which we can then view on our local machine. Since we did not expose Kubernetes over the Internet we'll use an SSH tunnel to view the site.

$ kubectl create -f https://git.io/kube-dashboard
$ kubectl proxy
Starting to serve on 127.0.0.1:8001  
Now tunnel to the Packet host and navigate to http://localhost:8001/ui/ in a web-browser.

$ ssh -L 8001:127.0.0.1:8001 -N


For more information on the Dashboard check it out on Github.
