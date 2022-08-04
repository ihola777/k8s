# Install kubernetes
### Operation System
- Debian 10 buster
- 2 cores | 2G memory | 20G HD

1. Preperation
   - swap off
     ```
     # check swap status
     free -h

     # off swap temporarily
     sudo swapoff -a

     # off swap permanetly 
     comment the swap in /etc/fastab
     ```
   - network monitoring
     ```
     cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
     overlay
     br_netfilter
     EOF

     cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
     net.bridge.bridge-nf-call-ip6tables = 1
     net.bridge.bridge-nf-call-iptables = 1
     net.ipv4.ip_forward = 1
     EOF
     ```
     ```
     sudo sysctl --system
     sudo reboot
     ```
   - upgrade kernel
     ```
     # check hugetlb
     grep HUGETLB /boot/config-$(uname -r)
     # match below
     > # CONFIG_CGROUP_HUGETLB is not set
     > CONFIG_ARCH_WANT_GENERAL_HUGETLB=y
     > CONFIG_HUGETLBFS=y
     > CONFIG_HUGETLB_PAGE=y
     ```
     ```
     # replace the debian repo link to meet your location
     sudo echo "deb http://mirrors.tuna.tsinghua.edu.cn/debian buster-backports main contrib non-free" > /etc/apt/source.list.d/backports.list
     sudo update
     sudo apt -t buster-backports install linux-image-amd64
     sudo apt -t buster-backports install linux-headers-amd64 #optional
     sudo reboot
     sudo apt autoremove --purge
     ```

2. Install docker
   - alternative options: [docker](https://docs.docker.com/engine/install/)
     ```
     curl -fsSL https://get.docker.com -o get-docker.sh
     sudo sh get-docker.sh
     ```
     ```
     # additional steps you may concern
     sudo usermod -aG docker $USER
     sudo systemctl enable docker
     sudo systemctl start docker
     ```
   - setup cgroup driver
     ```
     sudo nano /etc/docker/daemon.json
     {
      "exec-opts": ["native.cgroupdriver=cgroupfs"]
     }

     sudo systectl daemon-reload
     sudo systemctl restart docker
     ```

3. Install kubernetes
   - install components 
     ```
     # replace kubernetes repo link to meet your location
     sudo curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
     sudo echo "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" >/etc/apt/sources.list.d/kubernetes.list
     sudo apt update
     sudo apt install -y kubelet kubeadm kubectl

     # hold the verions
     sudo apt-mart hold kubelet kubeadm kubectl
     ```
   - pull images
     ```
     # check the version
     kubeadm config images list
     ```
     ```
     # download the images via shell script
     #!/bin/bash
     images = (
       kube-apiserver:v1.24.1
       kube-controller-manager:v1.24.1
       kube-scheduler:v1.24.1
       kube-proxy:v1.24.1
       pause:3.7
       etcd:3.5.3-0
       coredns:v1.8.6
     )
     for imageName in ${images[@]} ; do
     docker pull registry.aliyuncs.com/google_containers/$imageName
     docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
     docker rmi registry.aliyuncs.com/google_containers/$imageName
     done
     ```

4. Finish setup
   - system init
     ```
     sudo kubeadm init
     --apiserver-advertise-address=192.168.1.240 \ #optional
     --image-repository registry.aliyuncs.com/google_containers \ #optional
     --kubernetes-version v1.24.1 \ #optional
     --service-cidr=10.96.0.0/12 \ #optional
     --pod-network-cidr=10.244.0.0/16 #optional
     ```  
   - local user configuration
     ```
     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config

     # root
     export KUBECONFIG=/etc/kubernetes/admin.conf
     # non-root
     sudo echo "export KUBECONFIG=$HOME/.kube/config" >> $HOME/.bashrc
     source .bashrc

     # copy configurations to nodes
     # replace user@ip below
     scp $HOME/.kube/config user@ip $HOME/.kube/config

     # import profile on nodes
     sudo echo "export KUBECONFIG=$HOME/.kube/config" >> $HOME/.bashrc
     source .bashrc
     ```
   - install pod network plugins
     - [calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)
     ```
     #less than 50 nodes
     kubectl apply -f https://projectcalico.docs.tigera.io/manifests/calico.yaml
     sudo reboot
     ```
   - join the network
     ```
     # replace masterIP below
     # to get the token: kubeadm config print
     sudo kubeadm join masterIP:6443 --token kpd70f.cqydbjn21911u8rb \
           --discovery-token-ca-cert-hash sha256:50e4247d4a3663a75f461347bb8497d123492864cc851b37a6e6f289fe6507d0
     ```