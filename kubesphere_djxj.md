## 安装

### 组成

| 名称  | 角色 |version | int. IP | ext. IP| OS-IMAGE|KERNEL-VERION|CONTAINER-RT.|
| :-  | :-  | :-  | :- | :- |:-|:-|:-|
|djxj02 | worker              |v1.23.10|192.168.3.184|none|Ub 22.04.3 LTS   |5.15.0-83-generic      |docker://24.0.6|
master.1| control-plane,master|v1.23.10|192.168.3.183|none|Ce Linux 7 (Core)|3.10.0-1160.el7.x86_64 |docker://20.10.8|
master.2| control-plane,master|v1.23.10|192.168.3.182|none|Ub 22.04.3 LTS   |5.15.0-83-generic      |docker://24.0.5|
node.1  | worker              |v1.23.10|192.168.3.185|none|Ce Linux 7 (Core)|3.10.0-1160.el7.x86_64 |docker://24.0.6|
node.2  | worker              |v1.23.10|192.168.3.186|none|Ub 22.04.3 LTS   |5.15.0-83-generic      |docker://24.0.5|


### 一. 准备

1.关闭防火墙
```
systemctl stop firewalld && systemctl disable firewalld && iptables -F
```
关闭selinux
```
sed -i 's/enforcing/disabled/' /etc/selinux/config && setenforce 0
```
2.关闭swap分区<br>
临时关闭
``` 
swapoff -a
```
永久关闭swap<br>
```
sed -ri 's/.*swap.*/#&/' /etc/fstab
``` 
3.设置机器名，修改hosts文件

设置主机名(不设置也可以，但是要保证主机名不相同)<br>
master.1上
```
hostnamectl set-hostname master.1 
```
node.1
```
hostnamectl set-hostname node.1
```
node.2
```
hostnamectl set-hostname node.2
```
修改本地hosts文件<br>
/etc/hosts 添加如下内容
```
192.168.3.183 master.1
192.168.3.185 node.1
192.168.3.186 node.2
```
4.修改内核参数<br>
```
cat /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

```

执行下面命令，使改变生效
>sysctl --system<br>

5.加载ip_vs内核模块 (<font color=#ffff>跳过这一步</font>)<br>
如果kube-proxy 模式为ip_vs则必须加载，这里采用iptables

```
modprobe ip_vs 
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4

```

设置下次开机自动加载
```
cat > /etc/modules-load.d/ip_vs.conf << EOF 
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
EOF
```

### 二. 安装Docker

1.如果已经安装过Docker，先删除旧的Docker
```
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

2.添加Docker repository yum源

国内源, 速度更快, 推荐
```
#ustc
sudo yum-config-manager --add-repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo

#ali:(本次安装没有成功，采用ustc源安装成功)
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 官方源, 服务器在国外, 安装速度慢
# $ sudo yum-config-manager \
#     --add-repo \
#     https://download.docker.com/linux/centos/docker-ce.repo

```
3.开始安装Docker
```
#更换respo后执行makecache
sudo yum makecache fast
sudo yum install docker-ce docker-ce-cli containerd.io
```

更改dockker的Cgroup driver模式为systemd<br>
vim /etc/docker/daemon.josn
加入:<br>
```
{  
   "exec-opts": ["native.cgroupdriver=systemd"],
   "registry-mirrors": ["https://ehdoihig.mirror.aliyuncs.com"]  
}
```


完成后开启Docker
```
sudo systemctl enable docker
sudo systemctl start docker
```
ubuntu用：
```
sudo apt install docker.io
```
4.验证是否安装成功
>sudo docker run hello-world

如果出现"Hello from Docker.", 则代表运行成功

如果在每次运行docker命令是, 在前面不添加sudo, 可以执行如下命令:
>sudo usermod -aG docker $USER

如果要列出所有docker版本<br>
>yum list docker-ce.x86_64 --showduplicates |sort<br>

### 三. 安装kubesphere(include kubeadm,kubelet和kubectl)<br>

安装socat，conntrack， ispvadm

if in China, set KKZONE to cn.
```
export KKZONE=cn
```


下载kk
```
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.7 sh -
```
生成配置文件
```
./kk create config --with-kubesphere
```

-------
1.配置yum源(这里使用阿里云的源)
```
cat > /etc/yum.repos.d/kubernetes.repo << EOF
name=Kubernetes<br>
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

----
在Ubuntu 22上安装Docker，可以通过以下命令完成：
```
sudo apt-get update  
sudo apt-get install docker.io
```
启动Docker服务并启用它，可以使用以下命令：

```
sudo systemctl start docker  
sudo systemctl enable docker

```
-----
./kk create config -f kubesphere-v3.3.0.yaml --with-kubernetes v1.26.5 --with-kubesphere v3.3.0
[kubernetes]



2.安装指定版本的kubeadm,kubelet,kubectl

>yum install -y kubelet-1.18.8 kubeadm-1.18.8 kubectl-1.18.8

由于不知道默认安装的最新版，国内的阿里云镜像站同步会有延迟，导致无法拉取镜像。如果你可以拉去到最新的镜像那请随意。<br>
下面命令安装最新版本
> yum install -y kubelet kubeadm kubectl

用下面命令可以检查安装版本<br>
>systemctl version
我安装的版本是
```
[root@master ~]# kubectl version
Client Version: v1.28.1
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
The connection to the server localhost:8080 was refused - did you specify the right host or port?
[root@master ~]# 

```
3.设置开机自启
>systemctl enable kubelet

4.列出所有版本
>yum list kubelet --showduplicates

五、部署Kubernetes Master节点<br>
1.master节点初始化
```
kubeadm init --kubernetes-version 1.28.1  --apiserver-advertise-address=192.168.2.38  --service-cidr=10.96.0.0/16 --pod-network-cidr=10.244.0.0/16 --image-repository registry.aliyuncs.com/google_containers --ignore-preflight-errors=all
```

```
kubeadm init --apiserver-advertise-address=192.168.1.110 --pod-network-cidr=192.168.0.0/16 --service-cidr=172.243.0.0/16
```
参数说明

--kubernetes-version v1.18.8 指定版本

--apiserver-advertise-address 为通告给其它组件的IP，一般应为master节点的IP地址

--service-cidr 指定service网络，不能和node网络冲突

--pod-network-cidr 指定pod网络，不能和node网络、service网络冲突

--image-repository registry.aliyuncs.com/google_containers 指定镜像源，由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

如果k8s版本比较新，可能阿里云没有对应的镜像，就需要自己从其它地方获取镜像了。

--control-plane-endpoint 标志应该被设置成负载均衡器的地址或 DNS 和端口(可选)

注意点：

版本必须和上边安装的kubelet,kubead,kubectl保持一致

2.等待拉取镜像

也可用自己提前给各个节点拉取镜像 ，查看所需镜像命令: 
>kubeadm --kubernetes-version 1.18.8 config images list

等待镜像拉取成功后，会继续初始化集群，等到初始化完成后，会看到类似如下信息，保留最后两行的输出后边会用到

3.配置kubectl  
就是执行初始化成功后输出的那三条命令

>mkdir -p $HOME/.kube<br>
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config<br>
chown $(id -u):$(id -g) $HOME/.kube/config<br>

4.查看节点信息
>kubectl get nodes

[root@k8s-node1 k8s]# rm -rf /etc/containerd/config.toml
[root@k8s-node1 k8s]# systemctl restart containerd


