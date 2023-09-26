# HTTP & HTTPS

## 1.HTTP

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202309261453368.png" alt="image-20220106232550783" style="zoom:33%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202309261453369.png" alt="image-20220106232648764" style="zoom: 33%;" />



## 2.HTTPS

- 在 HTTP 基础上添加 TLS/SSL 加密，使通信不容易受到拦截和攻击
- SSL（前），TLS（后）

### 2.1 对称加密

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202309261453370.png" alt="image-20220106230657614" style="zoom:50%;" />

### 2.2 密钥交换

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202309261453372.png" alt="image-20220106231017368" style="zoom:50%;" />

### 2.3 证书

- 保存在源服务器的数据文件，向 certificate authority 授权中心申请
- HTTP:80 -> HTTPS:443

### 2.4 TLS握手过程

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202309261453373.png" style="zoom:50%;" />

- 使用了对称和非对称加密算法

  <img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/202309261453374.png" alt="image-20220106231934547" style="zoom:50%;" />

