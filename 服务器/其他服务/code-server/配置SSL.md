# 配置 SSL

markdown 预览需要在 https 加密模式下才能在浏览器中正常使用。本章介绍使用本地签名证书。如果有自己的域名以及 DNS，可以参考[官方教程](https://coder.com/docs/code-server/latest/guide#using-lets-encrypt-with-nginx)用 NGINX 配置.

## 1.安装

软件地址：https://github.com/coder/code-server/releases

## 2.配置

### 2.1 生成证书

```bash
# -cert-file [filename] 生成对应 crt 文件
# -cert-key [filename] 生成对应 key
$ mkcert -cert-file phone-code-server.crt -key-file phone-code-server.key 192.168.11.9 127.0.0.1
```

### 2.2 配置ssh证书

在 code-server 的配置文件 config.yaml 中配置 ssh 证书

```bash 
bind-addr: 127.0.0.1:8080
auth: password
password: 9e0369313ce44e7001249cfb
cert: /config/ssl/code-server.crt
cert-key: /config/ssl/code-server.key
```

再次启动 code-server，可以通过内网网址访问。

### 2.3 安装证书

通过 chrome 访问时，会提示不安全的证书，需要将 mkcert 的 CA 证书安装到访问的主机上。

通过 mkcert 命令查找 CA 证书位置

```bash
$ mkcert -CAROOT
~/.local/share/mkcert
```

进入对应文件夹，找到 rootCA.pem 文件，将其后缀修改为 .crt，拷贝到客户端。在 windows 中可以通过双击安装，安装时，要选择**受信任的根证书颁发机构**。

在ipad上的安装见官网教程：[Sharing a self-signed certificate with an iPad](https://coder.com/docs/code-server/latest/ipad#sharing-a-self-signed-certificate-with-an-ipad)

> **补充：删除证书**
>
> 删除本地证书的方法：`win+R`输入`certmgr.msc`。