# 1、ubuntu 更换源
```

/etc/apt/source.list

deb https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse

```
# 2、查看线程相关
```
ps -ef | grep -i redis 查询redis相关的进程  | 为管道符
kill -9 id 关闭进程 非正常关闭（如果是redis，数据不会写进redis中）
```

# 3、解压编译
```
tar -zxvf <解压路径> <解压的文件>
make 编译Makefile文件

```
# 4、查看指定软件包的列表和下载指定软件包的版本
```
apt-cache madison docker-ce (ubuntu用法)

apt install 软件名=版本

```
# 5、修改静态ip地址已经dns域名解析
```
ubuntu从17.10开始，已放弃在/etc/network/interfaces里固定IP的配置，即使配置也不会生效，而是改成netplan方式 ，配置写在/etc/netplan/01-netcfg.yaml或者类似名称的yaml文件里，18.04的server版本安装好以后，配置文件是：/etc/netplan/50-cloud-init.yaml，修改配置以后不用重启，执行 netplan apply 命令可以让配置直接生效


network:
  version: 2
  ethernets:
    ens33:   #配置的网卡名称
      addresses: [192.168.60.55/24]   #设置本机IP及掩码
      gateway4: 192.168.60.254   #设置网关(虚拟网卡的网关地址)
      nameservers:
          addresses: [114.114.114.114,8.8.8.8]   #设置DNS

如果宿主机是连接的是wifi，需要为NET模式配置一个端口转发ip地址为宿主机的ip地址
配置默认root登陆 vim /etc/ssh/sshd_config  PermitRootLogin yes
```

# 6、docker常用命令

```
主机复制文件到容器
sudo docker cp host_path containerID:container_path
容器复制文件到主机
sudo docker cp containerID:container_path host_path
删除所有虚悬镜像
docker rmi $(docker images -q -f dangling=true)
docker update --restart=always 容器id
宿主机访问虚拟机中的服务
查看虚拟机中的容器的hosts cat /etc/hosts

需要将虚拟机转换为静态ip地址
宿主机中添加 route -p add 172.17.0.0 mask 255.255.0.0 虚拟机ip
```

