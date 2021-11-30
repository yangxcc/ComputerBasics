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


:date:**<font size="4pt">每日记录</font>**

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

  - HTTPS的连接过程，比HTTP多了一个SSL/TLS连接过程（以RSA密钥协商算法为例）

    - ClientHello：发送客户端产生的随机数，TLS版本号以及支持的密码套件列表

    - ServerHello：确认版本号，选择密码套件，产生另一个随机数发给客户端，之后发送给客户端自己的数字证书，全都发过去之后ServerDone

    - change Ciper Spec：先校验服务器的数字证书，通过之后再向服务器发送一个随机数`pre-master`，之后对之前发送过的信息做一个摘要，叫做`Encrypted Handshake Message`

      **至此，客户端和服务端分享了三个随机数，通过这三个随机数生成会话密钥（Master Secret）**

    - change Ciper Spec + Encrypted Handshake Message：和客户端做一样的工作，如果都没问题，后续就使用生成的会话密钥进行加密通信

- 2021年11月13日，补上了前一天的网络知识（写到了day03.md），不同的密钥协商算法TLS握手过程的区别，下午复习了操作系统中的存储器，CPU对内存数据的读写以及怎么样写出让CPU运算更快的代码

  ```
  计算机网络
  RSA是比较简单的密钥协商算法，但是他并不具备前向安全性，因此又提出了DH算法
  DH算法分为 static DH（已废弃）和DHE算法（目前常用）
  而目前另一个比较常用的是ECDHE算法，这个密钥协商算法比RSA在第二次握手的时候多了一个server Key exchange记录，而且他具备前向安全性，而且通过椭圆曲线解决了DHE算法计算较慢的问题，而且他还能够在TLS第四次握手之前发送application data，也就是TLS False Start
  
  
  操作系统
  CPU Cache的数据结构和读取过程，内存地址是由索引号 + tag(组标记) + 偏移量组成，而CPU Cache是由索引号 + tag + 有效位 + 实际数据组成
  如何写出让CPU跑到更快的代码（和提高缓存命中率问题一样），因为L1 Cache分成了数据缓存和指令缓存
  所以要从两个方面：一是提高数据缓存命中率，而是提高指令缓存命中率
  除此之外，还需要考虑多核CPU的缓存命中率的提高
  ```


- 2021年11月14日，复习了计算机网络中对HTTPS的优化以及操作系统中保证内存和CPU Cache数据一致性 以及 不同CPU核心的Cache一致性 的方法

  - 首先是计算机网络对于HTTPS的优化，相比于HTTP，HTTPS的性能开销主要在两个方面：TLS握手过程和加密通信，优化方法共有5个方面
    - 硬件优化，HTTPS是计算密集型，所以要选择好的CPU，同时尽量选择支持`AES-NI`的CPU
    - 软件优化，将软件升级，可能提供一些更好的功能，修复bug等
    - 协议优化：密钥协商算法尽量选择ECDHE算法；将TLS1.2升级至1.3（2RTT--->1RTT）
    - 证书优化：传输过程中优化（选择ECDHE，相同安全强度下密钥短，所以证书小）；验证过程优化（CRL --> OCSP --> OCSP Stapling）
    - 会话复用：session id -->  session ticket --> pre-shared key（TLS1.3中才有）
  - 然后是操作系统，今天主要复习了以下内容
    - CPU Cache和内存中数据的一致性，保证一致性的两种方法：写直达（write through），写回（write back）
    - CPU不同核心之间的缓存一致性，需要实现两点：写传播和事务串行化，写传播可以通过总线嗅探来完成，但是总线嗅探不能够实现事务串行化而且开销较大，所以有了一种基于总线嗅探的MESI协议来实现上述两点，Modified，Exclusive，Shared，Invalidate，通过状态机之间的转化能够减轻总线压力

  

