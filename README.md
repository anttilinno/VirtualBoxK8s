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
