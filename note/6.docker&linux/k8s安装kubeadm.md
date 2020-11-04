# 安装kubeadm （一主两从，不具备高可用）
### 1、前期准备
```
需要关闭磁盘交换空间，并且虚拟机最小1CPU2核
sudo swapoff -a
注释掉 /etc/fstab 中的 swap （虚拟技术中磁盘交换会很废空间，不允许开启）
关闭防火墙  ufw disable
```
### 2、安装docker
```

```
### 3、kubeadm 快速搭建k8s环境
```
配置静态master ip地址，因为虚拟机中ip地址是为动态ip地址
编辑 vi /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        ens33:
          # 我的 Master 是 150 - 152，Node 是 160 - 162
          addresses: [192.168.141.150/24]
          gateway4: 192.168.141.2
          nameservers:
            addresses: [192.168.141.2]
    version: 2
netplan apply 配置生效

配置kubeadm：
    导出集群配置文件：kubeadm config print init-defaults --kubeconfig ClusterConfiguration > kubeadm.yml
    编辑配置文件：修改advertiseAddress为当前节点的地址，kubernetesVersion（很重要一定要跟下载的版本一致，否整下载下来的镜像文件跑不起来），imageRepository（镜像地址 registry.aliyuncs.com/google_containers）
    podSubNet（后期配置网络）
查看所需镜像列表
kubeadm config images list --config kubeadm.yml
拉取镜像
kubeadm config images pull --config kubeadm.yml
初始化
kubeadm init --config=kubeadm.yml --upload-certs | tee kubeadm-init.log

kubeadm 获取token  kubeadm token list
```

### 4、slave加入主节点
```
kubeadm join 命令
```