- 2021年11月15日，复习了网络中的HTTP/2，它的厉害之处总共有五个部分：
  - 首先是做到了对HTTP/1.1的向上兼容，没有改变语义层，在浏览器或者客户端背后悄悄升级
  - 第二是使用了HPACK头部压缩算法，减轻了带宽压力，减少了冗余的头部传输
  - 第三是使用二进制帧，将HTTP/1.1中的文本传输格式变成了二进制帧传输格式，更有利于计算机的理解
  - 第四是使用stream完成并发传输，多个stream中的frame可以乱序传输，但是同一个stream中的frame必须是有序的
  - 第五是服务器能够主动推送资源，减少了HTTP请求的数量

  > :x: 偷懒的一天

- 2021年11月16日，复习了操作系统中的伪共享和CPU是如何选择线程的；网络中的HTTP/3

  - :blue_book:操作系统
    - 伪共享：多个核心中的线程同时操作同一个block中的不同变量，会导致cache失效，频繁地与内存交互，解决方法由字节对齐和字节填充
    - CPU是如何选择线程的？内核中有3个调度类，对应了3个调度器和5种调度策略，分别是`Deadline,Runtime,CFS`，前两个是针对实时任务（优先级0-99），CFS是针对普通任务（优先级100-139），内核中的线程都被放到了一个运行队列中，即`rq`，这个`rq`分成了三种，分别是`dl_rq,rt_rq,cfs_rq`，优先级依次降低，也就是说CPU会首先调度dl_rq中的线程，依次类推，`cfs_rq`的数据结构是红黑树，对于普通任务来说，也是可以通过nice调整优先级的，（nice是优先级的修正参数）
  - :closed_book:网络
    - 今天主要复习了HTTP/3，它通过将底层的TCP更换成UDP（QUIC协议）解决了HTTP/2中存在的问题：队头阻塞（不会阻塞全部请求，只会阻塞单个的stream），TCP和TLS握手导致的延迟（UDP无连接，TLS升级成了1.3，只需要1个RTT，后续连接还可以使用pre-shared key），网络迁移需要重新连接（用连接id）这三个问题，同时还在应用层对HTTP/2中的问题进行优化，比如将HPACK更换成了QPACK，QPACK使用两个单向流解决了HPACK中存在的队头阻塞问题；简化了HTTP/2中二进制帧格式（只有帧类型和数据长度了）
  
- 2021年11月17日，复习了操作系统中的软中断；网络中的TCP常见问题，三次握手以及SYN攻击


  - :blue_book:操作系统


    - 中断是操作系统用来响应硬件设备请求的一种机制，操作系统收到中断请求之后会立刻停止CPU目前正在进行的工作，去处理中断，而且处理中断请求的过程中，可能会关闭其他的中断请求，也就是说这个过程中如果有其他的中断请求到来，可能会丢失；因此为了减少对正常进程的影响和减少中断丢失，中断处理程序要短且尽可能地快
    
    - 软中断是中断的下半部分，为了解决上面的问题操作系统把中断分成了上下两个部分，第一部分是硬中断，直接处理硬件请求，耗时短；第二部分是软中断，将一些耗时的操作交给软中断来处理，软中断是内核线程，每个CPU核心都有一个
    
      比如网卡接收网络包，硬中断将网卡驱动中的网络包读到内存中，软中断根据网络协议栈将包打开，直至发给应用程序

  - :closed_book:网络


    - TCP中的常见问题，比如TCP的头部格式，为什么需要TCP协议（IP层不可靠），什么是TCP协议（面向连接的、可靠的、基于字节流传输的），什么是TCP连接（序列号，socket，窗口大小），如何确定一个唯一的TCP连接（四元组），**TCP和UDP的区别，分别的应用场景（重要！）**，
    - TCP三次握手（SYN(Seq=Client_isn),SYN+ACK(Seq=Server_isn,ack=Client_isn+1),ACK(Server_isn+1)）
    - SYN攻击（占满SYN队列，解决方式假设防火墙，Cookie，修改内核参数，半连接队列满了之后就给新的连接发送RST）



