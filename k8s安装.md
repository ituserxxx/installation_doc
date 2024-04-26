

调试用    
```
#查看组件状态
kubectl get pods -n kube-system

# 查看服务日志
journalctl -f -u kubelet.service

# 查看节点
kubectl get nodes

#打印加入命令
kubeadm token create --print-join-command
```






腾讯云 3台 2h4G ubuntu 2204 版服务器，按量计费，用完即毁

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
hostnamectl set-hostname k8s-slave-one  && bash
hostnamectl set-hostname k8s-slave-two  && bash
```
安装环境（所有节点上面执行）

使用 swapoff 命令关闭当前正在使用的 swap 分区 （主、从节点操作）
```
# 临时关闭
swapoff -a
# 永久关闭
sed -i '/swap/d' /etc/fstab
```
注释掉 /etc/fstab 文件中关于 swap 分区的条目 （主、从节点操作）
```
找到类似 UUID=<swap_partition_uuid> swap swap defaults 0 0 的行，并在行首添加 # 注释掉它。
```
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/d33e2363-b28a-4c47-999c-3dc0ab41ade9)


关闭防火墙：ufw disable

针对特定的内网地址范围开放所有端口
```
 ufw allow from 10.206.0.0/16   (内网 slave 网段)
``` 

配置Ubuntu软件源 
```
vi /etc/apt/sources.list
```
最后增加内容
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

安装k8s环境（所有节点上面执行）
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

找到 sandbox_image 关键字,用下面内容替换它，这里的 regitry 地址就是 上面  kubeadm config images pull 出现的地址
```
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9" 
```

编辑 /etc/crictl.yaml 文件，输入以下内容
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

输出如下
```
pause_threshold = 0.02
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
```

初始化 kubeadm （master 节点执行）
```
# kubeadm init 是用于初始化创建一个新的 Kubernetes 集群的命令
# --pod-network-cidr=33.33.0.0/16 指定了 Pod 网络的 CIDR 地址段为 33.33.0.0/16。这个地址段用来分配给 Kubernetes 中的 Pod 网络，以便于不同 Pod 之间的通信和网络隔离。
# --apiserver-advertise-address 参数一定要用master节点的内网ip

kubeadm init --apiserver-advertise-address 10.206.0.15 --pod-network-cidr=33.33.0.0/16 --image-repository registry.aliyuncs.com/google_containers
```
如果执行完 kubeadm init 后 报错1
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/2a9dcc13-a0a0-4ecb-b820-2d823c622178)
解决1
查看 kubelet 状态
```
systemctl status kubelet
```
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/dfa0efdc-8c56-44a8-bcc1-f397f25f1b44)
发现状态正常，但是各组件报错


初始化 kubeadme 之后有如下输出
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/7810c511-2af3-4038-b3cc-a65b9cce0aa1)


**先保存图中的 3 那2行命令，在子节点加入集群的时候需要用**

根据提示执行 1 或者 2 命令
- 命令 1 如果当前用户是非root用户执行
- 命令 2 当前用户是root用户执行（我这只需要执行2）

查看安装是否成功
```
kubectl get pods -n kube-system
```

到这里面为止，主节点已经安装完成，接下来是从节点


从节点加入集群（在所有从节点操作）


检测是否可加入
```
nc -zv 10.206.0.6 6443   （主节点内网ip）
```
从节点执行加入集群命令使用的是上面初始化成功后提示的命令3（**执行这个命令一定要一行执行，不要像提示的那样分行了**）
```
kubeadm join 10.206.0.15:6443 --token blhkgv.5q05s2mbhon7mv7m  --discovery-token-ca-cert-hash sha256:b6dd53da9115cc56fe668cae809dc30eb74d2d221e155c0f9a1094704ddef111
```

如果有以下报错（都是从节点上面）

![image](https://github.com/ituserxxx/installation_doc/assets/66945660/423eb242-0981-451b-9178-8ee6db58101e)

![image](https://github.com/ituserxxx/installation_doc/assets/66945660/d1f96485-9b78-458b-adeb-4cc4474cad00)

![image](https://github.com/ituserxxx/installation_doc/assets/66945660/5d1f1335-a951-48b0-a003-6a69b2306e30)

用下面方式解决


重启  kubelet
```
systemctl restart kubelet
```
重置 kubeadm
```
kubeadm reset
```
再重新加入集群，执行上面的 kubeadm join 命令


加入成功后有如下提示
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/7b3966c2-8212-4280-bc3a-e4b2aca93a22)

加入后在主节点查看从节点
```
kubectl get nodes
```
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/f52ffa3b-8049-4692-bd5c-4b4910eefd21)

在从节点查看
```
kubectl get nodes
```
显示如下

![image](https://github.com/ituserxxx/installation_doc/assets/66945660/1ac8238a-e9f1-4db8-8b9a-f10cd36246fd)
解决

```
mkdir ~/.kube
cp /etc/kubernetes/kubelet.conf  ~/.kube/config
kubectl get pod
```

配置网络(flannel)

查看 node 。此时 node 状态应该是 NotReady，因为还没配置网络插件（ master 节点）

下载默认配置文件
```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```

修改 kube-flannel.yml，在第 200 行左右增加指定网卡的配置：
```
- --iface=XXX

#如果机器存在多网卡的话，最好指定内网网卡的名称。CentOS 下一般是 eth0 或者 ens33 。不指定的话默认会使用第一块网卡。
#也可以不指定，使用默认配置
```

然后改这段
```
net-conf.json: |
    {
      "Network": "33.33.0.0/16", //这里的ip需要修改
      "Backend": {
        "Type": "vxlan"
      }
    }

```
将 Network 改为 ​ kubeadm init 命令时指定的那个--pod-network-cidr=33.33.0.0/16 

创建、启动 flannel
```
kubectl create -f kube-flannel.yml 
kubectl apply -f kube-flannel.yml 
​
#再次查看 pod ，coredns 的状态是否变为变为 Running

kubectl get pods -n kube-system

# 如果有报错：卸载重新安装
# 使用 kubectl delete 命令删除之前创建的资源对象。在这种情况下，运行以下命令：

kubectl delete -f kube-flannel.yml

等待资源对象删除成功后，可以再次运行 kubectl apply 命令来重新创建资源对象。使用以下命令：

kubectl apply -f kube-flannel.yml
```
![image](https://github.com/ituserxxx/installation_doc/assets/66945660/449fc65e-1b31-4a8f-8f26-f5656c68d430)

![image](https://github.com/ituserxxx/installation_doc/assets/66945660/b0eeb34d-8145-4ecc-9f85-01e0182daf6a)


