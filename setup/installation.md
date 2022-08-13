# Install kubernetes
### Operation System
- Debian 10 buster
- 2 cores | 2G memory | 20G HD

1. Preperation on all nodes
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

2. Install docker on all nodes
   - alternative options: [docker](https://docs.docker.com/engine/install/)
     ```
     curl -fsSL https://get.docker.com -o get-docker.sh
     sudo sh get-docker.sh --mirror Aliyun
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

3. Install kubernetes on all nodes
   - install components 
     ```
     # replace kubernetes repo link to meet your location
     su -
     curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
     echo "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" >/etc/apt/sources.list.d/kubernetes.list
     apt update
     apt install -y kubelet kubeadm kubectl

     # hold the verions
     apt-mark hold kubelet kubeadm kubectl
     exit
     ```
   - pull images on master nodes
     ```
     # check the version
     kubeadm config images list
     ```
     ```
     # download the images via shell script
     #!/bin/bash
     images=(
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
   - system init on master node
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
     scp $HOME/.kube/config user@ip:$HOME/.kube/config

     # import profile on nodes
     sudo echo "export KUBECONFIG=$HOME/.kube/config" >> $HOME/.bashrc
     source .bashrc
     ```
   - install pod network plugins on all nodes
     - [calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises)
     ```
     #less than 50 nodes
     kubectl apply -f https://projectcalico.docs.tigera.io/manifests/calico.yaml
     sudo reboot
     ```
   - join to cluster
     ```
     #to get the token: 
     kubeadm token list

     #to create a new token: 
     kubeadm token create
     
     #to request --discovery-token-ca-cert-hash
     openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \ openssl dgst -sha256 -hex | sed 's/^.* //'

     #replace $masterIP/$TOKEN/$HASH below
     su -
     kubeadm join $masterIP:6443 --token $TOKEN \
           --discovery-token-ca-cert-hash $HASH
     ```