- 2021年11月18日，复习了操作系统的负数补码和内核等知识

  - 为什么负数要用补码来表示（因为如果不用补码的话，还需要加一层判断，如果是负数的话，需要把加法转成减法，减法转成加法）

  - 十进制小数怎么转化成2进制（乘2取整）

  - 计算机中是怎么存储二进制小数（浮点数，类似于科学计数法，符号位 + 指数位 + 尾数位）

    - 符号位0表示正数，1表示负数
    - 指数位表示小数点在尾数中的位置，指数位越大，能够表示的范围就越大
    - 尾数位表示精度，尾数位越长，精度越高

  - 为什么0.1+0.2不等于0.3，因为十进制的0.1和0.2在计算机中转化成2进制之后，是无限循环小数，计算机只能够存储一个近似值，这也就导致了浮点数的精度损失

  - Linux内核 V.S. Windows内核

    - 内核的功能：进程管理、内存管理、硬件通信管理、提供系统调用
    - Linux内核：宏内核，可执行文件格式ELF，MultiTask，SMP（对称任务处理，CPU地位一样）
    - Windows内核：混合内核（微内核 + 宏内核），可执行文件格式PE，MultiTask，SMP（对称任务处理，CPU地位一样）

    宏内核的意思就是这个内核中包含了全部的工作，比如进程管理、文件系统、内存管理等，他是一个可执行程序，代表作Linux

    微内核就是只保留一个最基本的可执行单元，其他的放到用户空间，代表作华为的鸿蒙OS

    混合内核就是一个微内核外面套着一个宏内核，代表作Windows
  
- 2021年11月19日，复习了计算机网络中的TCP四次挥手

  - TCP四次挥手的过程（FIN,ACK,FIN,ACK）

  - 为什么需要四次挥手（被动断开方可能有数据没有处理完）

  - 为什么要有TIME_WAIT状态，TIME_WAIT的时间为什么是2MSL

    - 避免历史的连接的数据包被新连接收到，造成数据混乱
    - 保证连接正确关闭（有可能被动断开方没有收到ACK，重发FIN）

  - TIME_WAIT时间过长有哪些危害

    - 如果客户端是主动断开方：有可能导致客户端端口不够用了，全都被处于TIME_WAIT状态的连接占用，没办法建立新连接
    - 如果服务器是主动断开方：有可能导致服务器线程池资源耗尽，无法处理新的请求，因为即使处于TIME_WAIT状态的连接，仍然需要线程来维持他的状态

  - 如果优化TIME_WAIT

    - `tcp_tw_reuse + timestamp`重用time_wait状态的连接，使用时间戳来区分是不是历史数据包
    - `tcp_max_tw_buckets`这个参数表示最多可以处于TIME_WAIT状态，一旦超过了这个数就会重置连接
    - 设置socket选项，通过调用close可以跳过四次挥手，很危险

  - 如果连接过程中有一个掉线了，如果在线的一方开启了保活机制，那么会向掉线的一方发送探测报文  （2小时  75秒  9次）

    保活机制比较有争议！两个缺点：有可能由于网络短暂出现错误，导致好的连接断开；增加网络带宽负担，有人认为他应该在应用层实现，有人认为他就应该在TCP协议层实现

