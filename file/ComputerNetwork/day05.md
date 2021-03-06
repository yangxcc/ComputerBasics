## HTTPS如何优化

### 分析性能损耗

首先，在分析HTTPS有哪些方面可以优化的时候，我们先要来看一下HTTPS相较于HTTP在哪些方面存在性能开销

- **SSL/TLS握手阶段的开销**
- **握手后，对称加密报文的传输**

更详细一点，针对SSL/TLS握手阶段的性能开销问题，TLS握手过程不仅增加了网络时延（2RTT），而且握手过程中的一些步骤也会产生性能损耗，比如：

- 对于ECDHE算法而言，握手过程中客户端和服务端都要临时生成椭圆曲线的公私钥
- 客户端在验证服务器的数据证书的时候，会访问CA的CRL或者OCSP，目的是验证服务器证书有没有被吊销
- 最后生成会话密钥的时候（pre-master），也是需要经过计算

而针对握手之后，通信报文的加密，现在主流的AES，ChaCha20性能都是非常不错的了，而且一些CPU厂商还针对这个方面在硬件层面做了优化，因此这个环节的损耗比较小



> **<font color=red>那么HTTPS到底可以从那几个方面进行优化呢？</font>**



### 硬件优化

因为HTTPS是计算密集型，不是I/O密集型，所以我们应该针对CPU进行优化，而不是对网卡、硬盘等设备进行优化

一个好的CPU可以提高计算性能，因为HTTPS在连接过程中存在大量的计算过程，而且，我们在选择CPU的时候，尽量**选择支持`AES-NI`特性的CPU**，因为这种款式的CPU在指令级别优化了AES算法，所以能够加速数据通信过程中的加解密

```shell
# 如果服务器是linux，那么可以使用以下指令查看cpu是否支持AES-NI指令集
sort -u /proc/crypto |grep module |grep aes
```



### 软件优化

对于软件优化，我们可以把旧版本的软件升级成新版本的，因为新版本的软件通常会对旧版本做一些优化同时修复一些bug，条件一些功能等，比如将Linux2.x升级成Linux4.x ... ，因为新版本的软件不仅会提供新的特性还会修复老版本中存在的问题等

但是软件升级和硬件优化一样，是属于一个“牵一发而动全身“的大工程，因此这两种优化不仅会消耗大量的人力物力，还有可能会造成出现一些不必要的麻烦，影响产品上线



### 协议优化

在对HTTPS的优化中，对于协议的优化其实就是对于密钥交换过程的优化，可以分成下面的两个部分

#### 密钥交换算法的优化

对于密钥交换算法，我们尽量选择ECDHE，因为在TLS 1.2中，RSA需要4次握手，2RTT才能够进行后续的通信加密，而且RSA密钥协商算法不具备前向安全性，而使用ECDHE密钥协商算法，不仅能够保证前向安全性，还支持TLS False Start，在第三次握手完成之后，第四次握手之前就能够发送数据，即抢跑，1个RTT之后就能够进行加密通信了

值得注意的是，即使选择了ECDHE算法，不同的椭圆曲线之间也存在性能差异，应该尽量选择`x25519`椭圆曲线，因为这个曲线是目前最快的了。比如在 Nginx 上，可以使⽤` ssl_ecdh_curve` 指令配置想使用的椭圆曲线，把优先使用的放在前面

> 对于对称加密密钥方面，如果系统对于安全性的要求不是很高的话，我们可以使用AES_128_GCM,，而不是使用AES_256_GCM，因为密钥短的计算更快。比如在 Nginx 上，可以使用` ssl_ciphers` 指令配置想使用的非对称加密算法和对称加密算法，也就是密 钥套件，⽽且把性能最快最安全的算法放在最前面



#### TLS协议优化

<img src="../../image/ComputerNetwork/image-20211114171559585.png" alt="image-20211114171559585" style="zoom:67%;" />

