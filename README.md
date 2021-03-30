## kubenetest-V1.20环境安装



### 1、网络设置



```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```



### 2、安装kubeadm



```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
systemctl start kubelet
```



### 3、docker环境安装

```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
                  
sudo yum install -y yum-utils

sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce-19.03.0

## 此处要用systemd方式运行
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d
systemctl enable docker.service
systemctl daemon-reload
systemctl restart docker

## 关闭防火墙，如果不关闭，需要开放对应的端口，具体见https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
systemctl disable firewalld.service
reboot

 docker rmi $(docker images -q) -f

```



### 4、修改host以及关闭交换分区



```shell
hostnamectl set-hostname k8s-master
## 在host添加映射

IP   k8s-master
IP   api.k8s.songcw.com  ## 下面初始化需要用到

##关掉swapoff
swapoff -a

#或者注释
vi /etc/fstab 里面最后一行
```



### 5、k8s主节点初始化



```shell
### 建议用–control-plane-endpoint 参数指定apiserver的域名和端口，好处是后面如果要做kubernetes master node的HA，会有多个apiserver 服务器，可以用DNS做轮询或用HAproxy 做负载均衡。用域名加端口就不用在后面去改证书。kubernetes 为提升安全性，内部相互访问都会使用证书认证，这些证书会绑定IP或域名，如果直接用IP地址，后面改动–control-plane-endpoint中的IP指向，会要重新生成这些证书，而用域名就没有这个问题了，可以随时改动域名指向的IP。

kubeadm init --image-repository registry.aliyuncs.com/google_containers --control-plane-endpoint "api.k8s.songcw.com:6443" --pod-network-cidr=192.168.0.0/16

kubeadm init --kubernetes-version=1.20.5  \
--apiserver-advertise-address=192.168.200.223  \
--image-repository registry.aliyuncs.com/google_containers  \
--service-cidr=192.168.0.0/16 --pod-network-cidr=192.168.0.0/16


mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


#在node节点上加入环境变量
export KUBECONFIG=/etc/kubernetes/admin.conf

```





### 6、安装calico网络(只需要在主节点初始化)

```shell
# 注意此处需要修改calico.yaml里面的CALICO_IPV4POOL_CIDR value的网段不能和初始化master的--pod-network-cidr的网段一样，否则会出现网络问题(connect: no route host)
kubectl apply -f https://docs.projectcalico.org/v3.18/manifests/calico.yaml

watch kubectl get pods -n calico-system
```



### 7、k8s加入节点



```shell
 kubeadm join api.k8s.songcw.com:6443 --token r5cjzf.ebt3tgbo51y1wagc \
    --discovery-token-ca-cert-hash sha256:b508ddb800487d803425f2f7705ce504a2af06db2931873da6c46448e7e3208b 
    
 
  ### 查看上次生成的join token
  kubeadm token create --print-join-command

```



### 8、安装dashboard



```shell
# 允许端口访问dashboard
kubectl patch svc kubernetes-dashboard \
        -n kubernetes-dashboard \
        -p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":8443,"nodePort":30001}]}}'


cat > /etc/kubernetes/dashboard/dashboard-adminuser.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard  
EOF

## 查看admin-user账户的token
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```



### 9、安装ingress

```shell
docker pull registry.cn-beijing.aliyuncs.com/google_registry/nginx-ingress-controller:0.30.0
docker tag 89ccad40ce8e quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
docker rmi  registry.cn-beijing.aliyuncs.com/google_registry/nginx-ingress-controller:0.30.0

mkdir -p /etc/kubernetes/ingress
wget https://github.com/kubernetes/ingress-nginx/archive/nginx-0.30.0.tar.gz
tar xf nginx-0.30.0.tar.gz
cp -a ingress-nginx-nginx-0.30.0/deploy/static/mandatory.yaml ./

vim mandatory.yaml

##修改如下内容
apiVersion: apps/v1
  kind: DaemonSet   # 从Deployment改为DaemonSet
   metadata:
     name: nginx-ingress-controller
     namespace: ingress-nginx
     labels:
       app.kubernetes.io/name: ingress-nginx
       app.kubernetes.io/part-of: ingress-nginx

 #注释掉 replicas: 1
 spec:
  #replicas: 1  
  
   nodeSelector:
         kubernetes.io/hostname: k8s-master   # 修改处
# 如下几行为新加行  作用【允许在master节点运行】
         tolerations:
         - key: node-role.kubernetes.io/master
           effect: NoSchedule


    ports:
    - name: http
    containerPort: 80
    # 添加处【可在宿主机通过该端口访问Pod】
    hostPort: 80    
    protocol: TCP
    - name: https
    containerPort: 443
    # 添加处【可在宿主机通过该端口访问Pod】
    hostPort: 443   
    protocol: TCP


kubectl apply -f mandatory.yaml

[root@k8s-master ingress]# kubectl get ds -n ingress-nginx -o wide
NAME  DESIRED CURRENT   READY   UP-TO-DATE AVAILABLE NODE SELECTOR  AGE  CONTAINERS IMAGES  SELECTOR
nginx-ingress-controller   1         1         1       1       1     kubernetes.io/hostname=k8s-master   9m47s   nginx-ingress-controller   quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0   app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/part-of=ingress-nginx


```



### ingress使用http



```shell
## 创建 ingress-http.yaml

cat > /etc/kubernetes/ingress/ingress-https.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-http
  namespace: default
spec:
  rules:
    - host: app1.k8s.songcw.com
      http:
        paths:
        - path: /
          backend:
            service: 
              name: myapp-clusterip1
              port: 
                number: 80
    - host: app2.k8s.songcw.com
      http:
        paths:
        - path: /
          backend:
            service: 
              name: myapp-clusterip2
              port: 
                number: 80
EOF



```



### ingress使用https访问

```shell
mkdir -p /etc/kubernetes/ingress/cert

openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BeiJing/O=BTC/OU=MOST/CN=zhang/emailAddress=ca@test.com"

kubectl create secret tls tls-secret --key tls.key --cert tls.crt

cat > /etc/kubernetes/ingress/ingress-https.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-https
  namespace: default
spec:
  tls:
    - hosts:
      - app1.k8s.songcw.com
      - app2.k8s.songcw.com
      secretName: tls-secret
  rules:
    - host: app1.k8s.songcw.com
      http:
        paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: myapp-clusterip1
              port:
                number: 80
    - host: app2.k8s.songcw.com
      http:
        paths:
        - pathType: Prefix
          path: "/"
          backend:
            service:
              name: myapp-clusterip2
              port:
                number: 80
EOF
```



### Ingress-Nginx实现Rewrite重写



官网地址：https://kubernetes.github.io/ingress-nginx/examples/rewrite/



#### 注解说明:

nginx.ingress.kubernetes.io/rewrite-target	必须重定向的目标URL	String
nginx.ingress.kubernetes.io/ssl-redirect	指示位置部分是否只能由SSL访问(当Ingress包含证书时，默认为True)	Bool
nginx.ingress.kubernetes.io/force-ssl-redirect	即使Ingress没有启用TLS，也强制重定向到HTTPS	Bool
nginx.ingress.kubernetes.io/app-root	定义应用程序根目录，Controller在“/”上下文中必须重定向该根目录	String
nginx.ingress.kubernetes.io/use-regex	指示Ingress上定义的路径是否使用正则表达式	Bool



```shell
cat > /etc/kubernetes/ingress/ingress-rewrite.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: https://www.baidu.com
  name: rewrite
  namespace: default
spec:
  rules:
  - host: rewrite.k8s.songcw.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: myapp-clusterip1
            port: 
              number: 80
EOF
```



