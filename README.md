### Prerequisites

 - Install VirtualBox (6.0.4)
 - Download Debian netinstall ISO (9.6)
 - Create NAT Network (NatNetwork)
 - Create VM with **Adapter 1**(NatNetwork), **Adapter 2**(Bridged adapter)
 - Install Debian with automatic install (use url https://bit.ly/preseedly)

### Inside VM

#### Configure network

    antti@debian:~$ cat /etc/network/interfaces
    # This file describes the network interfaces available on your system
    # and how to activate them. For more information, see interfaces(5).
    
    source /etc/network/interfaces.d/*
    
    # The loopback network interface
    auto lo
    iface lo inet loopback
    
    # The primary network interface
    allow-hotplug enp0s3
    iface enp0s3 inet dhcp
    
    auto enp0s8
    iface enp0s8 inet static
        address 192.168.0.107/24
        netmask 255.255.255.0
        gateway 192.168.0.1

#### Initialize kubernetes cluster with flannel prerequisite

    sudo kubeadm init --pod-network-cidr=10.244.0.0/16

#### Add configuration for running kubectl as user

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

#### Use flannel as network driver

    sudo sysctl net.bridge.bridge-nf-call-iptables=1 #(optional?)
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

#### Taint master:

    kubectl taint nodes --all node-role.kubernetes.io/master-

### Route traffic from enp0s3 to enp0s8

    root@debian:~# iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 32192 -j DNAT --to 10.0.2.4:32192
    root@debian:~# iptables -A FORWARD -i enp0s8 -p tcp --dport 32192 -d 10.0.2.4 -j ACCEPT

Use this [tutorial](https://supergiant.io/blog/using-traefik-as-ingress-controller-for-your-kubernetes-cluster/) as base.

If tutorial uses **minikube ip** then instead of
`$(minikube ip)` use `$(ip route get 1 | awk '{print $NF;exit}')`

### And Kubernetes service with ingress

##### Let’s first create a new ServiceAccount to provide Traefik with the identity in your cluster. 

    kubectl create  -f  traefik-service-acc.yaml
##### Next, let’s create a ClusterRole with a set of permissions which will be applied to the Traefik ServiceAccount.

    kubectl create  -f  traefik-cr.yaml
#####  Finally, to enable these permissions, we should bind the ClusterRole to the Traefik ServiceAccount.

    kubectl create  -f  traefik-crb.yaml
##### Next, we will deploy Traefik to your Kubernetes cluster. We’ll be using the Deployment manifest to deploy Traefik to your Kubernetes cluster.

    kubectl create  -f  traefik-deployment.yaml
##### Now, let’s check to see if the Traefik Pods were successfully created:

    kubectl  --namespace=kube-system get pods
##### Let’s create a Service to access Traefik Ingress Pods. To this end, we need a Service that exposes two NodePorts.

    kubectl create  -f  traefik-svc.yaml
    kubectl describe svc traefik-ingress-service  --namespace=kube-system
    
##### Now we have Traefik as the Ingress Controller in the Kubernetes cluster. However, we still need to define the Ingress resource and a Service that exposes Traefik Web UI.

    kubectl create  -f  traefik-webui-svc.yaml
    kubectl describe svc traefik-web-ui  --namespace=kube-system
##### Next, we need to create an Ingress resource pointing to the Traefik Web UI backend.

    kubectl create  -f  traefik-ingress.yaml

To make the Traefik Web UI accessible in the browser via the traefik-ui.vbox , we need to add a new entry to our /etc/hosts file. *(Or we don't need.)*
You should now be able to visit http://enp0s8/:NodePort in the browser and view the Traefik web UI.

##### Let’s demonstrate how Traefik Ingress Controller can be used to set up name-based routing for a list of frontends.

    kubectl create  -f  animals-deployment.yaml
##### Create a Service for each Deployment to make the Pods accessible:

    kubectl create  -f  animals-svc.yaml
##### Finally, let’s create an Ingress with three frontend-backend pairs for each Deployment. bear.vbox ,moose.vbox , and hare.vbox will be our frontends pointing to corresponding backend Services.

    kubectl create  -f  animals-ingress.yaml
