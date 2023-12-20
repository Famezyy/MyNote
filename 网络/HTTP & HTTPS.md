---
typora-copy-images-to: ./img/HTTP & HTTPS
---

# HTTP & HTTPS

## 1.HTTP

<img src="img/HTTP & HTTPS/202309261453368.png" alt="image-20220106232550783" style="zoom:33%;" />

<img src="img/HTTP & HTTPS/202309261453369.png" alt="image-20220106232648764" style="zoom: 33%;" />



## 2.HTTPS

HTTPS（Hypertext Transfer Protocol Secure）在 HTTP 基础上添加 TLS/SSL 加密（先 SSL，再 TLS），使通信不容易受到拦截和攻击，它使用混合加密，包括对称加密和非对称加密。

1. **对称加密**

   对称加密使用相同的密钥（称为会话密钥），用于加密和解密通信的数据。这种加密算法的主要优点是速度较快，因为加密和解密使用相同的密钥。然而，密钥的安全分发可能是一个挑战。

2. **非对称加密**

   非对称加密使用一对密钥，分为公钥和私钥。公钥用于加密数据，而私钥用于解密数据。HTTPS 中的非对称加密通常用于建立安全的通信通道。服务器拥有私钥，而公钥是公开的。客户端使用服务器的公钥加密数据，只有服务器拥有相应的私钥才能解密它。

在 HTTPS 握手过程中，通常会使用非对称加密来确保密钥的安全分发。一旦安全通道建立，会话密钥就会用对称加密算法进行加密和解密，以提高通信的效率。这种混合使用对称和非对称加密的方法能够充分发挥两者的优点，提供了安全且高效的通信机制。

**密钥交换分发**

<img src="img/HTTP & HTTPS/202309261453372.png" alt="image-20220106231017368" style="zoom: 33%;" />

**对称密钥加解密**

<img src="img/HTTP & HTTPS/202309261453370.png" alt="image-20220106230657614" style="zoom: 33%;" />

**TLS握手过程**

<img src="img/HTTP & HTTPS/202309261453373.png" style="zoom:50%;" />

使用对称和非对称加密算法：

<img src="img/HTTP & HTTPS/202309261453374.png" alt="image-20220106231934547" style="zoom: 50%;" />

