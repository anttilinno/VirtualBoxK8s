Install VirtualBox (6.0.2)
Download Debian (9.6)
Create NAT Network (NatNetwork)
Create VM with Adapter 1(NatNetwork), Adapter 2(Bridged adapter)
Install Debian with automatic install (use url https://bit.ly/preseedly)

sudo dhclient enp0s8 (secondary network device)
ip addr show dev ep0s8

Initialize kubernetes cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

Add configuration for running kubectl as user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Use flannel as network driver
sudo sysctl net.bridge.bridge-nf-call-iptables=1 <-- Is this necessary?
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

Taint master:
kubectl taint nodes --all node-role.kubernetes.io/master-

Route traffic from enp0s3 to enp0s8
root@debian:~# iptables -t nat -A PREROUTING -i enp0s8 -p tcp --dport 32192 -j DNAT --to 10.0.2.4:32192
root@debian:~# iptables -A FORWARD -i enp0s8 -p tcp --dport 32192 -d 10.0.2.4 -j ACCEPT

If tutorial uses minikube ip then instead of
$(minikube ip) use $(ip route get 1 | awk '{print $NF;exit}')

Use tutorial https://supergiant.io/blog/using-traefik-as-ingress-controller-for-your-kubernetes-cluster/ as base


