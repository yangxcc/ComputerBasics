# ComputerBasics
计算机网络，操作系统，未来会加入对Linux和MySQL的学习

**<font color="red">写前在最前面：计算机网络和操作系统的知识绝大多数来自于“小林coding”这个微信公众号，CSDN同名</font>**

未来的1~2个月我会每天定时复习一下计算机网络和操作系统的知识，并在此文件上做好记录，加油！

**仓库目录**

```
|--- image
       |--- OperationSystem
       |--- ComputerNetwork
|--- file
       |--- OperationSystem
                  |--- day01
                  |--- day02
                  |--- day03
                  ...
       |--- ComputerNetwork
                  |--- day01
                  |--- day02
                  |--- day03
                  ...
|--- project
        |--- 可能会有个小项目，比如Socket网络编程
```


**<font size="4pt">每日记录</font>**

- 2021年11月9日   创建此仓库，对仓库结构进行创建
- 2021年11月10日，复习了计算机网络中的TCP/IP模型，HTTP中的五类常见面试题，具体内容如下
  - TCP/IP网络模型中各层的大致作用
  - HTTP是什么？`超文本  传输  协议`
  - HTTP的特性是什么?`无状态  明文传输`
  - HTTP和HTTPS的区别已经HTTP中存在哪些安全问题，HTTPS是怎么解决这些安全问题的
  - HTTP/1.1的性能   `长连接   管道网络传输   --->  队头阻塞`
  - HTTP的演变过程，每一次版本的迭代都对前者进行了哪些优化，解决了哪些问题
- 2021年11月11日，复习了操作系统中硬件结构中的一小部分，具体内容如下

  - 冯诺依曼结构，五个部分的概述
  - CPU位宽和线路位宽，硬件的位数表示的是CPU的位宽，软件的位数表示的是指令的位数，线路位宽表示地址总线和数据总线的个数
  - 指令执行的基本过程，CPU+控制单元+程序计数器+地址总线+数据总线 --> 指令寄存器，控制单元分析指令类型 --> CPU控制单元/逻辑算术单元运算 --> 写回，从fetch -> Decode -> Excutor -> save/write这个过程叫做CPU的指令周期
  - 指令，程序 --> 汇编 --> 机器码，MIPS指令集，32位，不同类型的指令（R,I,J型）机器码格式不同
  - 指令的类型，数据传输指令，运算指令，跳转指令，信号类型指令（比如中断），闲置类型指令（nop）
  - 指令的执行速度，时钟频率（CPU主频），时钟周期（计算机世界中最小的时间单位），提高指令的执行速度也就是减少CPU的执行时间
  - 32位一定比64位好吗，64位的优势？能够一次性算更大的数，有更大的寻址空间


- 2021年11月12日，复习了网络中的HTTP优化和HTTPS的连接过程

  - HTTP的优化从三个方面：分别是避免发送HTTP请求（缓存），减少发送HTTP请求（合并，减少重定向，延迟发送请求），压缩数据（有损压缩和无损压缩）

  - HTTPS的连接过程，比HTTP多了一个SSL/TLS连接过程

    - ClientHello：发送客户端产生的随机数，TLS版本号以及支持的密码套件列表

    - ServerHello：确认版本号，选择密码套件，产生另一个随机数发给客户端，之后发送给客户端自己的数字证书，全都发过去之后ServerDone

    - change Ciper Spec：先校验服务器的数字证书，通过之后再向服务器发送一个随机数`pre-master`，之后对之前发送过的信息做一个摘要，叫做`Encrypted Handshake Message`

      **至此，客户端和服务端分享了三个随机数，通过这三个随机数生成会话密钥（Master Secret）**

    - change Ciper Spec + Encrypted Handshake Message：和客户端做一样的工作，如果都没问题，后续就使用生成的会话密钥进行加密通信