可以看到，TLS需要经过4次握手，2RTT才能够进行加密通信，先要通过 Client Hello 和 Server Hello 两次握手交换双方产生的随机数以及协商出后续使用的加密算法，再互相交换公钥（客户端产生的第三个随机数），最后计算出通信需要的会话密钥

而TLS1.3只需要进行2次握手，1RTT便能够进行加密通信了，具体做法是：客户端在`client Hello`记录中带上了支持的椭圆曲线以及客户端对应的公钥，服务器会选定一个椭圆曲线作为参数，同时再返回消息时带上自己的证书信息，以及服务器这边的公钥，因此，在两次握手之后，客户端和服务器都分别拥有了可以合成会话密钥的材料，于是计算出密钥之后便可以开始进行加密通信了



除此之外，TLS1.3还对支持的密码套件进行了删减，废除了不支持前向安全性的RSA和DH算法，只支持ECDHE算法，对于对称加密和签名算法，只支持目前最安全的几种，如下：

- TLS_AES_256_GCM_SHA384 
- TLS_CHACHA20_POLY1305_SHA256 
- TLS_AES_128_GCM_SHA256 
- TLS_AES_128_CCM_8_SHA256 
- TLS_AES_128_CCM_SHA256

TLS之所以支持这么少的密码套件是因为TLS1.2由于支持各种古老且不安全的密码套件，中间人可以利用降级攻击，伪造客户端的 Client Hello 消息，替换客户端支持的密码套件为一些不安全的密码套 件，使得服务器被迫使用这个不安全的密码套件进行HTTPS连接，从而破解密文



### 证书优化

为了验证服务器的身份，客户端和服务器在进行TLS握手过程中（第二次握手的过程中），服务器会把自己的数字证书发送给客户端，然后客户端通过这个数字证书来验证服务器的身份，对于证书的优化可以分成两个部分：

- 一个是证书传输过程中的优化
- 另一个是证书验证过程的优化

首先，想要优化证书在传输过程中的消耗，那肯定是减少证书的大小，这样可以节约带宽，减少客户端的计算量。所以，对于服务器证书应该选择椭圆曲线（ECDHE）证书，因为在相同安全强度的情况下，ECDHE的密钥长度要比RSA短



> 另一个方面是优化证书验证的过程，那么我们来看一下证书验证的过程

事实上，证书的验证是一个复杂的过程，客户端会走证书链来逐级验证，**验证的过程不仅需要使用公钥来解密证书、用签名算法验证数字证书的完整性，而且为了知道证书是否被CA吊销，客户端有时还要去访问CA，下载ORL或者OCSP，以确认证书是否还在有效期内**

- **CRL称为证书吊销列表（Certificate Revocation List）**，这个列表是由CA定期维护的，里面存放的是已经过期的证书，如果一个证书存在于这个列表里面，表示这个证书已经不被CA信任了

  可以看出，CRL有两个问题：

  - 第一个问题：由于CRL是由CA定期维护的，所以可能存在一个场景：某个服务器的证书刚被吊销之后，CRL中还没有更新出来，因此用户去遍历这个CRL列表的时候，还是会认为这个服务器是可信的
  - 第二个问题：随着吊销证书的增多，CRL列表会越来越多，文件会越来越大，而且下载下来之后，客户端还需要进行遍历，如果列表过大，那么可能会造成证书验证过程的时延过高，拖慢HTTPS的连接

- **OCSP称为在线证书状态协议（Online Certificate Status Protocol）**，它的工作方式是向CA发送查询请求，然后CA返回证书的状态，很明显，使用这种方式不用想CRL那样，需要先下载下来然后再去遍历，这是他的优点，但是他的缺点也很明显，因为进行证书验证的时候要向CA发请求，所以这就需要看CA的”脸色“了，也就是CA服务器的网络状态，如果网络状态不好或者CA服务器繁忙，那么将会有很大的延时

