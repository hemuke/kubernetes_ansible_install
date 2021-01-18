k8s文档安装 ansible版本
1）环境说明
所有服务器必须是centos7，并且需要修改主机名hostnamectl set-hostname 和下面列表相对应
192.168.135.158 k8s-ansible
192.168.135.70 k8s-master01
192.168.135.109 k8s-master02
192.168.135.106 k8s-master03
192.168.135.6 k8s-node01
192.168.135.149 k8s-node02
192.168.135.119 k8s-master-lb

2）安装ansible，配置ansible，相关操作初始化
在192.168.135.158 k8s-ansible上面操作
a）yum install -y ansible

b）将ansible.zip解压覆盖原/etc/ansible目录

c）修改主机名hostnamectl set-hostname k8s-ansible

d）修改 cat /etc/hosts
127.0.0.1    localhost localhost.localdomain localhost4 localhost4.localdomain4
::1          localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.135.70 k8s-master01
192.168.135.109 k8s-master02
192.168.135.106 k8s-master03
192.168.135.6 k8s-node01
192.168.135.149 k8s-node02
192.168.135.119 k8s-master-lb
192.168.135.158 k8s-ansible

e）k8s-ansible生成秘钥对，并且也要给自己服务器授权免秘钥登录，剩余的k8s-master k8s-node的服务器也必须要免秘钥登录
ssh-keygen -t rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master01
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master02
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master03
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node01
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node02
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-ansible

从k8s-ansible服务器上面登录每个服务器k8s-master01 k8s-master02 k8s-master03 k8s-node01 k8s-node02并将他们的主机名修改成对应的名字

f）修改/etc/ansible/hosts.yml全局变量文件
#需要修改IP地址
[etcd]
etcd01 ansible_ssh_host=192.168.135.70
etcd02 ansible_ssh_host=192.168.135.109
etcd03 ansible_ssh_host=192.168.135.106
#需要修改IP地址
[k8snode]
k8s-node01 ansible_ssh_host=192.168.135.6
k8s-node02 ansible_ssh_host=192.168.135.149

#需要修改IP地址
[k8sansible]
k8s-ansible ansible_ssh_host=192.168.135.158

[all:vars]
#需要修改IP地址
#k8s主节点1的IP地址
k8sMaster01=192.168.135.70

#需要修改IP地址
#k8s主节点2的IP地址
k8sMaster02=192.168.135.109

#需要修改IP地址
#k8s主节点3的IP地址
k8sMaster03=192.168.135.106

#可以不修改
#k8s service的IP地址
k8sService=10.96.0.1

#需要修改
#k8s的负载均衡的地址
k8sMasterLB=192.168.135.119

#不需要修改
#etcd k8s证书的存放路径
preInstallPath=/etc/ansible/roles/4.pre_install/files/pki

#可以不修改
#k8sService的IP段
k8sServiceSection=10.96.0.0/16

#可以不修改
#k8sPod的IP段
podIPSection=10.95.0.0/16

#可以不修改
#k8sService的DNS地址
k8sServiceDNS=10.96.0.10

#不需要修改
#bootstrap的存放路径
bootstrapPath=/etc/ansible/roles/6.k8s/files

#不需要修改
#calico ETCD_CA
etcdCA=`cat /etc/etcd/ssl/etcd-ca.pem | base64 | tr -d '\n'`

3）因为单位使用的是F5负载均衡器需要事先配置好，如果没有F5，需要事先安装好负载均衡器，为了试验方便这里用nginx代替了，也可以用keepalived+haproxy。
192.168.135.119
a）修改主机名 hostnamectl set-hostname k8s-master-lb
b）安装nginx 	配置nginx镜像源	yum install -y nginx
c）安装时间同步服务器	yum install -y chrony
d）关闭防火墙
e）配置nginx配置文件并启动,nginx的系统层面优化 nginx配置文件优化略
cat /etc/nginx/nginx.conf
user  nginx;
worker_processes  8;
worker_cpu_affinity auto;
worker_rlimit_nofile 30000;
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    use epoll;
    worker_connections  30000;
    multi_accept on;
}
stream {
    log_format basic '$remote_addr [$time_local]'
                     '$protocol $status $bytes_sent $bytes_received'
                     '$session_time $upstream_addr'
                     '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';
    access_log /var/log/nginx/k8s.log basic buffer=32k;
    upstream backend {
            server 192.168.135.70:6443 max_fails=2 fail_timeout=10s;
            server 192.168.135.109:6443 max_fails=2 fail_timeout=10s;
            server 192.168.135.106:6443 max_fails=2 fail_timeout=10s;
    }
    server {
            listen 8443;
            proxy_next_upstream on;
            proxy_next_upstream_timeout 0;
            proxy_next_upstream_tries 0;
            proxy_connect_timeout 1s;
            proxy_timeout 1m;
            proxy_upload_rate 0;
            proxy_download_rate 0;
            proxy_pass backend;
    }
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    tcp_nopush     on;
    keepalive_timeout  65;
    gzip  on;
    include /etc/nginx/conf.d/*.conf;
}

4）脚本机上面开始执行脚本
登录k8s-ansible

a）cd /etc/ansible&&ansible all -m ping
测试免秘钥登录

b）cat /etc/ansible/all.yml，2.mount角色为我单位内部使用，一般需要注释
[root@k8s-ansible ansible]# cat all.yml 
- hosts: etcd,k8snode
  roles:
    - 0.configureEtcHost
    - 1.base
#    - 2.mount
    - 3.docker

#- hosts: etcd,k8snode,k8sansible
#  roles:
#    - 4.pre_install
#    - 5.etcd
#    - 6.k8s

c）执行ansible-playbook all.yml -f 5 建议先安装基础组件

d）cat /etc/ansible/all.yml，2.mount角色为我单位内部使用，一般需要注释
[root@k8s-ansible ansible]# cat all.yml 
#- hosts: etcd,k8snode
#  roles:
#    - 0.configureEtcHost
#    - 1.base
#    - 2.mount
#    - 3.docker

- hosts: etcd,k8snode,k8sansible
  roles:
    - 4.pre_install
    - 5.etcd
    - 6.k8s

e）执行ansible-playbook all.yml -f 5 安装k8s相关组件，安装完检查

ansible 角色7为添加节点

注意事项
1) 单独部署一台服务器并且安装ansible并命名k8s-ansbile,并且要和k8s-master k8s-node免密登录
2) 这套ansible安装采取的是master和etcd在同一个节点，并且只支持3个节点的k8s-master和etcd
3)metric-server的yaml文件里面是修改过两段配置文件的目的是让metric-server去链接IP以及访问api-server的时候能够进行聚合证书认证
    spec:
      containers:
        - args:
            - --cert-dir=/tmp
            - --secure-port=4443
            - --metric-resolution=30s
            - --kubelet-insecure-tls
            - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
            - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.pem # change to front-proxy-ca.crt for kubeadm
            - --requestheader-username-headers=X-Remote-User
            - --requestheader-group-headers=X-Remote-Group
            - --requestheader-extra-headers-prefix=X-Remote-Extra-
4)calico 更改了镜像地址，以及pod网段IP地址
5)coredns 更改了镜像地址，以及service网段IP地址
6)kubelet的服务添加了安全参数 可以根据自己的需求修改
