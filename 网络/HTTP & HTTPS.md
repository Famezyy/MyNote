# HTTP & HTTPS

## 1.HTTP

<img src="img/HTTP & HTTPS/image-20220106232550783.png" alt="image-20220106232550783" style="zoom:33%;" />

<img src="img/HTTP & HTTPS/image-20220106232648764.png" alt="image-20220106232648764" style="zoom: 33%;" />



## 2.HTTPS

- 在 HTTP 基础上添加 TLS/SSL 加密，使通信不容易受到拦截和攻击
- SSL（前），TLS（后）

### 2.1 对称加密

<img src="img/HTTP & HTTPS/image-20220106230657614-16581257078814.png" alt="image-20220106230657614" style="zoom:50%;" />

### 2.2 密钥交换

<img src="img/HTTP & HTTPS/image-20220106231017368-16581257196647.png" alt="image-20220106231017368" style="zoom:50%;" />

### 2.3 证书

- 保存在源服务器的数据文件，向 certificate authority 授权中心申请
- HTTP:80 -> HTTPS:443

### 2.4 TLS握手过程

<img src="img/HTTP & HTTPS/image-20220106231545954-165812572752910.png" style="zoom:50%;" />

- 使用了对称和非对称加密算法

  <img src="img/HTTP & HTTPS/image-20220106231934547-165812574146913.png" alt="image-20220106231934547" style="zoom:50%;" />