- 2021年11月20日，复习了网络中的Socket编程和操作系统中的内存管理机制

  - :blue_book:操作系统

    直接操作物理地址没有办法做到多个进程同时执行，所以就引入了虚拟地址

    - 分段机制，把程序分成四个段：代码段、数据段、堆段、栈段，虚拟地址被分成了两个部分：段选择因子+段内偏移量，其中段选择因子中包括段号，特殊权限标记等，短号是最重要的，每个段的大小不固定
    - 分页机制，将虚拟地址和物理地址都化成固定大小的页，Linux中每页为4KB，虚拟地址分成了两个部分：页号 + 页内偏移量，每个进程都对应一个页表，为了解决空间问题，采取多级页表，为了解决多级页表中存在的时间问题（多了地址转换的过程），采用了TLB快表
    - 段页式管理，每个程序对应一个段表，段表中的每一项对应一个页表，段表中存放的是页表的起始地址，页表中存放的是物理地址

    Linux内存管理主要采用的是分页机制，但是由于Intel CPU发展的缘故，绕不开分段的思想，因此他将每段的地址都设置成了从0到4G（32位），这样每个程序都相当于面对的是线性地址

  - :closed_book:网络

    socket编程中的方法使用阶段

    在三次握手的时候，connect的成功返回是在第二次握手结束之后（服务器返回给客户端ACK+SYN），accept的成功返回是在三次握手结束之后，连接成功之后

    在四次挥手的时候，主动断开方（假设是客户端）调用close，在第三次挥手**的时候**，服务器发送FIN报文，调用close，关闭连接，表示没有数据要发送了

- 2021年11月21日，复习了操作系统中的进程和线程的基本概念以及模型；网络中的TCP重传机制和滑动窗口

  - :blue_book:操作系统

    - 进程，程序的一次动态执行

      - 进程的状态：创建 就绪 运行 阻塞 终止 挂起
      - 进程的数据结构PCB：进程标识符，用户标识符，资源清单，保存的CPU的上下文信息，优先级，当前状态等信息
      - 进程控制：创建、终止、挂起、唤醒等状态的转化过程
      - 进程的上下文切换：CPU上下文切换（保存前一次执行的信息，CPU指令寄存器和程序计数器，加载本次的信息），进程上下文的切换是由内核管理，所以既需要用户空间的资源切换，比如虚拟空间、栈、全局变量等资源，还需要内核空间的资源切换，比如内核堆栈、寄存器等资源

    - 线程，进程的一条执行流程，共享进程的虚拟地址、全局变量等资源，独享自己的寄存器、栈等资源

      - 进程和线程的比较

      - 线程相较于进程开销较小，体现在创建、终止快，同一进程内的线程切换快，通信不需要经过内核

      - 线程上下文切换：两种情况，在同一个进程和不在同一个进程内

      - 线程的实现

        - 用户线程，由用户空间的库函数管理和调度，操作系统看不见，只能够看见他们所在的进程，类似于多对一模式
        - 内核线程，由操作系统内核管理和调度，类似于一对一模型
        - 轻量级线程LWP，内核空间支持的用户线程，由操作系统管理，和用户线程的对应模型为`1:1,1:N,M:N`

        以及上述三种模型各自的优缺点

  - :closed_book:网络

    - 重传机制

      - 超时重传，时间周期长，以时间驱动
      - 快速重传，以数据为驱动，连续收到三次相同的ACK触发重传机制
      - SACK，能够知道缺少了哪个报文
      - D-SACK，能够知道什么原因导致的报文丢失

    - 滑动窗口

      - 发送窗口，四部分，三个变量来划分`SND_WND,SND_UNA,SND_NXT`
      - 接收窗口，三部分，两个变量来划分`RCV_WND,RCV_NXT`

      滑动窗口的大小由接收窗口决定，接收窗口大小和发送窗口大小的关系是约等于，因为存在网络延迟



