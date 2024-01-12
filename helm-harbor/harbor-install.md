# 通过helm在kubernetes上搭建Harbor
references：

https://www.jianshu.com/p/9df9ae97db39

http://www.mydlq.club/article/66/#3%E6%8C%82%E8%BD%BD-nfs-%E4%B8%8E%E5%88%9B%E5%BB%BA%E7%9B%AE%E5%BD%95

https://blog.csdn.net/weixin_43866248/article/details/128471302

Kubernetes 版本：v1.23.0 <br>
Helm 版本：v3.13.2 <br>
Helm chart 版本：<br>


## 前期准备
### 1. 安装 nfs
references:

https://zhuanlan.zhihu.com/p/606174368?utm_id=0


一、服务端

1.1 安装NFS服务：

#执行以下命令安装NFS服务器，<br>
#apt会自动安装nfs-common、rpcbind等13个软件包
​```
sudo apt install -y nfs-kernel-server
```
创建
1.2 创建共享目录

```
sudo mkdir -p /hdd/nfs/harbor/registry
sudo mkdir -p /hdd/nfs/harbor/chartmuseum
sudo mkdir -p /hdd/nfs/harbor/jobservice
sudo mkdir -p /hdd/nfs/harbor/database
sudo mkdir -p /hdd/nfs/harbor/redis
sudo mkdir -p /hdd/nfs/harbor/trivy
```
修改权限：
```
sudo chmod -R 777 /hdd/nfs/harbor
```
修改file owner
```
sudo chown -R 1000:1000 /hdd/nfs/
```

1.3 编写配置文件：
#编辑/etc/exports 文件：

sudo vi /etc/exports
#/etc/exports文件的内容如下：

eg：
#设置共享目录
```
/hdd/nfs *(rw,sync,no_subtree_check,no_root_squash)
​```
1.4 重启nfs服务：
```
sudo service nfs-kernel-server restart
```

1.5 常用命令工具：

#在安装NFS服务器时，已包含常用的命令行工具，无需额外安装。 
#显示已经mount到本机nfs目录的客户端机器。
```
sudo showmount -e localhost
```
#将配置文件中的目录全部重新export一次！无需重启服务。
```
sudo exportfs -rv
```
#查看NFS的运行状态
```
sudo nfsstat
```
#查看rpc执行信息，可以用于检测rpc运行情况
```
sudo rpcinfo
```
#查看网络端口，NFS默认是使用111端口。
```
sudo netstat -tu -4
```

二、客户端：

2.1 安装客户端工具：
#在需要连接到NFS服务器的客户端机器上，
#需要执行以下命令，安装nfs-common软件包。
#apt会自动安装nfs-common、rpcbind等12个软件包
```
sudo apt install -y nfs-common
```
2.2 查看NFS服务器上的共享目录
#显示指定的（192.168.3.156）NFS服务器上export出来的目录

sudo showmount -e 192.168.3.156


2.3 创建本地挂载目录
```
sudo mkdir -p /hdd/nfs/harbor/registry
sudo mkdir -p /hdd/nfs/harbor/chartmuseum
sudo mkdir -p /hdd/nfs/harbor/jobservice
sudo mkdir -p /hdd/nfs/harbor/database
sudo mkdir -p /hdd/nfs/harbor/redis
sudo mkdir -p /hdd/nfs/harbor/trivy
```

2.4 挂载共享目录
#将NFS服务器192.168.3.167上的目录，挂载到本地的/mnt/目录下
```
sudo mount -t nfs 192.168.3.156:/hdd/nfs /hdd/nfs
​```
③修改文件目录权限
文件权限很重要，在这踩了很大的坑，Redis和database一直报权限不足
-R 代表harbor下的所有文件夹

sudo chmod -R 777 /hdd/nfs/harbor
如果以上权限还不够的话，将文件属主改为你当前用户

sudo chown -R 1000:1000 /hdd/nfs/


### 2. 安装 Helm
下载 需要的版本
```
https://github.com/helm/helm/releases
```
解压：
```
tar -zxvf helm-v3.13.2-linux-amd64.tar.gz
```
在解压目录中找到helm程序，移动到需要的目录中(mv linux-amd64/helm /usr/local/bin/helm)


### 3. 创建 namespace

```
kubectl create namespace harbor
```
### 4. 生成证书

```
openssl req -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 3650 -out ca.crt  -subj  "/C=CN/ST=Zhejiang/L=Hangzhou/O=example/OU=example/CN=192.168.3.156"
openssl req -newkey rsa:4096 -nodes -sha256 -keyout tls.key -out tls.csr  -subj  "/C=CN/ST=Zhejiang/L=Hangzhou/O=example/OU=example/CN=192.168.3.156"

vim extfile.cnf
#填入以下内容
subjectAltName = IP:192.168.3.156
## 生成证书
openssl x509 -req -days 3650 -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial  -extfile extfile.cnf -out tls.crt
kubectl create secret generic hub-mydlq-tls --from-file=tls.crt --from-file=tls.key --from-file=ca.crt -n harbor
```
生成secret：
```
kubectl create secret generic harbor-tls --from-file=tls.crt --from-file=tls.key --from-file=ca.crt -n harbor
```
备忘-生成证书时根据提示输入：
subject=C = CN, ST = Zhengjiang, L = Hangzhou, O = Dajiaxiaojia, emailAddress = lingjian99@yeah.net

### 设置harbor清单

添加harbor helm源

```
helm repo add harbor https://helm.goharbor.io
```
添加helm国内库
```
helm repo add azure http://mirror.azure.cn/kubernetes/charts
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts 

helm repo add stable https://burdenbear.github.io/kube-charts-mirror/

helm repo update
```
-------------
Kubernetes Helm 配置国内镜像源
1、删除默认的源

1
helm repo remove stable
2、增加新的国内镜像源

1
2
3
helm repo add stable https://burdenbear.github.io/kube-charts-mirror/
或者
helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
3、查看helm源添加情况

1
helm repo list
4、搜索测试

1
helm search mysql
5、其他

1
2
微软仓库（推荐使用）：http://mirror.azure.cn/kubernetes/charts
阿里仓库：https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

------------------
安装 
helm install harbor harbor/harbor -f values.yaml -n harbor