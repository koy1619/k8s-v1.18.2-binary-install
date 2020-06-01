拓扑

[![_20200601141117.jpg](https://wx2.sbimg.cn/2020/06/01/_20200601141117.jpg)](https://sbimg.cn/image/kGn1l)

---

服务器规划

| 主机名    | IP   |
| :-----: | :----:  |
| k8s-master   |  10.127.0.16     |
| k8s-node-1     |  10.127.0.17   |
| k8s-node-2     |  10.127.0.18  |
| slb    |  10.127.0.10   |


网段划分

| 名称    | IP网段   | 备注   |
| :-----: | :----:  | :----:  |
| service-cluster-ip   |  10.10.0.0/16     | 可用地址 65534     |
| pods-ip                |  10.20.0.0/16   | 可用地址 65534     |
| 集群dns                |  10.10.0.2  | 用于集群service域名解析     |
| k8s svc        | 10.10.0.1  | 集群 kubernetes svc 解析ip     |


服务版本
>* Docker CE version 19.03.6
>* Kubernetes Version 1.18.2
>* Etcd Version v3.4.7
>* Calico Version v3.14.0
>* Coredns Version 1.6.5
>* Metrics-Server Version v0.3.6
>* nginx-ingress-controller Version 0.24.1

---

K8S集群说明
>* 所有主机使用 CentOS 7.6.1810 版本，并且内核都升到5.x版本。
>* K8S 集群使用 Iptables 模式（kube-proxy 注释中预留 Ipvs 模式配置）
>* Calico 使用 IPIP 模式
>* 集群域使用默认 svc.cluster.local
>* 10.10.0.1 为集群 kubernetes svc 解析ip
>* haproxy设置TCP监听nginx-ingress的svc端口


部署顺序

>* 一、初始化
>* 关闭 firewalld
>* 关闭 swap
>* 关闭 Selinux
>* 修改内核参数
>* 预先设置 PATH
>* 设置hostname
>* 判断内核并升级
>* 安装docker

>* 二、证书生成
>* 准备cfssl证书生成工具
>* 自签TLS证书,metrics-server 证书

>* 三、部署Etcd集群
>* 从Github下载二进制文件
>* 拷贝master上的ssl证书到node
>* 设置kubeconfig
>* 依次启动etcd
>* 健康检查

>* 四、部署Master Node
>* 从Github下载二进制文件; 解压二进制包
>* 部署kube-apiserver
>* 部署kube-controller-manager
>* 部署kube-scheduler
>* 配置kubelet证书自动续期和创建Node授权用户
>* 批准kubelet证书申请并加入集群

>* 五、部署Worker Node
>* 创建工作目录并拷贝二进制文件
>* 拷贝master上的ssl证书到node
>* 部署kubelet
>* 部署kube-proxy

>* 六、组件安装
>* 部署CNI网络(calico)
>* 部署metrics-server
>* 部署CoreDNS
>* 部署nginx-ingress(负载均衡)

>* 七、高可用架构(扩容多Master架构)可选
>* 部署多个master节点
>* 使用slb负载apiserver
>* 修改所有Node连接slb-apiserver





**********************************************************************

**ps 部分文件内容根据实际情况，须修改对应配置项，下文都有说明**

**master如需做高可用集群，则安装多个master节点，使用slb负载apiserver**

**********************************************************************

# 初始化

```bash
master # sh init.sh  k8s-master

node # sh init.sh  k8s-node-1

node # sh init.sh  k8s-node-2
```


# TLS证书

```
# 须根据实际情况修改证书里面的IP地址
master # sh mktls.sh
```

# etcd_install_config

```
master # sh etcd_install.sh

master # kubernetes_install.sh    # kubernetes二进制文件安装

master # sh kubeconfig.sh   # 根据实际情况修改KUBE_APISERVER地址

master # scp -r /k8s/* root@all_node:/k8s/

master # sh etcd_conf.sh etcd01 10.127.0.16 etcd01=https://10.127.0.16:2380,etcd02=https://10.127.0.17:2380,etcd03=https://10.127.0.18:2380

node # sh etcd_conf.sh etcd02 10.127.0.17 etcd01=https://10.127.0.16:2380,etcd02=https://10.127.0.17:2380,etcd03=https://10.127.0.18:2380

node # sh etcd_conf.sh etcd03 10.127.0.18 etcd01=https://10.127.0.16:2380,etcd02=https://10.127.0.17:2380,etcd03=https://10.127.0.18:2380

# etcd健康检查
$ ETCDCTL_API=3 /k8s/etcd/bin/etcdctl  --write-out=table \
--cacert=/k8s/kubernetes/ssl/ca.pem --cert=/k8s/kubernetes/ssl/server.pem --key=/k8s/kubernetes/ssl/server-key.pem \
--endpoints=https://10.127.0.16:2379,https://10.127.0.17:2379,https://10.127.0.18:2379 endpoint health
```




# kubernetes_config

```
master # sh apiserver.sh 10.127.0.16 https://10.127.0.16:2379,https://10.127.0.17:2379,https://10.127.0.18:2379 
master # sh controller-manager.sh 127.0.0.1
master # sh scheduler.sh 127.0.0.1

# 查看master三个服务是否正常运行
master # ps -ef | grep kube
master # netstat -ntpl | grep kube-

kubectl  get cs


#配置kubelet证书自动续期和创建Node授权用户
kubectl create clusterrolebinding  kubelet-bootstrap --clusterrole=system:node-bootstrapper  --user=kubelet-bootstrap

#创建自动批准相关 CSR 请求的 ClusterRole
kubectl apply -f /app/yaml/system/tls-instructs-csr.yaml

#自动批准 kubelet-bootstrap 用户 TLS bootstrapping 首次申请证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-approve-csr --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient --user=kubelet-bootstrap

#自动批准 system:nodes 组用户更新 kubelet 自身与 apiserver 通讯证书的 CSR 请求
kubectl create clusterrolebinding node-client-auto-renew-crt --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient --group=system:nodes

#自动批准 system:nodes 组用户更新 kubelet 10250 api 端口证书的 CSR 请求
kubectl create clusterrolebinding node-server-auto-renew-crt --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeserver --group=system:nodes


#解决无法查询pods日志问题
kubectl apply -f apiserver-to-kubelet-rbac.yml
```

# node_install

```
master # scp -r /k8s/* root@all_node:/k8s/

node # sh kubelet.sh 10.10.0.2 k8s-node-1 cluster.local
node # sh proxy.sh k8s-node-1
node # netstat -ntpl | egrep "kubelet|kube-proxy"
```



# Component_install

```
[root@k8s-master install_k8s]# kubectl  get node
NAME         STATUS     ROLES    AGE   VERSION
k8s-node-1   NotReady   <none>   20s   v1.18.2

# calico网络组件安装
# 需修改 etcd-key etcd-cert etcd-ca etcd_endpoints KUBERNETES_SERVICE_HOST 配置项
kubectl  create -f calico-etcd.yaml

kubectl  get pods -n kube-system  | grep calico


[root@k8s-master install_k8s]# kubectl  get node
NAME         STATUS   ROLES    AGE     VERSION
k8s-node-1   Ready    <none>   146m    v1.18.2

# metrics-server监控安装
cd /app/yaml/metrics-server-v0.3.6/
kubectl  create -f .
kubectl get pods -n kube-system |grep metrics-server
kubectl -n kube-system logs -l k8s-app=metrics-server  -f
kubectl  top node
kubectl  top pod

# coredns安装 可自定义上游域名 修改 hosts 配置项
kubectl create -f coredns.yaml

# master 打污点
kubectl taint node k8s-master key1=value1:NoSchedule
```

# nginx-ingress

```
kubectl  create -f nginx-ingress.yaml

# ssl secret
kubectl create secret tls ebuy-secret --key server.key --cert server.crt



# demo 测试
kubectl  create -f demo.yaml

# 解析域名到任意nodeip访问


# 负载均衡
[root@k8s-master ~]# kubectl  get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.10.32.187   <none>        80:32080/TCP,443:32443/TCP   6m31s


# haproxy配置

listen nginx_gress_http
     mode tcp
     balance roundrobin
     bind 10.127.0.10:80
     timeout client 30s
     timeout server 30s
     timeout connect 30s
     server k8s_node_1 k8s-node-1-ip:32080 weight 1 check inter 2000 rise 5 fall 10
     server k8s_node_2 k8s-node-2-ip:32080 weight 1 check inter 2000 rise 5 fall 10

listen nginx_gress_https
     mode tcp
     balance roundrobin
     bind 10.127.0.10:443
     timeout client 30s
     timeout server 30s
     timeout connect 30s
     server k8s_node_1 k8s-node-1-ip:32443 weight 1 check inter 2000 rise 5 fall 10
     server k8s_node_2 k8s-node-2-ip:32443 weight 1 check inter 2000 rise 5 fall 10


# 解析域名到10.127.0.10访问
         
kubectl  get nodes,cs,svc,pods  -o wide  --all-namespaces
```

#  参考
https://mp.weixin.qq.com/s/_TxeS1Fiy9XEOEEnF31deA