- **OCSP Stapling，**为了解决上面的网络开销，OCSP Stapling出现了，这种方式是服务器定期向CA查询证书的状态，获得一个带有时间戳的响应结果并缓存到服务器上，当客户端进行连接请求的时候，服务器会把这个响应结果一起发送给客户端，这样客户端解密之后直接看这个响应结果是否过期就好了，而且由于签名的存在，服务器是没有办法更改这个响应结果的，这样客户端也就无需在向CA请求了



### 会话复用

TLS握手的目的就是协商出后面通信使用的对称密钥，所以如果我们把客户端和服务器首次建立起连接的会话密钥缓存起来，下次建立HTTPS连接之后，直接复用这个会话密钥，不就减少了TLS握手的损耗了吗！**（很明显，这种方式不具备前向安全性）**

上述的方式就叫做会话复用，一般有两种方式：

- **Session ID**，其工作原理是客户端和服务器首次建立起TLS连接之后，双方都会在本地缓存会话密钥，并使用唯一的session id来标记，session id和缓存的会话密钥是key-value的关系，等到下次再连接的时候，客户端会在请求中（client-hello）带上这个session id，服务器收到之后就会通过这个session id在自己的本地缓存中找，如果找到了，那么就会跳过其余的握手过程，这样只使用一个RTT便能够建立连接，当然，**出于安全性的考虑，会话密钥缓存需要设置过期时间**

  从上面的过程中，也能够看出问题，如下

  - 服务器会和很多的客户端进行连接，所以随着连接过的客户端增多，服务器的缓存中会有越来越多的会话密钥，这样导致服务器的内存压力越来越大
  - 第二个问题是，现在很多大型网站都是用负载均衡，也就是不只是有一个服务器，因此客户端在之后连接的时候不一定连接上上一次的服务器，因此这时候还是需要走完成的握手流程

- **Session Ticket**，为了解决上面的两个问题，出现了Session Ticket，首先，服务器不再缓存这个Session Ticket了，而且交给客户端缓存，类似于Cookie。当客户端和服务器首次建立TLS连接之后，服务器会把会话密钥作为Ticket发送给客户端，交给客户端缓存，之后客户端和服务器进行连接时，客户端会把这个Ticket发在请求中发送给服务器，服务器解密之后验证这个Ticket的有效期，如果没有过期，那么将会跳过后面的TLS握手过程，直接就能够恢复加密通信了。对于集群服务器，需要保证每台服务器的Session Ticket是一样的，这样客户端携带Ticket访问任意一台服务器就都能够恢复会话了

- **pre-shared key**，无论是Session id还是Session Ticket，都需要1RTT才能够恢复通信，但是在TLS1.3中出现的pre-shared需要0RTT就能够恢复会话，实际上，它的原理和Session Ticket类似，只是在客户端向服务器发送握手消息时，除了带上Session Ticket还会携带上请求，如下图：

  <img src="../../image/ComputerNetwork/image-20211114180957433.png" alt="image-20211114180957433" style="zoom:50%;" />

> 需要注意的是，Session id、Session Ticket、pre-shared key都**不具备前向安全性**，因为一旦加密会话密钥被破解或者服务器泄露会话密钥，前面劫持的通信密文都会被破解同时，Session id、Session Ticket、pre-shared key应对**重放攻击**也很困难

**<font color=green size=4pt>什么是重放攻击呢？</font>**

<img src="../../image/ComputerNetwork/image-20211114180301043.png" alt="image-20211114180301043" style="zoom:50%;" />

假设A想向B证明自己的身份，B要求A的密码作为身份证明，因此A就会像B提供自己的密码，如果这个密码被攻击者C给截获到，那么E也能够假冒A对B进行访问，所以如果这个过程中Session ID或者Session Ticket和post报文被截获到，那么E就能够通过A的Session ID或者Session Ticket向B发送POST报文，而POST报文通常是会引起数据的变化，因此重放攻击可能会导致用户在不知情的情况下，数据库被攻击者破坏

**避免重放攻击的一个方式就是给会话密钥设置一个过期时间，而且可以只针对安全的HTTP请求使用会话复用，比如get**