- 2021年11月22日，复习了网络中的TCP流量控制和阻塞控制

  - TCP流量控制
    - 经典的流量控制模型
    - 内存缓冲区对发送窗口和接收窗口的影响，不能够先减少缓冲区大小，在收缩窗口大小，这样会导致丢包和发送窗口变为负数的现象
    - 窗口关闭，可用窗口大小变为0，存在的潜在风险就是由于接收方通知发送方有可用窗口的ACK报文丢失，造成死锁，解决方法就是通过一个计时器，探测，一般最多探测三次
    - 糊涂窗口综合症，由于应用程序繁忙，导致接收方内存缓冲区的数据处理不及时，滞留在缓冲区中，导致窗口越来越小，然后发送窗口能够发送的数据包也越来越小，tcp头部+ip头部就40个字节了，发送小的数据包很不合算
      - 产生原因发送方能够发送小的数据，接收方不管可用窗口是多少都会通知发送方
      - 解决方法就是处理上面的两个原因：接收方当可用窗口大小 >= MSS或者可用窗口大小>=缓存空间的一半的时候才能够通知发送方，发送方收到ACK响应或者累计的数据包>=MSS,后面这个是Nagle算法来完成，TCP中默认打开，对于一些交互性强的需要把Nagle关闭，通过socket参数NO_DELAY
  - TCP阻塞控制
    - 流量控制是避免接收方的缓冲区被填满，阻塞控制是避免网络被填满
    - 发送窗口swnd=min{cwnd, rwnd}
    - 慢启动，指数增长，每收到一个ACK，cwnd += 1，阈值为`ssthresh`
    - 阻塞避免，由慢启动的指数增长变为线性增长，每收到一个ACK，cwnd += 1/cwnd
    - 阻塞发生，超时重传，cwnd直接变成1，一夜回到解放前，从慢启动开始，会导致网络突然卡顿
    - 快速恢复，快速重传，拥塞窗口先变成原来的一半cwnd=cwnd/2，ssthresh=cwnd,执行快速恢复算法
      - 如果收到了重复的ACK，cwnd+1
      - 如果收到了新的ACK，cwnd=ssthresh + 3

  

  > :x: 偷懒的一天 

- 2021年11月23日，复习了操作系统中的进程通信的6种方式

  - 管道，单向字节流无格式传输，有大小限制，匿名管道只能够在有亲缘关系的进程之间使用，命名管道可以解决只能在这种亲缘关系管道之间使用的弊端，通过`mkfifo`创建。需要注意的是命名管道会被存储在文件系统中，而匿名管道不会，它的生命周期随着进程的消失而消失，管道其实就是内核中的一串缓存。管道的缺点就是通信效率太低了，不适合通信频繁的场景，优点就是直到何时写入管道，何时读取

  - 消息队列，这种方式类似于发邮件，解决了管道通信效率低的问题，但是有两个缺点：通信处理不及时，消息体和消息队列长度有限制，分别通过内核参数`MSGMAX`和`MSGMNB`决定，同时它最大的缺点就是消息队列存储在内核中，进程通信需要频繁地从用户态和内核态之间拷贝数据

  - 共享内存，这种方式解决了消息队列中频繁从用户态和内核态之间拷贝数据的缺陷，具体做法是因为进程都有自己独立的虚拟地址空间，在每个进程的虚拟地址空间中找出一块映射到相同的物理内存上，这样就不用用户态和内核态切换了

  - 信号量，在多线程系统中，共享内存可能会造成数据混乱，所以需要搭配信号量来完成进程之间的通信，信号量通常表示的是共享资源的数量，PV原语，互斥信号量1，同步信号量0

  - 信号，和信号量没关系，它是用来处理异常情况下的进程通信，来源有硬件来源（ctrl+c,ctrl+z等），软件来源（kill命令）

  - socket，主要有三种模式：基于TCP的网络编程模型，基于UDP的网络编程模型，本地进程通信模型（和前两个最大的区别在于他需要绑定一个本地文件，而前两个绑定IP地址和端口号）

    ```c
    // 系统调用
    int socket(int domain, int type, int protocal)
    // domain   AF_INET代表ipv4，AF_INET6代表ipv6，AF_LOCAL/AF_UNIX代表本地
    // type     SOCK_STREAM表示字节流传输，SOCK_DGRAM表示数据报传输
    // prorocal 基本通过前两个参数就能够确定是什么协议了，因为TCP是字节流传输，UDP是数据报传输    
    ```

  此外，对于线程应该关注的重点是线程间对共享资源的互斥和同步访问，而不是线程的通信，因为线程可以共享进程的资源，比如全局变量、内存地址等
  
