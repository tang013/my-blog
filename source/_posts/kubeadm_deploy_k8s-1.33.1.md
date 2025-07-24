在 Ubuntu 系统上搭建一个高可用 Kubernetes 集群，包含：

-   1 个主 master 节点
    
-   2 个 node 节点
    

## **环境准备**

### **硬件要求**

-   所有 master 节点: 至少 2GB RAM (推荐 4GB+), 2 CPU 核心
    
-   node 节点: 根据工作负载需求
    
-   所有节点至少 20GB 磁盘空间
    
-   所有节点间网络互通
    

### **系统要求**

-   Ubuntu 24.04.2 LT (所有节点使用相同版本)
    
-   所有节点设置静态 IP
    
-   所有节点时间同步 (建议安装 chrony)
    
-   k8s 1.33.1
    

## **步骤 1：在所有节点上执行的基础设置**

### **1.1 更新系统**

    sudo apt update && sudo apt upgrade -y

### **1.2 安装必要工具**

    sudo apt install -y apt-transport-https ca-certificates curl software-properties-common chrony

### **1.3 禁用交换空间**

    sudo swapoff -a
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

### **1.4 设置主机名 (在每个节点上分别设置)**

-   Master 节点：
    

    sudo hostnamectl set-hostname k8s-master01

-   node01：
    

    sudo hostnamectl set-hostname k8s-node01

-   node02：
    

    sudo hostnamectl set-hostname k8s-node02

### **1.5 编辑 /etc/hosts 文件 (所有节点相同)**

    sudo tee -a /etc/hosts <<EOF
    <master1节点IP> k8s-master1
    <node1节点IP> k8s-node1
    <node2节点IP> k8s-node2
    EOF

### **1.6 加载内核模块**

    sudo tee /etc/modules-load.d/k8s.conf <<EOF
    overlay
    br_netfilter
    EOF
    
    sudo modprobe overlay
    sudo modprobe br_netfilter
    
    # 设置必要的内核参数
    sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF
    
    sudo sysctl --system

## **步骤 2：在所有节点上安装容器运行时 (containerd)**

### **2.1 安装 containerd**

    sudo apt install -y containerd

### **2.2 配置 containerd**

    sudo mkdir -p /etc/containerd
    containerd config default | sudo tee /etc/containerd/config.toml
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
    
    
    # 修改/etc/crictl.yaml 文件
    cat > /etc/crictl.yaml <<EOF
    runtime-endpoint: unix:///run/containerd/containerd.sock
    image-endpoint: unix:///run/containerd/containerd.sock
    timeout: 10
    debug: false
    EOF

### **2.3 重启并启用 containerd**

    sudo systemctl restart containerd
    sudo systemctl enable containerd

## **步骤 3：在所有节点上安装 Kubernetes 组件**

### **3.1 添加 Kubernetes 仓库**

    sudo apt-get install -y apt-transport-https ca-certificates curl gpg
    
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

### **3.2 安装 kubelet, kubeadm 和 kubectl**

    apt update
    
    #查看软件包详情
    root@k8s-master01:~# apt-cache madison kubelet
       kubelet | 1.33.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
       kubelet | 1.33.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
    root@k8s-master01:~# apt-cache madison kubeadm
       kubeadm | 1.33.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
       kubeadm | 1.33.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
    root@k8s-master01:~# apt-cache madison kubectl
       kubectl | 1.33.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
       kubectl | 1.33.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.33/deb  Packages
    
    
    #安装
    apt install -y kubelet=1.33.1-1.1 kubeadm=1.33.1-1.1 kubectl=1.33.1-1.1
    #锁定包的版本
    apt-mark hold kubelet kubeadm kubectl

## **步骤 4：在Master 节点上初始化集群**

### **4.1 初始化 Kubernetes 集群**

    sudo kubeadm init --control-plane-endpoint "192.168.50.113:6443" \
    --pod-network-cidr=10.244.0.0/16 \
    --upload-certs \
    --apiserver-advertise-address=192.168.50.113

### **4.2 设置 kubectl 配置文件**

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

### **4.3 安装网络插件 (Calico)**

    curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml
    kubectl apply -f calico.yaml

### **5.4 获取加入集群的命令**

    # 获取其他 control plane 节点加入命令
    kubeadm token create --print-join-command
    
    # 获取 node 节点加入命令
    kubeadm token create --print-join-command | sed 's/--control-plane //'

## **步骤 6：在 node 节点上加入集群**

使用主 master 节点输出的 node 加入命令：

    sudo kubeadm join <负载均衡器IP或DNS>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

## **步骤 7：验证集群状态**

在 master 节点上执行：

### **7.1 检查节点状态**

    kubectl get nodes

等待所有节点状态变为 `Ready` (可能需要几分钟)

![](/upload/image.png)

### **7.2 检查所有 pod 状态**

    kubectl get pods --all-namespaces

### **7.3 检查高可用状态**

    kubectl get endpoints kube-scheduler -n kube-system -o yaml
    kubectl get endpoints kube-controller-manager -n kube-system -o yaml

## **可选步骤：配置 kubectl 自动补全**

    echo 'source <(kubectl completion bash)' >>~/.bashrc
    echo 'alias k=kubectl' >>~/.bashrc
    echo 'complete -o default -F __start_kubectl k' >>~/.bashrc
    source ~/.bashrc

## **故障排除**

如果节点无法加入集群或状态不正常，可以尝试以下命令：

1.  检查 kubelet 状态：
    

    sudo systemctl status kubelet

2.  查看日志：
    

    sudo journalctl -u kubelet -f

3.  重置节点 (如果需要重新加入)：
    

    sudo kubeadm reset