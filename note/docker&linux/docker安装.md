## 1、卸载原来老版本的docker
> sudo apt-get remove docker docker-engine docker.io containerd runc

## 2、更新源
> sudo apt-get update

## 3、下载需要的包
> sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common

## 4、添加docker GPG钥匙
> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

## 5、打印key 
> sudo apt-key fingerprint 0EBFCD88

## 6、添加源
> sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

## 7、更新源
> sudo apt-get update

## 8、查看历史版本
> apt-cache madison docker-ce

## 9、下载对应版本
```java
sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io 或者 sudo apt-get -y install docker-ce
```

## 10、镜像加速器
```java
vim /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```