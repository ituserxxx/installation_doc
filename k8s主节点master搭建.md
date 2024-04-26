设置 root 密码
```
sudo passwd root
```

进入 root 用户，都以 root 用户操作
```
su root
```

临时设置 hostname
```
hostnamectl set-hostname k8s-master  && bash
```

临时关闭 swapoff 命令关闭当前正在使用的 swap 分区 （主、从节点操作）
```
swapoff -a
```

永久关闭
```
vim /etc/fstab
```
找到类似 UUID=<swap_partition_uuid> swap swap defaults 0 0 的行，并在行首添加 # 注释掉它。
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/eeb448bb-2a38-47f4-890e-f562e81421f1)


配置Ubuntu软件源
```
vi /etc/apt/sources.list
```

在最后增加内容
```
deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
```

安装k8s环境
```
apt-get update -y

apt-get install -y ca-certificates curl gnupg apt-transport-https

mkdir -p /etc/apt/keyrings

curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

apt-get update -y

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list

# 从这里开始，下面的步骤单独执行，不要全部复制一起执行

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /etc/apt/trusted.gpg.d/kubernetes-archive-keyring.gpg add -

apt-get update -y

# 这步会有点慢，稍等一下
apt-get install -y docker.io kubelet kubeadm kubectl

kubeadm config images pull  --image-repository registry.aliyuncs.com/google_containers
```

生成配置文件
```
mkdir -p /etc/containerd
containerd config default >  /etc/containerd/config.toml
```

编辑 /etc/containerd/config.toml 文件
```
vim /etc/containerd/config.toml
```

找到 sandbox_image 关键字,用下面内容替换它，这里的 regitry 地址就是 上面 kubeadm config images pull 出现的地址
```
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
```
编辑 /etc/crictl.yaml 文件，
```
vim  /etc/crictl.yaml
```

输入以下内容
```
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 0
debug: false
pull-image-on-create: false
```
重启 containerd
```
systemctl restart containerd
```
验证 pause 镜像是否生效
```
containerd config dump | grep pause
```
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/abe504e3-616e-492b-b195-c93a3d4cc8ef)


初始化 kubeadm
```
kubeadm init --apiserver-advertise-address 10.206.0.8 --pod-network-cidr=33.33.0.0/16 --image-repository registry.aliyuncs.com/google_containers

# kubeadm init 是用于初始化创建一个新的 Kubernetes 集群的命令
# --pod-network-cidr=33.33.0.0/16 指定了 Pod 网络的 CIDR 地址段为 33.33.0.0/16。这个地址段用来分配给 Kubernetes 中的 Pod 网络，以便于不同 Pod 之间的通信和网络隔离。
# --apiserver-advertise-address 参数一定要用master节点的内网ip

```
完成后
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/d3bb262d-40e2-4919-94b3-19693e66d33b)

根据提示执行 1 或者 2 标记

- 标记 1 如果当前用户是非root用户执行
- 标记 2 当前用户是root用户执行（我这只需要执行2）


配置网络(flannel)

查看node 状态应该是 NotReady，因为还没配置网络插件，下载默认配置文件
```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

（可选）修改 kube-flannel.yml，在第 200 行左右增加指定网卡的配置：
```
- --iface=XXX
```

如果机器存在多网卡的话，最好指定内网网卡的名称。CentOS 下一般是 eth0 或者 ens33 

不指定的话默认会使用第一块网卡，也可以不指定，使用默认配置，然后改这段
```
net-conf.json: |
    {
      "Network": "33.33.0.0/16", //这里的ip需要修改：将 Network 改为 ​ kubeadm init 命令时指定的那个--pod-network-cidr=33.33.0.0/16
      "Backend": {
        "Type": "vxlan"
      }
  }
```

创建、启动 flannel
```
kubectl create -f kube-flannel.yml 
kubectl apply -f kube-flannel.yml 
​```

再次查看 pod ,coredns 的状态是否变为变为 Running
```
kubectl get pods -n kube-system
```

查看组件状态
```
kubectl get pods -n kube-system
```
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/bb73c3a5-d3a5-4701-aa8a-96f19ff2c314)


查看节点状态
```
kubectl get nodes
```

![image](https://github.com/ituserxxx/installation_doc/assets/66945660/112a6977-b3bb-4203-800d-307593141cde)






