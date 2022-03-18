### 部署时注意整改的内容：
1. master高可用改为keepalived+api server，可以去掉nginx，keepalived检测6443端口
1. calico默认采用vxlan模式
1. 以DS方式启动ingress controller

--------------------------------------------------------------------------------------------------

本文档删除了传统部署的多活master的HA配置，使用VIP做故障切换


0、主机名规划

```
10.15.6.66 MasterA
10.15.6.75 MasterB
10.15.6.74 MasterC
10.15.6.76 NodeA
10.15.6.77 NodeB
10.15.6.78 NodeC
```


1、更改主机名

```
hostnamectl set-hostname MasterA
hostnamectl set-hostname MasterB
hostnamectl set-hostname MasterC
hostnamectl set-hostname NodeA
hostnamectl set-hostname NodeB
hostnamectl set-hostname NodeC
```


2、执行初始化

```
yum install wget -y

cd /root && mkdir /root/shell && wget https://jerry.plus/1a2699623e1b8b111676fe63de6551e5 -O /root/shell/init-server.sh

sh /root/shell/init-server.sh

timedatectl set-timezone Asia/Shanghai

yum update -y

升级内核，参考https://linux.cn/article-8310-1.html  *升级内核会导致NFS挂载失败，正式环境升级内核需要注意*
```


3、安装keepalived

`yum install keepalived -y `

4、安装docker

```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast

yum install -y docker-ce-18.09.9-3.el7

mkdir -p /etc/docker

cat > /etc/docker/daemon.json <<EOF
{
             "registry-mirrors": ["https://q2uvt0x7.mirror.aliyuncs.com"],
             "bip": "1.1.0.1/24",
             "data-root": "/data/docker",
             "exec-opts": ["native.cgroupdriver=systemd"],
             "storage-driver": "overlay2", 
             "storage-opts":["overlay2.override_kernel_check=true"],
             "log-driver": "json-file", 
             "log-opts": { 
               "max-size": "100m", 
               "max-file": "10" 
              }
}
EOF

systemctl enable docker && systemctl daemon-reload && systemctl start docker
```

## 5服务器设置白名单 
````html
假如采用传统请执行一下命令：

systemctl stop firewalld
systemctl mask firewalld

并且安装iptables-services：
yum install iptables-services

设置开机启动：
systemctl enable iptables

iptables启动、关闭、保存：

systemctl [stop|start|restart] iptables
#or
service iptables [stop|start|restart]


service iptables save
#or
/usr/libexec/iptables/iptables.init save
````



### 5.1 MasterA 举例
```shell script
vim /etc/sysconfig/iptables


*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
#这里开始增加白名单服务器ip(请删除当前服务器的ip地址)
-N whitelist
#-A whitelist -s 10.15.6.66 -j ACCEPT
-A whitelist -s 10.15.6.75 -j ACCEPT
-A whitelist -s 10.15.6.74 -j ACCEPT
-A whitelist -s 10.15.6.76 -j ACCEPT
-A whitelist -s 10.15.6.77 -j ACCEPT
-A whitelist -s 10.15.6.78 -j ACCEPT
#这里结束白名单服务器ip
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 30771 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT  
-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j ACCEPT 
#上面这些 ACCEPT 端口号，公网内网都可以访问

#下面这些 whitelist 端口号，仅限服务器之间通过内网访问
#这里添加为白名单ip开放的端口
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 30771 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 6443 -j whitelist
#这结束为白名单ip开放的端口
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited

COMMIT
```

````shell script
service iptables restart

````

### 5.2 MasterB 举例
```shell script
vim /etc/sysconfig/iptables


*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
#这里开始增加白名单服务器ip(请删除当前服务器的ip地址)
-N whitelist
-A whitelist -s 10.15.6.66 -j ACCEPT
-A whitelist -s 10.15.6.75 -j ACCEPT
#-A whitelist -s 10.15.6.74 -j ACCEPT
-A whitelist -s 10.15.6.76 -j ACCEPT
-A whitelist -s 10.15.6.77 -j ACCEPT
-A whitelist -s 10.15.6.78 -j ACCEPT
#这里结束白名单服务器ip
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 30771 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT  
-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j ACCEPT 
#上面这些 ACCEPT 端口号，公网内网都可以访问

#下面这些 whitelist 端口号，仅限服务器之间通过内网访问
#这里添加为白名单ip开放的端口
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 30771 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 6443 -j whitelist
#这结束为白名单ip开放的端口
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited

COMMIT
```

````shell script
service iptables restart

````
### 5.3 NodeA 举例
```shell script
vim /etc/sysconfig/iptables


*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
#这里开始增加白名单服务器ip(请删除当前服务器的ip地址)
-N whitelist
-A whitelist -s 10.15.6.66 -j ACCEPT
-A whitelist -s 10.15.6.75 -j ACCEPT
-A whitelist -s 10.15.6.74 -j ACCEPT
#-A whitelist -s 10.15.6.76 -j ACCEPT
-A whitelist -s 10.15.6.77 -j ACCEPT
-A whitelist -s 10.15.6.78 -j ACCEPT
#这里结束白名单服务器ip
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 30771 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT  
-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j ACCEPT 
#上面这些 ACCEPT 端口号，公网内网都可以访问

#下面这些 whitelist 端口号，仅限服务器之间通过内网访问
#这里添加为白名单ip开放的端口
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 30771 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 6443 -j whitelist
#这结束为白名单ip开放的端口
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited

COMMIT
```

