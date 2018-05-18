### * VMs:
2a0a:e5c0:2:2:400:84ff:fe41:f250/64  kube-master
2a0a:e5c0:2:2:400:84ff:fe41:f251/64  kube-node-1
2a0a:e5c0:2:2:400:84ff:fe41:f252/64  kube-node-2
2a0a:e5c0:2:2:400:84ff:fe41:f24d/64  kube-nat64-dns64

### Subnets:
2a0a:e5c0:2:9:f251::/80   kube-node-1 subnet
2a0a:e5c0:2:9:f252::/80   kube-node-2 subnet
2a0a:e5c0:2:9:1234::/123   cluster service subnet
2a0a:e5c0:2:10::/96       Prefix used inside the cluster for packets requiring NAT64 translation

### Routing table:

2a0a:e5c0:2:9:f251::/80  via 2a0a:e5c0:2:2:400:84ff:fe41:f251
2a0a:e5c0:2:9:f252::/80  via 2a0a:e5c0:2:2:400:84ff:fe41:f252
2a0a:e5c0:2:9:1234::/123  via 2a0a:e5c0:2:2:400:84ff:fe41:f250
2a0a:e5c0:2:10::/96   via 2a0a:e5c0:2:2:400:84ff:fe41:f24d

### update hosts file 
2a0a:e5c0:2:2:400:84ff:fe41:f250  kube-master
2a0a:e5c0:2:2:400:84ff:fe41:f251  kube-node-1
2a0a:e5c0:2:2:400:84ff:fe41:f252  kube-node-2
2a0a:e5c0:2:2:400:84ff:fe41:f24d  kube-nat64-dns64



### add routes:
ip -6 route add 2a0a:e5c0:2:9:f252::/80  via 2a0a:e5c0:2:2:400:84ff:fe41:f252
ip -6 route add 2a0a:e5c0:2:9:f251::/80 via 2a0a:e5c0:2:2:400:84ff:fe41:f251
ip -6 route add 2a0a:e5c0:2:9:1234::/123  via 2a0a:e5c0:2:2:400:84ff:fe41:f250
ip -6 route add  2a0a:e5c0:2:10::/96 via 2a0a:e5c0:2:2:400:84ff:fe41:f24d

### update sysctl
sysctl -w net.ipv6.conf.all.forwarding=1
sysctl -w net.ipv6.conf.all.forwarding=1

### make sure it is updated

sysctl -a |grep net.ipv6.conf.all.forwarding
sysctl -a |grep net.bridge.bridge-nf-call-ip6tables


### install docker 

apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
add-apt-repository "deb https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
apt-get update && apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')


### install packages:

apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl

### start services
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet


### update the default kube-dns
KUBE_DNS_SVC_IPV6=2a0a:e5c0:2:9:1234::a
sudo sed -i "s/--cluster-dns=.* /--cluster-dns=$KUBE_DNS_SVC_IPV6 /" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf



####On kube-node-1, create a CNI network plugin using pod subnet fd00:101::/64:


sudo -i
# Backup any existing CNI netork config files to home dir
mv /etc/cni/net.d/* $HOME
cat <<EOT > 10-bridge-v6.conf
{
  "cniVersion": "0.3.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cbr0",
  "isDefaultGateway": true,
  "ipMasq": true,
  "hairpinMode": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [
        {
          "subnet": "2a0a:e5c0:2:9:f251::/80",
          "gateway": "2a0a:e5c0:2:9:f251::1"
        }
      ]
    ]
  }
}
EOT
exit



### On kube-node-2, create a CNI network plugin using pod subnet fd00:102::/64:


sudo -i
# Backup any existing CNI netork config files to home dir
mv /etc/cni/net.d/* $HOME
cat <<EOT > 10-bridge-v6.conf
{
  "cniVersion": "0.3.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cbr0",
  "isDefaultGateway": true,
  "ipMasq": true,
  "hairpinMode": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [
        {
          "subnet": "2a0a:e5c0:2:9:f252::/80",
          "gateway": "2a0a:e5c0:2:9:f252::1"
        }
      ]
    ]
  }
}
EOT
exit



### On the master node, create a kubeadm config file as follows:

cat << EOT > kubeadm_v6.cfg
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
api:
  advertiseAddress: 2a0a:e5c0:2:2:400:84ff:fe41:f250
networking:
  serviceSubnet: 2a0a:e5c0:2:9:1234::/123
nodeName: kube-master
EOT


### On the Kubernetes master node and each minion node:

sed -i "s/nameserver.*/nameserver 2a0a:e5c0:2:2:400:84ff:fe41:f24d/" /etc/resolv.conf


## Run kubeadm init on master node
## On the master node, run:

sudo -i
kubeadm init --config=kubeadm_v6.cfg




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

  kubeadm join [2a0a:e5c0:2:2:400:84ff:fe41:f251]:6443 --token 5hd9d7.1dkjbi58rgy1j4vw --discovery-token-ca-cert-hash sha256:de5f5eb29ec712354c588d361c25230180f6ee97249b920ddd7fb695be6da55c

