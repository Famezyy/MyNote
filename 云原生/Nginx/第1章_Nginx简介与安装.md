# 第1章_Nginx简介与安装

## 1.简介

Nginx 是一个高性能的 HTTP 和反向代理 Web 服务器，由伊戈尔·赛索耶夫开发，源代码以类 BSD 许可证的形式发布，第一个公开版本 0.1.0 发布于 2004 年 10 月 4 日。Nginx 因高稳定性、丰富的功能集、内存消耗少、并发能力强而闻名。Nginx 相关地址如下：

```bash
源码地址：https://trac.nginx.org/nginx/browser
官网地址：https://www.nginx.org/
```

Nginx 有以下 3 个主要社区分支：

1. Nginx 官方版本
   
   更新迭代快，提供免费版本和商业版本。

2. Tengine
   
   Tengine 是由淘宝网发起的 Web 服务器项目。在 Nginx 的基础上针对大访问量网站的需求添加了很多高级功能和特性。

3. OpenResty
   
   一个基于 Nginx 与 Lua 的高性能 Web 平台，由章亦春老师开发，内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项，用于方便地搭载能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。
   
   OpenResty 的目标是让 Web 服务直接跑在 Nginx 服务内部，充分利用 Nginx 的非阻塞 I/O 模型，不仅对 HTTP 客户端请求，甚至对远程后端（MySQL、Redis 等）都进行一致的高性能响应。

   并且开发人员可以使用 Lua 脚本语言调动 Nginx 支持的各种 C 以及 Lua 模块，快速构造出足以胜任 10KB～1000KB 以上单机并发连接的高性能 Web 应用。

   ```bash
   官网地址：https://openresty.org/cn/
   组件地址：https://openresty.org/cn/components.html
   ```

### 1.1 正向代理与反向代理

正向代理的最大特点是客户端知道目标服务器的地址，需要配置目标服务器信息，如 IP 和端口。一般来说，正向代理服务器是一台与客户端网络连通的局域网内部的机器或者是可以打通两个隔离网络的**双网卡**机器。通过正向代理的方式，客户端和 HTTP 请求可以转发到之前与客户端网络不通的其他不同的目标服务器。

反向代理与正向代理相反，客户端不知道目标服务器的信息，不需要进行特别的设置。客户端向反向代理服务器直接发送请求，接着反向代理将请求转发给目标服务器，并将目标服务器的响应结果按原路返回给客户端。

通俗点说，正向代理是对客户端的伪装，隐藏了客户端的 IP、头部或者其他信息，服务器得到的是伪装过的客户端信息；反向代理是对目标服务器的伪装，隐藏了目标服务器的 IP、头部或者其他信息。

## 2.Nginx的启动与停止

后续案例主要在 Windows 系统上演示，使用的是 32 位的 OpenResty 版本（64 位的版本进行 Lua 调试会发生断点不能命中的情况）。

### 2.1 Nginx的启动命令和参数

OpenResty 的原始启动命令为 nginx，其参数有`-v`、`-t`、`-p`、`-c`、`-s`等。

- `-v`：查看 Nginx 版本
  
- `-c`：指定一个新的 Nginx 配置文件来代替默认的 Nginx 配置文件
  
  ```bash
  $ nginx -c nginx-debug.conf
  ```

- `-t`：测试 Nginx 的配置文件语法是否正确，不会运行配置文件
  
  ```bash
  $ nginx -t -c nginx-debug.conf
  nginx: the configuration file ./nginx-debug.conf syntax is ok
  nginx: configuration file ./nginx-debug.conf test is successful
  ```

- `-p`：设置前缀路径
  
  ```bash
  $ nginx -p ./ -c nginx-debug.conf
  ```

  `-p ./`表示将当前目录作为前缀路径，及 nginx-debug.conf 配置文件中所用到的相对路径都加上这个前缀。

- `-s`：给 Nginx 进程发送信号，包含`stop`、`reload`、`quit`。
  
  ```bash
  $ nginx -p ./ -c nginx-debug.conf -s reload
  $ nginx -p ./ -c nginx-debug.conf -s stop
  $ nginx -p ./ -c nginx-debug.conf -s quit
  ```

### 2.2 通过包管理器安装Openresty

参考：https://openresty.org/cn/linux-packages.html

#### 1.Ubuntu

**（1）安装导入 GPG 公钥时所需的几个依赖包（整个安装过程完成后可以随时删除它们）**

```bash
sudo apt-get -y install --no-install-recommends wget gnupg ca-certificates
```

**（2）导入 GPG 密钥**

- ubuntu 16 ~ 20 版本
  
  ```bash
  wget -O - https://openresty.org/package/pubkey.gpg | sudo apt-key add -
  ```

- ubuntu 22 及以上版本
  
  ```bash
  wget -O - https://openresty.org/package/pubkey.gpg | sudo gpg --dearmor -o /usr/share/keyrings/openresty.gpg
  ```

**（3）添加官方 APT 仓库**

对于 x86_64 或 amd64 系统，可以使用下面的命令：

- ubuntu 16 ~ 20 版本
  
  ```bash
  echo "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main" \
  | sudo tee /etc/apt/sources.list.d/openresty.list
  ```

