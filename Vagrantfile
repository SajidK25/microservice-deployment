VAGRANTFILE_API_VERSION = "2"

NODES = {
  "k8s-master" => "192.168.56.10",
  "k8s-worker1" => "192.168.56.11",
  "k8s-worker2" => "192.168.56.12"
}

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end

  NODES.each do |name, ip|
    config.vm.define name do |node|
      node.vm.hostname = name
      node.vm.network "private_network", ip: ip
      node.vm.provision "shell", inline: <<-SHELL
        # Update and install prerequisites
        apt-get update
        apt-get install -y apt-transport-https ca-certificates curl gpg

        # Install containerd
        apt-get install -y containerd
        systemctl enable containerd
        systemctl start containerd

        # Configure containerd for Kubernetes
        mkdir -p /etc/containerd
        containerd config default > /etc/containerd/config.toml
        sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
        systemctl restart containerd

        # Load kernel modules for Kubernetes networking
        modprobe br_netfilter
        echo "br_netfilter" > /etc/modules-load.d/k8s.conf
        sysctl -w net.bridge.bridge-nf-call-iptables=1
        sysctl -w net.ipv4.ip_forward=1
        cat <<EOF >/etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
EOF
        sysctl --system

        # Disable swap
        swapoff -a
        sed -i '/ swap / s/^/#/' /etc/fstab

        # Install Kubernetes components
        mkdir -p /etc/apt/keyrings
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
          gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
        echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
          tee /etc/apt/sources.list.d/kubernetes.list
        apt-get update
        apt-get install -y kubelet kubeadm kubectl
        apt-mark hold kubelet kubeadm kubectl

        # Specific configuration for master node
        if [ "#{name}" = "k8s-master" ]; then
          # Initialize control plane with verbose output
          kubeadm init --apiserver-advertise-address=192.168.56.10 --pod-network-cidr=10.244.0.0/16 --v=5
          
          # Set up kubeconfig for the vagrant user
          mkdir -p /home/vagrant/.kube
          cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
          chown vagrant:vagrant /home/vagrant/.kube/config

          # Wait for API server to be ready
          export KUBECONFIG=/home/vagrant/.kube/config
          for i in {1..30}; do
            if kubectl get nodes >/dev/null 2>&1; then
              # Apply Flannel CNI
              kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
              break
            fi
            echo "Waiting for API server to be ready..."
            sleep 10
          done

          # Generate join command and save it for worker nodes
          kubeadm token create --print-join-command > /vagrant/join-command.sh
        fi

        # Specific configuration for worker nodes
        if [ "#{name}" != "k8s-master" ]; then
          # Wait for join command to be available
          for i in {1..30}; do
            if [ -f /vagrant/join-command.sh ]; then
              bash /vagrant/join-command.sh --v=5
              break
            fi
            echo "Waiting for join command..."
            sleep 10
          done
        fi
      SHELL
    end
  end
end