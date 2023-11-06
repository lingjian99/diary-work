## 基本信息

网络由安装在同一个windows操作系统上的VMWare虚拟机，配置较低，实践为主。
| 主机 | 角色 | IP | 配置 | OS 
| -: | -: | -: |-: | -: |
| ks.master.1 | master | 192.168.3.177 | 2 processors，4GB |CentOS 7.9.2009 x86_64，VM
| ks.master.2 | master.  | 192.168.3.176 | 1 processor， 4GB | CentOS 7.9.2009 x86_64, VM
|ks.node.1 | worker | 192.168.3.179 | 1 processor， 4GB | CentOS 8.5.2111 x86_64, VM


#### <font color=#ffff>Kubernetes安装</font>
##### 1. 准备

网络规划，配置机器，修改hostname（如果必要），并在/etc/hosts文件添加主机信息（我不知道这一步是不是必要）

配置服务器时区为Asia/Shanghai

```
timedatectl set-timezone Asia/Shanghai
```
验证服务器时区，正确配置如下。
```
[jian@master ~]# timedatectl<br>
    Local time: Sat 2023-09-02 14:43:50 CST
Universal time: Sat 2023-09-02 06:43:50 UTC
    RTC time: Sat 2023-09-02 06:43:50
    Time zone: Asia/Shanghai (CST, +0800)
    NTP enabled: yes
NTP synchronized: yes
RTC in local TZ: no
    DST active: n/a
```


关闭防火墙
```
systemctl stop firewalld
systemctl disable firewalld
```
ubuntu 关闭防火墙命令
```
sudo systemctl stop ufw.service
sudo systemctl disable ufw.service

sudo ufw status
```


禁用SELinux

查看SELinux

> sestatus<br>

暂时关闭, 永久关闭
```
setenforce 0 && sed -i 's/enforcing/disabled/' /etc/selinux/config  
```
或者：编辑/etc/selinux/config文件，将“SELINUX”项的值改为“disabled”。

关闭swap
```
swapoff -a && sed -ri 's/.*swap.*/#&/' /etc/fstab
```

将桥接的IPv4流量传递到iptables的链
```
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1
EOF

```
并通过 “sysctl --system” 命令使其生效

sudo sysctl - p sudo sysctl -p /etc/sysctl.d/k8s.conf

安装socat and conntrack which rae required when install kubespere
```
yum install -y socat conntrack
```
#### 2. 安装Docker

版本：Docker version 24.0.5

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum makecache fast
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
use ali mirror in China instead:
```
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

```
#### 3. 安装Kubernetes

从 GitHub 发布页面下载 KubeKey 或直接使用以下命令。

curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.7 sh -

if in china use the following command instead:
```
export KKZONE=cn
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.7 sh -
chmod +x kk
```

创建示例配置文件
命令如下：
```
./kk create config [--with-kubernetes version] [--with-kubesphere version] [(-f | --file) path]
```


#### 4. 安装flannet

---
安装完成以后，执行kubectl get nodes -o wide 

```
[jian@master ~]$ kubectl get nodes -o wide <br>
NAME     STATUS   ROLES                  AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE<br>
KERNEL-VERSION                CONTAINER-RUNTIME<br>
master   Ready    control-plane,master   22h     v1.23.0   192.168.3.172   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64        docker://1.13.1<br>
node1    Ready    <none>                 5h1m    v1.23.0   192.168.3.169   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64        docker://24.0.5<br>
node2    Ready    <none>                 6m39s   v1.23.0   192.168.3.170   <none>        CentOS Linux 8          4.18.0-348.7.1.el8_5.x86_64   docker://24.0.5<br>
```