- 2021年11月24日，补齐了前一天的TCP三次握手中的异常情况、TCP快速连接、TCP延迟确认和Nagle算法

  - TCP三次握手中的异常情况
    - 第一次握手的SYN包丢失：发送方会触发超时重传，重传时间指数式增长，重传次数通过内核参数`tcp_syn_retires`确定
    - 第二次握手的SYN+ACK包丢失：发送方会触发超时重传，重传SYN包，同时接收方也会超时重传，重传SYN+ACK包，重传次数由内核参数`tcp_synack_retires`确定
    - 第三次握手的ACK包丢失，这时候发送方已经处于`ESTABLISHED`状态，所以重传ACK的次数由`tcp_retires2`确定
  - TCP快速连接，第一次连接仍然为2RTT，但是服务器会产生一个加密的cookie，客户端缓存到本地，之后连接时客户端的SYN包中会携带这个cookie以及请求数据，这时只需要1RTT，直到cookie过期
  - TCP延迟确认，在接收方开启，通过socket参数`TCP_QUICKACK`设置，Nagle算法在发送方开启，通过socket参数`TCP_NODELAY`确定




- 2021年11月25日，复习了操作系统中的多线程互斥和同步

  - 首先是概念：互斥指的是多个线程不能够同时访问一个共享资源，同步指的是多个线程必须协作按照某种顺序共同完成任务
  - 然后指出了互斥和同步实现的方式有两种：锁和信号量，信号量的功能要比锁强，因为信号量在实现同步问题上更加方便，两个都很方便实现互斥
  - 然后讲解了锁的类型：最基本的是两个自旋锁和互斥锁，这两个锁的实现原理相似，只是当加锁失败后的处理方式不同，自旋锁在加锁失败之后会一直忙等待，不会让出CPU，而互斥锁在加锁失败之后会让出CPU，切换线程；之后又学习了读写锁，分为三种：读优先锁，写优先锁和读写公平锁，最后学习了悲观锁和乐观锁，前面提到的所有锁都是悲观锁，乐观锁其实就是不加锁，认为多线程共同操作共享内存发生冲突的几率很小，比如在线文档，git，svn等
  - 信号量主要就是PV原语两个操作，P操作就是将信号量减1，如果结果小于0，说明目前有线程在持有锁，本线程会阻塞；V操作就是将信号量加1，如果结果小于等于0，说明目前有线程处于阻塞状态，会唤醒一个处于阻塞队列的队头元素，将其TCB加入到就绪队列中
  - 接着学习了三个经典的同步问题：生产者-消费者问题，哲学家就餐问题，读者-写者问题，以及他们各自相应的实现
  - 最后学习了死锁，必须满足四种条件：互斥条件、持有并等待、不可剥夺条件、环路等待条件，避免死锁只需要破坏其中一个条件即可，比如顺序分配资源、使用定时锁、死锁检测等




> :x:偷懒了三天，赶紧看呀，整理心情，给👴支楞起来，加油呀！



- 2021年11月28日，复习了操作系统中的调度算法
  - 进程调度算法
    - 先来先服务进程调度算法 FCFS
    - 短作业优先调度算法SJF
    - 最高响应比进程调度算法HRRN
    - 时间片轮转调度算法RR
    - 最高优先级调度算法HPR
    - 多级队列反馈调度算法MQ.
  - （物理）内存页面置换算法
    - 最佳页面置换算法OPT，未来最不常使用，理想算法，通过用来做比较标准
    - 先进先出页面置换算法
    - 最近最久未使用页面置换算法，双线链表
    - 时钟页面置换算法，环形链表
    - 最不常用页面置换算法，计数器最小的页面换出，不好，没有考虑时间因素
  - 磁盘调度算法
    - 先来先服务算法FCFS
    - 最短寻道时间优先，有可能会出现饥饿现象
    - 扫描算法 SCAN，不会出现饥饿现象，效率较高，寻道时间短，缺点是每个页的访问频率不同
    - 循环扫描算法 C-SCAN，优点是每个页的访问频率基本相同，缺点是和SCAN相比寻道时间增多了，而且返回过程不处理请求
    - LOOK，优化SCAN，不用走到最始端或者最末端在转换方向，走到请求的最远端就可以转化方向
    - C-LOOK，优化C-SCAN，优化同SCAN