- ubuntu 22 及以上版本
  
  ```bash
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/openresty.gpg] http://openresty.org/package/ubuntu $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/openresty.list > /dev/null
  ```
  而对于 arm64 或 aarch64 系统，则可以使用下面的命令:

- ubuntu 16 ~ 20 版本
  
  ```bash
  echo "deb http://openresty.org/package/arm64/ubuntu $(lsb_release -sc) main" \
  | sudo tee /etc/apt/sources.list.d/openresty.list
  ```

- ubuntu 22 及以上版本
  
  ```bash
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/openresty.gpg] http://openresty.org/package/arm64/ubuntu $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/openresty.list > /dev/null
  ```

**（4）更新 APT 索引**

```bash
sudo apt-get update
```

然后就可以像下面这样安装软件包：

```bash
sudo apt-get -y install openresty
```

这个包同时也推荐安装`openresty-opm`和`openresty-restydoc`包，所以后面两个包会缺省安装上。如果你不想自动关联安装，可以用下面方法关闭自动关联安装：

```bash
sudo apt-get -y install --no-install-recommends openresty
```

#### 2.CentOS

运行下面的命令就可以添加仓库（对于 CentOS 8 或以上版本，应将下面的`yum`都替换成`dnf`）：

- CentOS 9 或者更新版本
  
  ```bash
  # add the yum repo:
  wget https://openresty.org/package/centos/openresty2.repo
  sudo mv openresty2.repo /etc/yum.repos.d/openresty.repo
  
  # update the yum index:
  sudo yum check-update
  ```

- CentOS 8 或者更老版本
  
  ```bash
  # add the yum repo:
  wget https://openresty.org/package/centos/openresty.repo
  sudo mv openresty.repo /etc/yum.repos.d/openresty.repo
  
  # update the yum index:
  sudo yum check-update
  ```

然后就可以像下面这样安装软件包，比如 openresty：

```bash
sudo yum install -y openresty
```

如果你想安装命令行工具`resty`，那么可以像下面这样安装`openresty-resty`包：

```bash
sudo yum install -y openresty-resty
```


命令行工具`opm`在`openresty-opm`包里，而`restydoc`工具在`openresty-doc`包里头。

列出所有 openresty 仓库里头的软件包：

```bash
sudo yum --disablerepo="*" --enablerepo="openresty" list available
```

### 2.3 包管理器安装普通Nginx

```bash
yum install -y epel-release
yum install -y nginx
```

### 2.4 编译安装普通版本

#### 1.下载并解压文件

```bash
tar -zxvf nginx-1.21.6.tar.gz
```

#### 2.编译

```bash
./configure --prefix=/usr/local/nginx
make & make install
```

> **扩展**
>
> 可在编译时只添加需要的模块：
>
> ```bash
> ./configure --prefix=/usr/local/nginx --without-mail_pop3_module --without-mail_imap_module --without-mail_smtp_module 
> ```
> 
> 使用`./configure --help`查看所有选项。

> **注意**
>
> - 如果出现警告或报错：
>
>   ```bash
>   checking for OS
>    + Linux 3.10.0-693.el7.x86_64 x86_64
>   checking for C compiler ... not found
>   ./configure: error: C compiler cc is not found
>   ```
>
>   需要安装`gcc`
>
>   ```bash
>   yum install -y gcc
>   ```
>
>
> - 如果提示以下错误：
>
>   ```bash
>   ./configure: error: the HTTP rewrite module requires > the PCRE library.
>   You can either disable the module by using --without-http_rewrite_module
>   option, or install the PCRE library into the system, or build the PCRE library
>   statically from the source with nginx by using --with-pcre=<path> option.
>   ```
>
>   需要安装`pcre`库
>
>   ```bash
>   yum install -y pcre pcre-devel
>   ```
>
> - 如果提示：
>
>   ```bash
>   ./configure: error: the HTTP gzip module requires the zlib library.
>   You can either disable the module by using --without-http_gzip_module
>   option, or install the zlib library into the system, or build the zlib library
>   statically from the source with nginx by using --with-zlib=<path> option.
>   ```
>
>   需要安装`zlib`库
>
>   ```bash
>   yum install -y zlib zlib-devel
>   ```

#### 3.安装成系统服务

**（1）创建服务脚本**

```bash
vi /usr/lib/systemd/system/nginx.service
```

**（2）服务脚本内容**

```bash
[Unit]
Description=nginx - web server
After=network.target remote-fs.target nss-lookup.target
[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
ExecQuit=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true
[Install]
WantedBy=multi-user.target
```

**（3）重新加载系统服务**

```bash
systemctl daemon-reload
```

**（4）启动服务**

```bash
systemctl start nginx.service
```

**（5）开机启动**

```bash
systemctl enable nginx.service
```

### 2.5 启动Nginx

进入安装好的目录`/usr/local/nginx/sbin`

```bash
# 默认后台启动
./nginx 启动
./nginx -s stop 快速停止
./nginx -s quit 优雅关闭，在退出前完成已经接受的连接请求
./nginx -s reload 重新加载配置
```

### 2.6 防火墙设置

**关闭防火墙**

```bash
systemctl stop firewalld.service
```

**禁止防火墙开机启动**

```bash
systemctl disable firewalld.service
```

**放行端口**

```bash
firewall-cmd --zone=public --add-port=80/tcp --permanent
```

**重启防火墙**

```bash
firewall-cmd --reload
```

