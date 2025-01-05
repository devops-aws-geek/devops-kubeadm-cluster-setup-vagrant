VAGRANTFILE_API_VERSION = "2"

$k8s_master = <<ENDSCRIPT
  set -euxo pipefail

  # Kubernetes Variable Declaration
  KUBERNETES_VERSION="v1.30"
  CRIO_VERSION="v1.30"
  KUBERNETES_INSTALL_VERSION="1.30.0-1.1"

  # Disable swap
  sudo swapoff -a

  # Keeps the swap off during reboot
  (crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
  sudo apt-get update -y

  # Create the .conf file to load the modules at bootup
  echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf

  sudo modprobe overlay
  sudo modprobe br_netfilter

  # Sysctl params required by setup, params persist across reboots
  echo -e "net.bridge.bridge-nf-call-iptables = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\nnet.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/k8s.conf


  # Apply sysctl params without reboot
  sudo sysctl --system

  sudo apt-get update -y
  sudo apt-get install -y apt-transport-https ca-certificates curl gpg

  # Install CRI-O Runtime
  sudo apt-get update -y
  sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates
  sudo mkdir -p /etc/apt/keyrings
  curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
      gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

  echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
      tee /etc/apt/sources.list.d/cri-o.list

  sudo apt-get update -y
  sudo apt-get install -y cri-o

  sudo systemctl daemon-reload
  sudo systemctl enable crio --now
  sudo systemctl start crio.service

  echo "CRI runtime installed successfully"

  # Install kubelet, kubectl, and kubeadm
  curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
      gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

  echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
      tee /etc/apt/sources.list.d/kubernetes.list

  sudo apt-get update -y
  sudo apt-get install -y kubelet="$KUBERNETES_INSTALL_VERSION" kubectl="$KUBERNETES_INSTALL_VERSION" kubeadm="$KUBERNETES_INSTALL_VERSION"

  # Prevent automatic updates for kubelet, kubeadm, and kubectl
  sudo apt-mark hold kubelet kubeadm kubectl

  sudo apt-get update -y

  # Install jq, a command-line JSON processor
  sudo apt-get install -y jq

  # Retrieve the local IP address of the eth0 interface and set it for kubelet
  local_ip="$(ip --json addr show eth1 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"
  echo $local_ip
  # Write the local IP address to the kubelet default configuration file
  echo "KUBELET_EXTRA_ARGS=--node-ip=$local_ip" | sudo tee /etc/default/kubelet

  
  PUBLIC_IP_ACCESS="false"
  NODENAME=$(hostname -s)
  POD_CIDR="192.168.0.0/16"

  # Pull required images

  sudo kubeadm config images pull

  # Initialize kubeadm based on PUBLIC_IP_ACCESS

  if [[ "$PUBLIC_IP_ACCESS" == "false" ]]; then
      
      MASTER_PRIVATE_IP=$(ip addr show eth1 | awk '/inet / {print $2}' | cut -d/ -f1)
      sudo kubeadm init --apiserver-advertise-address="$MASTER_PRIVATE_IP" --apiserver-cert-extra-sans="$MASTER_PRIVATE_IP" --pod-network-cidr="$POD_CIDR" --node-name "$NODENAME" --ignore-preflight-errors Swap

  elif [[ "$PUBLIC_IP_ACCESS" == "true" ]]; then

      MASTER_PUBLIC_IP=$(curl ifconfig.me && echo "")
      sudo kubeadm init --control-plane-endpoint="$MASTER_PUBLIC_IP" --apiserver-cert-extra-sans="$MASTER_PUBLIC_IP" --pod-network-cidr="$POD_CIDR" --node-name "$NODENAME" --ignore-preflight-errors Swap

  else
      echo "Error: MASTER_PUBLIC_IP has an invalid value: $PUBLIC_IP_ACCESS"
      exit 1
  fi

  # Configure kubeconfig
  mkdir -p "$HOME"/.kube
  sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
  sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config

  # Install Claico Network Plugin Network 

  kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
ENDSCRIPT

$k8s_worker = <<SCRIPT
  set -euxo pipefail

  # Kubernetes Variable Declaration
  KUBERNETES_VERSION="v1.30"
  CRIO_VERSION="v1.30"
  KUBERNETES_INSTALL_VERSION="1.30.0-1.1"

  # Disable swap
  sudo swapoff -a

  # Keeps the swap off during reboot
  (crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
  sudo apt-get update -y

  # Create the .conf file to load the modules at bootup
  echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf

  sudo modprobe overlay
  sudo modprobe br_netfilter

  # Sysctl params required by setup, params persist across reboots
  echo -e "net.bridge.bridge-nf-call-iptables = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\nnet.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/k8s.conf


  # Apply sysctl params without reboot
  sudo sysctl --system

  sudo apt-get update -y
  sudo apt-get install -y apt-transport-https ca-certificates curl gpg

  # Install CRI-O Runtime
  sudo apt-get update -y
  sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates
  sudo mkdir -p /etc/apt/keyrings
  curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
      gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

  echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
      tee /etc/apt/sources.list.d/cri-o.list

  sudo apt-get update -y
  sudo apt-get install -y cri-o

  sudo systemctl daemon-reload
  sudo systemctl enable crio --now
  sudo systemctl start crio.service

  echo "CRI runtime installed successfully"

  # Install kubelet, kubectl, and kubeadm
  curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
      gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

  echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
      tee /etc/apt/sources.list.d/kubernetes.list

  sudo apt-get update -y
  sudo apt-get install -y kubelet="$KUBERNETES_INSTALL_VERSION" kubectl="$KUBERNETES_INSTALL_VERSION" kubeadm="$KUBERNETES_INSTALL_VERSION"

  # Prevent automatic updates for kubelet, kubeadm, and kubectl
  sudo apt-mark hold kubelet kubeadm kubectl

  sudo apt-get update -y

  # Install jq, a command-line JSON processor
  sudo apt-get install -y jq

  # Retrieve the local IP address of the eth0 interface and set it for kubelet
  local_ip="$(ip --json addr show eth1 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"

  # Write the local IP address to the kubelet default configuration file
  echo "KUBELET_EXTRA_ARGS=--node-ip=$local_ip" | sudo tee /etc/default/kubelet
SCRIPT

Vagrant.configure("2") do |config|
    config.vm.define "master" do |master|
      master.vm.box_download_insecure = true    
      master.vm.box = "bento/ubuntu-22.04"        ## For ubuntu 18.04 use - hashicorp/bionic64
      master.vm.network "private_network", ip: "100.0.0.4"
      master.vm.hostname = "master"
      master.vm.provider "virtualbox" do |v|
        v.name = "master"
        v.memory = 2048
        v.cpus = 2
        v.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
        v.customize ["modifyvm", :id, "--uartmode1", "file", File::NULL]
      end
      master.vm.provision "shell", inline: $k8s_master
    end
  
    config.vm.define "worker" do |worker|
      worker.vm.box_download_insecure = true 
      worker.vm.box = "bento/ubuntu-22.04"        ## For ubuntu 18.04 use - hashicorp/bionic64
      worker.vm.network "private_network", ip: "100.0.0.5"
      worker.vm.hostname = "worker"
      worker.vm.provider "virtualbox" do |v|
        v.name = "worker"
        v.memory = 2048
        v.cpus = 1
        v.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
        v.customize ["modifyvm", :id, "--uartmode1", "file", File::NULL]
      end
      worker.vm.provision "shell", inline: $k8s_worker
    end
  
  end
  