- 2021年11月29日，复习了网络中的TCP的全连接队列和半连接队列以及TCP中的相关优化

  - 半连接队列：又叫SYN队列，SYN攻击会使其变满，可以通过`tcp_syncookie`开启cookie来绕过SYN队列，增大半连接队列的长度不仅需要增大`tcp_max_syn_backlog`，还需要增大全连接队列的长度；此外，减缓SYN攻击还可以通过减少SYN+ACK包的重发次数

  - 全连接队列：又叫accept队列，当应用程序处理较慢，并发量大时可能会造成全连接队列变满，全连接队列的长度等于`min{backlog, somaxconn}`，可以通过增加这两个参数来增加全连接队列的长度

  - TCP三次握手的优化

    - 全连接队列和半连接队列长度的调整
      - 可以通过`ss -lnt`直接查看全连接队列的长度，在listen和非listen状态下的Send-Q和Recv-Q的意义是不同的
      - 半连接队列满了，可以通过命令`netstat -s |grep "SYNs to LISTEN"`查看被丢弃的请求
      - 全连接队列满了之后，不但ACK会被丢弃，SYN也会被丢弃，可以通过命令`netstat -s |grep overflowed`查看
    - SYN重发次数由`tcp_syn_retires`决定
    - SYN+ACK重发次数由`tcp_synack_retires`决定
    - 可以通过`tcp_fastopen`开启fast open，这样可以绕过三次握手

  - TCP四次挥手的优化

    - FIN报文的重传次数是由`tcp_orphan_retires`决定的
    - close关闭和shutdown关闭的区别，后者更加优雅，可以关闭一个方向上的读/写，通过close是直接关闭，称为孤儿连接
    - 调整孤儿连接的最大个数`tcp_max_orphans`
    - 调整`FIN_WAIT_2`状态的时间，只适用于使用close关闭的连接，`tcp_fin_timeout`
    - 调整time_wait状态连接的最大值，`tcp_max_tw_buckets`
    - 复用time_wait状态的连接`tcp_te_reuse + tcp-timestamp`

  - TCP传输性能的提升

    - 扩大窗口的大小`tcp_window_scaling`，TCP报头中的选项字段

    - 发送窗口的最大值要尽可能地接近带宽时延积，发送缓冲区的范围通过`tcp_wmem`来确定，发送缓冲区会自动调整

    - 接收缓冲区的范围通过`tcp_rmem`来指定，接收缓冲区需要打开`tcp_moderate_rcvbuf`才可以自动调整

      需要注意不要再socket中指定发送缓冲区和接收缓冲区的大小，因为这样会导致自动调整功能失效

    - TCP连接的内存范围`tcp_mem`，这个值越大，说明服务器分配给用于TCP连接的内存越多，能够支持的并发量越大



