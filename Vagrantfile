# -*- mode: ruby -*-
# vi: set ft=ruby :

# This script to install Kubernetes will get executed after we have provisioned the box 
$script = <<-EOF

# Load IPVS modules - https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4

# Install kubernetes
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF2 >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF2
apt-get update
apt-get install -y kubelet kubeadm kubectl

# kubelet requires swap off
swapoff -a

# keep swap off after reboot
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Get the IP address that VirtualBox has given this VM
IPADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
echo This VM has IP address $IPADDR

echo "KUBELET_EXTRA_ARGS=--node-ip=$IPADDR" > /etc/default/kubelet
EOF

$master_script = <<-EOF
# Set up Kubernetes
IPADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
NODENAME=$(hostname -s)
kubeadm config images pull
kubeadm init --apiserver-advertise-address=$IPADDR --apiserver-cert-extra-sans=$IPADDR --pod-network-cidr=10.244.0.0/16 | tee /vagrant/master.txt

# Set up admin creds for the vagrant user
echo Copying credentials to /home/vagrant...
sudo --user=vagrant mkdir -p /home/vagrant/.kube
cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

# install pod network add-on
export KUBECONFIG=/home/vagrant/.kube/config
# variant of https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
sudo -E --user=vagrant kubectl apply -f https://raw.githubusercontent.com/equick/kubernetes-vagrant/master/kube-flannel.yml
EOF

$node_script = <<-EOF3
KUBEJOIN=$(grep 'kubeadm join' /vagrant/master.txt | sed 's/.*kubeadm/kubeadm/')
eval $KUBEJOIN
EOF3

$ingress_script = <<EOF4
# https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/installation.md
kubectl apply -f https://raw.githubusercontent.com/equick/kubernetes-vagrant/master/nginxinc/kubernetes-ingress/master/install/common/ns-and-sa.yaml
kubectl apply -f https://raw.githubusercontent.com/equick/kubernetes-vagrant/master/nginxinc/kubernetes-ingress/master/install/common/default-server-secret.yaml
kubectl apply -f https://raw.githubusercontent.com/equick/kubernetes-vagrant/master/nginxinc/kubernetes-ingress/master/install/common/nginx-config.yaml
kubectl apply -f https://raw.githubusercontent.com/equick/kubernetes-vagrant/master/nginxinc/kubernetes-ingress/master/install/rbac/rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/equick/kubernetes-vagrant/master/nginxinc/kubernetes-ingress/master/install/daemon-set/nginx-ingress.yaml
EOF4

$msg = <<EOF5
echo "After the worker nodes come up run a test deployment:"
echo "kubectl apply -f https://raw.githubusercontent.com/equick/kubernetes-vagrant/master/deploy-svc-ingress.yaml"
echo "And curl http://test.example.com/kb"
EOF5


Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
  config.vm.network "private_network", type: "dhcp"
  config.vm.synced_folder ".", "/vagrant", disabled: false
  config.vm.provision "docker"
  config.vm.provision "shell", inline: $script

  config.vm.provider "virtualbox" do |vb|
    vb.cpus = 2
    vb.linked_clone = true
  end

  config.vm.define "master01" do |master01|
    master01.vm.hostname = "master01"
    config.vm.provider "virtualbox" do |vb|
      vb.name = "master01"
      vb.memory = 1536
    end
    master01.vm.provision "shell", inline: $master_script
    master01.vm.provision "shell", inline: $ingress_script, privileged: false
  end

  config.vm.define "worker01" do |worker01|
    worker01.vm.hostname = "worker01"
    config.vm.provider "virtualbox" do |vb|
      vb.name = "worker01"
      vb.memory = 1024
    end
    worker01.vm.provision "shell", inline: $node_script
  end

  config.vm.define "worker02" do |worker02|
    worker02.vm.hostname = "worker02"
    config.vm.provider "virtualbox" do |vb|
      vb.name = "worker02"
      vb.memory = 1024
    end
    worker02.vm.provision "shell", inline: $node_script
    worker02.vm.provision "shell", inline: $msg
  end

end