````shell script
service iptables restart

````

### 5.4 NodeB 举例
```html
vim /etc/sysconfig/iptables

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
#这里开始增加白名单服务器ip(请删除当前服务器的ip地址)
-N whitelist
-A whitelist -s 10.15.6.66 -j ACCEPT
# -A whitelist -s 10.15.6.75 -j ACCEPT
-A whitelist -s 10.15.6.74 -j ACCEPT
-A whitelist -s 10.15.6.76 -j ACCEPT
-A whitelist -s 10.15.6.77 -j ACCEPT
-A whitelist -s 10.15.6.78 -j ACCEPT
#这里结束白名单服务器ip
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 30771 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT  
-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j ACCEPT 
#上面这些 ACCEPT 端口号，公网内网都可以访问

#下面这些 whitelist 端口号，仅限服务器之间通过内网访问
#这里添加为白名单ip开放的端口
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 30771 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j whitelist
-A INPUT -m state --state NEW -m tcp -p tcp --dport 6443 -j whitelist
#这结束为白名单ip开放的端口
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited

COMMIT

```
````shell script
service iptables restart

````


6、下载需要的k8s软件，本文档采用1.19.9

`yum install -y kubelet-1.19.9 kubeadm-1.19.9 kubectl-1.19.9`

7、在master上下载编译好的kubeadm二进制文件，暂时在运维的群共享里

`wget https://jerry.plus/fb620d15e174ca5cb4388b5a75d1ee18 -O ./kubeadm`

8、配置VIP

使用非抢占模式，防止某个节点的apiserver重启导致频繁故障切换
由于k8s本身对apiserver存在健康检查机制，keepalived本身无需配置健康检查，但需要在计划任务配置脚本检测keepalived本身
```
vi /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state BACKUP   
    interface eth0  # 网卡名称，CentOS 7 为系统识别总线自生成，以实际网卡名为准
    nopreempt
virtual_router_id 51
    priority 100   # 其他主机修改为100以下
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass xxxxxxxx  # 认证密码，多个vrrp组请修改此密码
    }
    virtual_ipaddress {
        xx.xx.xx.xx/32  # VIP地址
    }
}
```



9、创建kubeadm配置文件

```
vim kubeadm-config.yaml

apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.19.9
controlPlaneEndpoint: "xx.xx.xx.xx:6443"
imageRepository: ccr.ccs.tencentyun.com/tcb-100024104308-urur
networking:
  dnsDomain: cluster.local
  serviceSubnet: "172.31.2.0/24"
  podSubnet: "172.31.1.0/24"
apiServer:
  certSANs:
  - 10.15.6.66
  - 10.15.6.75
  - 10.15.6.74
  - 10.15.6.92
  timeoutForControlPlane: 4m0s
  extraVolumes:
  - name: localtime
    hostPath: /etc/localtime
    mountPath: /etc/localtime
    readOnly: true
    pathType: File
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: 
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    readOnly: true
    pathType: File
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
scheduler: 
  extraVolumes:
  - hostPath: /etc/localtime
    mountPath: /etc/localtime
    name: localtime
    readOnly: true
    pathType: File
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
```


10、初始化master（使用新上传的kubeadm，解决证书1年后过期问题）

`./kubeadm init --config kubeadm-config.yaml --upload-certs`


11、初始化其他master（使用新上传的kubeadm，解决证书1年后过期问题）

```
./kubeadm join xx.xx.xx.xx:6443 --token 1q3wpz.u1damy0rh12u0l5h \
    --discovery-token-ca-cert-hash sha256:9ceb1590eded8d2ad4404505c4be4d79cc64d4e79a0351aed5bddb938708d40c \
    --control-plane --certificate-key 73bff999c21a8d469e071f946ff0fb283f3d23f72295d00b130abdc12293efe6
```

12、工作节点加入集群（使用新上传的kubeadm，解决证书1年后过期问题）

```
./kubeadm join xx.xx.xx.xx:6443 --token 1q3wpz.u1damy0rh12u0l5h \
    --discovery-token-ca-cert-hash sha256:9ceb1590eded8d2ad4404508c4be4d79cc64d4e79a0351aed5bddb938708d40c
```


13、部署calico网络插件（上传文件做链接）

上传calico定义文件，然后kubectl apply -f 应用

文件位置：calico-vxlan.yaml

14、完成其他部署（rancher、ingress）

（1）ingress-controller

上传ingress定义文件，然后kubectl apply -f 应用

文件位置：ingress-controller.yaml

（2）rancher

```
wget https://get.helm.sh/helm-v3.2.0-linux-amd64.tar.gz
tar zxvf helm-v3.2.0-linux-amd64.tar.gz && mv linux-amd64/helm /usr/bin/helm


helm repo add rancher-stable http://rancher-mirror.oss-cn-beijing.aliyuncs.com/server-charts/stable

kubectl create namespace cattle-system

helm install rancher rancher-stable/rancher \
 --namespace cattle-system \
 --set hostname=xx.xx.xx.xx.nip.io  \
 --set ingress.tls.source=secret \
 --set rancherImageTag=v2.5.6

```