- 2021年11月30日，复习了操作系统中的文件系统相关内容

  - 文件系统的基本组成：索引节点（inode）+ 目录项（dentry），索引节点是文件的唯一标识，其包含内容有索引节点id，文件大小，文件创建、修改时间、文件类型等，目录项中包含了文件名、索引节点指针以及目录项之间的关联关系

    - 目录和目录项不同的是：目录是文件，存储在磁盘上，目录项是由内核维护的一个数据结构，存储在内存中， 内核会把已经用过的目录通过目录项这个结构缓存到内存中
    - 文件系统读写的基本单位叫数据块，一个数据块是8个扇区大小，也就是4k，一个页面大小也是4k，这就对应起来了，文件系统读写的基本单位是数据块

  - 虚拟文件系统，因为文件系统的种类众多，所以需要使用一个虚拟文件系统（VFS）来统一接口

  - 文件的使用，操作系统会为进程打开的所有文件维护一个文件打开表，文件打开表中的每一项都是一个文件描述符，文件描述符中的内容包括文件指针、文件计数器、文件磁盘位置和访问权限。

  - 文件的存储机制

    - 连续存储方式，优点是能够一次性查询出需要的数据块位置，缺点是会产生磁盘空间碎片，以及文件大小不易扩展

    - 非连续空间存储方式，分为两种，链式存储方式和索引存储方式

      - 链式存储方式又分成了两种：隐式存储和显式存储，都能够解决连续存储方式中的两个缺点

        - 隐式存储指的是在文件头中存放起始块和末尾块的位置，缺点是连接不稳定，如果因为发生硬件/软件错误而导致指针丢失或损坏，部分文件数据也会丢失；第二是每个数据块中都需要拿出一部分来存储下一个数据块的指针
        - 显式存储指的是将每个数据块的指针都放到一个表中，这个表叫做文件分配表FAT，表放在内存中，文件头中需要有指向这个表的指针

        很明显，不管是隐式存储还是显示存储，都不适合大文件

      - 索引存储方式，为每个文件创建一个或者多个索引数据块，索引数据块中存放的是每个数据块的指针，那么文件头中需要有指向这个索引数据块的指针，即使文件很少，也需要拿出一个数据块来存储其他文件数据块的指针，如果文件很大，那么就需要使用组合的方式来解决大文件的存储

        - 索引 + 链表的方式：在索引数据块上留出一个指向下一个索引数据块的空间
        - 索引 + 索引的方式：索引数据块中存放的是其他索引数据块的地址

  - UNIX文件系统，综合了上述文件组织/存储方式的优点，如果需要的数据块个数小于10，那么顺序存储，直接读取，如果大于了10，还有一级索引存储，二级索引存储和三级索引存储，所以UNIX文件系统中的文件头中需要有13个指针

  - 空闲空间的管理，空闲表法，空闲链表法，位图法（Linux中是使用位图法来管理空闲空间的）

  - 文件系统结构，这个结构由块组组成，一个块组128M，块组中包含超级块、块组描述符表、数据位图、索引节点位图、索引节点列表、数据块

  - 目录的存储，目录存储在磁盘中，很普通文件不同的是，普通文件存储的是文件数据，而目录文件存储的是目录项或者文件，如果一个目录下的文件和目录项有很多，那么会使用hash，这时候需要使用预备措施来防止哈希冲突

  - 软链接和硬链接

    - 硬链接：多个指针指向同一个文件，不能够跨文件系统
    - 软链接：相当于创建了一个新文件（inode），只是这个新文件的内容是指向另一个文件的路径，可以跨文件系统

  - 文件IO

    - 直接和非直接IO，指的是是否使用内核缓冲区，直接IO指的是应用程序通过文件系统直接访问磁盘，不会发生用户程序和内核缓存之间的数据拷贝

    - 缓冲和非缓冲IO，指的是是否使用标准库提供的缓冲区，他们两个都使用内核缓冲区

    - 阻塞IO和非阻塞IO，指的是应用程序是否会阻塞自身运行

      - 阻塞IO等待的是两个过程：一个是数据准备好，另一个是数据从内核缓冲区拷贝到应用程序缓冲区

      - 非阻塞IO有两种，第一种是比较笨重，虽然不会阻塞，这个过程线程不会去干别的事情，也不会发起CPU，而是不断地轮询查看数据是否准备好；第二种是多路复用非阻塞IO，这种是数据准备好了之后，内核会通过事件通知应用程序，然后应用程序只需要等待用户程序和内核缓存之间的数据拷贝过程即可，在没有被通知的情况下，CPU可以去做别的事情

        > 多路复用非阻塞IO又被称作NIO

    - 同步IO和异步IO

      前面所说的IO方式都是同步IO，因为他们都有一个或者两个等待过程

      而真正的异步IO是不会等待任何时间的，应用程序发起`aio_read`请求之后，内核会把数据准备好并完成从用户程序到内核缓存之间的数据拷贝过程，都完成了之后才回去通知应用程序

      > 异步IO又叫做AIO





# END

