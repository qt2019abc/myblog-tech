
# 服务器docker配置代理拉取镜像方法

背景：目前国内所有公共 Docker 镜像源都不稳定。 如果需要再服务器上拉取docker镜像会失败。 解决方法，让服务器docker配置本地的vpn代理。 

## 1.  创建 Docker 服务的 systemd 配置目录（如果不存在）：

```
sudo mkdir -p /etc/systemd/system/docker.service.d
```

## 2.  创建代理配置文件：

```
sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf # 写入以下内容（替换成你的真实代理地址）：

[Service]Environment="HTTP_PROXY=http://your-proxy-ip:port"
Environment="HTTPS_PROXY=http://your-proxy-ip:port"
Environment="NO_PROXY=localhost,127.0.0.1,::1"
```

从vpn代理软件获得系统代理设置的本地代理端口， 例如127.0.0.1:7897

服务器上需要能够访问到IP:7897端口


## 3.  重新加载 systemd 并重启 Docker：

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 4.  验证代理是否生效：

```
sudo systemctl show --property=Environment docker

# output:
Environment=HTTP_PROXY=http://192.168.21.41:7897 HTTPS_PROXY=http://192.168.21.41:7897 NO_PROXY=localhost,127.0.0.1,::1
```

## 5.  再次拉取镜像

这样就配置好了服务器走本地的代理了，可以正常拉取镜像。 