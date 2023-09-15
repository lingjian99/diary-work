
## 以docker方式安装etcd
reference：https://blog.51cto.com/u_15535797/6076228

安装完成后，执行:
```
> docker network create app-tier --driver bridge
> docker run -d --name etcd-server --network app-tier --publish 2379:2379 --publish 2380:2380 --env 
ALLOW_NONE_AUTHENTICATION=yes  --env ETCD_ADVERTISE_CLIENT_URLS=http://127.0.0.1:2379 bitnami/etcd:latest
```

