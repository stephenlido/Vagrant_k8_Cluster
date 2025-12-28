# Kubernetes Cluster - 1 Master + 2 Workers
# Works on Ubuntu Jammy (22.04+)

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.box_check_update = false

  # Base provider settings
  config.vm.provider "virtualbox" do |v|
    v.memory = 2048
    v.cpus = 2
  end

  # Kubernetes installation script (modern repo)
  kube_install_script = <<-SHELL
    set -e

    apt-get update -y
    apt-get install -y apt-transport-https ca-certificates curl gpg docker.io

    systemctl enable docker
    systemctl start docker

    # Add Kubernetes repo (new way)
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" > /etc/apt/sources.list.d/kubernetes.list

    apt-get update -y
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl
  SHELL

  # Master node
  config.vm.define "k8s-master" do |master|
    master.vm.hostname = "k8s-master"
    master.vm.network "private_network", ip: "192.168.56.10"

    master.vm.provision "shell", inline: <<-SHELL
      #{kube_install_script}

      kubeadm init --apiserver-advertise-address=192.168.56.10 --pod-network-cidr=10.244.0.0/16

      # Setup kubeconfig for vagrant
      mkdir -p /home/vagrant/.kube
      cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
      chown vagrant:vagrant /home/vagrant/.kube/config

      # Install Flannel CNI
      su - vagrant -c "kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"

      # Save join command
      kubeadm token create --print-join-command > /vagrant/join-command.sh
    SHELL
  end

  # Worker nodes
  (1..2).each do |i|
    config.vm.define "k8s-worker#{i}" do |node|
      node.vm.hostname = "k8s-worker#{i}"
      node.vm.network "private_network", ip: "192.168.56.1#{i}"

      node.vm.provision "shell", inline: <<-SHELL
        #{kube_install_script}
        if [ -f /vagrant/join-command.sh ]; then
          bash /vagrant/join-command.sh
        fi
      SHELL
    end
  end
end

