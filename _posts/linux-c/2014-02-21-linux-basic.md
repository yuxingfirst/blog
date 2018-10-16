---
title: Linux编程基础
layout: post
categories: [linux-c-cpp]
tags: [linux-c]
description:  .
---  

###一、Linux进程内存布局  
![img1][linux-process-memory-layout]  

如上所示，是32位模式下linux内存布局。32位的程序寻址空间是 4G，即 0x00000000 ~ 0xffffffff， 从上往下以此为:  

1. 内核空间(Kernel)
2. 栈空间(Stack)
3. mmap(Memory Mapping Segment)
4. 堆空间(Heap)
5. 静态存储区(BSS segment)
6. 数据区(Data segment)
7. 文本段(Text segment)


栈空间：  
一般程序内的局部变量分配在栈空间，栈空间的地址是往下增长的。

堆空间：  
用malloc、new动态申请的内存分配在堆空间，堆空间的地址是往上增长的。

静态存储区：  
程序中未初始化的静态变量，会分配在这个区间，用0填充

数据区：  
如字符串常量，已初始化的静态变量，一般分配在这个区间

文本段：  
存储进程的二进制映像

	//
	int main(int argv, char *arg[])
	{
		static　int　ａ；	//静态存储区
		char *p = "msg";	//p 分配在栈区，字符串常量msg分配在
		int *pi = malloc(sizeof(int));	//malloc动态申请的内存分配在堆区
	}

###二、进程间通讯的方式
1. socket套接字，点对点，全双工通信。
2. pipe管道。参考：[linux进程通信之管道](http://coderworm.com//unix/2013/12/16/unix-process-communicate-a.html)  

###三、如何定位内存泄漏  
1. 检测工具： valgrind。
2. 打日志，记录一段内存的分配和释放记录。
3. gdb调试。  

###四、linux下动态链接(.so)和静态链接(.a)的区别
通常情况下，对函数库的链接是放在编译时完成的，所有相关的对象文件(.o)与程序中用到的函数库被链接合成一个可执行文件。这样，程序在运行时与函数库再也没有关联，因为所有需要用到的函数都已经拷贝到自己的区间了，类似这样在编译时链接的函数库称为静态链接(.a)。  

那么动态链接是什么呢？ 就是程序库在程序运行时才载入，也就是说，程序在运行中如果需要某个动态库时，操作系统会首先查看所有运行之中的程序是否已经加载过这个动态链接库了，如果已经加载过了，则这个程序直接共享这个已经加载到内存中的动态链接库。  
比如,有A、B、C三个程序，都需要用到libx.a, liby.so， 那么当A B C三个程序都运行时，对于libx.a，内存中会有三分拷贝；而对于liby.so，则只会存在一份拷贝。可见，动态链接库能够大大的节省系统资源。  

对于使用动态链接库，在程序升级时，也会相当方便，只需要升级动态库即可，自己的程序不需要重新编译；而如果使用的是静态库，则必须要重新编译程序了。  

类似于，标准c库 libc.so, libc++.so等这些，都是动态库。

所以，动态链接库的好处有：  

1. 可以实现进程之间的资源共享
2. 将一些程序升级变得简单。用户只需要升级动态链接库，而无需重新编译链接其他原有的代码就可以完成整个程序的升级
3. 甚至可以真正坐到链接载入完全由程序员在程序代码中控制  

###五、多线程和多进程的区别  

数据共享

1. 进程是资源分配的最小单位，每个进程有自己的地址空间，含有自己的寄存器，存储空间(代码段，数据段，堆栈)。而线程是cpu调度的基本单位，即程序执行的最小单位；线程运行在进程中，共享进程的地址空间(即存储空间)，不过，每个线程有自己的寄存器和栈空间。
2. 同一进程下的线程共享的部分包括:全局变量和静态变量，这样多个线程之间的通信就非常方便了，不过随之而来的数据同步和互斥也比较麻烦。而进程之间的通信只能通过诸如：socket, pipe(管道), fifo(命名管道)等等，通信方式没有线程那么方便，不过也减少了数据同步与互斥的问题。

![img2][process-and-thread]

上下文切换

线程的上下文切换和进程的上下文切换的最主要差别，在于线程的上下文切换后，虚拟内存空间仍然是有效的(因为线程共享进程的内存空间)；而进程的上下文切换后，虚拟地址空间就失效了。另外，进程在切换后，处理器的内存地址缓存也都失效了，开销很大。  

线程私有的数据

线程私有的东西有: 栈空间，寄存器。

###六、常见的信号
	
	SIGHUP     	终止进程     		终端线路挂断
	SIGINT      终止进程     		中断进程
	SIGQUIT   	建立CORE文件终止进程，并且生成core文件
	SIGSEGV   	建立CORE文件       	段非法错误
	SIGBUS     	建立CORE文件       	总线错误
	SIGKILL   	终止进程     		杀死进程
	SIGALARM   	终止进程     		计时器到时
	SIGCHLD   	忽略信号     		当子进程停止或退出时通知父进程

	SIGPIPE     对一个已经关闭写端的套接字继续调用write会产生这个信号

###七、i++是否原子操作
原子操作，如果编译成一条指令，那么必定是原子操作；如果编译为多条指令，如果加了锁，则是原子操作，没加锁的话，就不一定是原子操作了。  

###八、各类linux系统同步机制
1： 互斥锁 mutex
pthread_mutex_t mutex=PTHREAD_MUTEX_INITIALIZER


2： 信号量


###九、什么是死锁，如何避免死锁  

死锁发生的条件:   

1 互斥条件 
		
	即某个资源在一段时间内只能由一个进程占有，不能同时被两个或两个以上的进程占有。
	这种独占资源如CD-ROM，打印机等等，必须在占有该资源的进程主动释放后，其他的进程才可能
	占有该资源。

2 占有且等待  
	
	进程至少已经拥有了一个资源，但又申请新的资源；由于该新资源已被另外进程占有，此时该进程阻塞；
	但是在它等待其他资源的时候继续占有不释放已有的资源。

3 不可抢占  

	进程所获得的资源在未使用完毕之前，资源申请者不能强行地从资源占有者手中夺取资源，而只能由该资
	源的占有者进程自行释放

4 环路等待  
	
	存在一个进程等待序列{P1，P2，...，Pn}，其中P1等待P2所占有的某一资源，P2等待P3所占有的某一
	源，......，而Pn等待P1所占有的的某一资源，形成一个进程循环等待环

如何避免:

1. 银行家算法(没研究过)
2. 按找同一种序列申请资源，释放资源(安全序列)

###十、举例说明各类linux系统的异步机制
socket套接字，异步非阻塞，IO多路复用，select, epoll

###十一、exit和_exit()的区别
exit是标准库函数，位于stdlib.h中，而 \_exit是unix系统函数，位于unistd.h。  
调用exit函数时，exit会执行一些额外的操作，而后再调用到内核级别的exit函数。如：为所有打开流调用fclose函数( 会造成所有缓冲的输出数据都被冲洗)，释放进程占用的内存。  
调用\_exit则直接退出。

###十二、如何实现守护进程
fork两次。  

	//
	pid_t pid = fork();
	switch(pid) {
		case -1:
			return;
		case 0:
			break;
		default:
			_exit();
	}

	pid = fork();
	switch(pid) {
		case -1:
			return;
		case 0:
			break;
		default:
			_exit();
	}
	//父进程变为了init

###十三、系统如何将一个信号通知到进程

###十四、c语言
1. 哪些库函数是高危函数(strcpy, strncpy)
2. 宏定义和展开
3. 位操作
4. 指针操作和计算
5. 内存分配


###十五、c++ string的实现(复制构造函数， 赋值操作符operator=)

注意字符串结尾的字符串结束符，拷贝构造函数，赋值操作符，析构函数, 深浅拷贝。

	class mystring
	{
		private:
			int size;
			char *data;

		public:
			mystring()
			:size(0), data(new char[1])
			{}
			
			~mystring();

			mystring(const mystring &rstring);

			mystring& operator=(const mystring &);
			
			const char* c_str();
	}

###十六、虚函数的作用和实现原理
c++中的虚函数主要是实现了多态的机制，即通过父类型的指针或引用指向其子类型的实例，然后可以通过父类型的指针或引用调用子类的成员函数。  

主要是通过一张虚函数表以及指向虚函数表的指针来实现的。每一个包含虚函数的类的实例都包含一个指向虚函数表首地址的指针。我们可以根据这个指针找到虚函数表进而找到对应的虚函数并实现调用。为了提高效率，c++编译器重视保证虚函数表的指针位于对象实例最开始的位置，所以我们可以直接通过对象实例的地址来访问虚函数表。   

###十七、sizeof求一个类的大小(注意注意成员变量，函数，虚函数，继承等等对类大小的影响)

	class A
	{
	}

	sizeof(A) = 1

含有虚函数的类需要考虑虚函数表指针。

###十八、指针和引用的区别  
区别  

1. 引用是一个别名，只能在定义的时候初始化且必须初始化，之后不可改变。指针可以改变
2. 指针是指向一块内存空间。
3. sizeof 引用得到的是指向的变量(对象)的大小；而sizeof 指针得到的是指针本身的大小。
 

###十九、extern c是干什么的(理清编译器的函数名修饰的机制)?

###二十、volatile的作用(cpu的寄存器缓存机制)
多线程编程中， 如果一个**共享变量**被volatile修饰，则编译器不会把它保存到寄存器中，，当需要使用这个变量的时候，每次都去实际保存这个变量的内存中访问该变量。

###二十一、hash, 为什么hashtable的桶数会取一个素数? 如何有效避免碰撞?  

###二十二、各类排序、快排(如何避免最糟糕的状态)

###二十三、tcp/udp的区别，udp调用connect的作用，tcp连接时序图，状态图

通常，在udp编程中，我们只需要使用下边两个函数就可以了:

	int sendto(int s, const void *msg, size_t len, int flags, const struct sockaddr *to, socklen_t tolen); 

	int  recvfrom(int  s, void *buf, size_t len, int flags, struct sockaddr *from,  socklen_t *fromlen);

并不需要像tcp那样需要预先创建连接，在发送或者接受数据的时候指定地址就可以了。不同有时候在进行udp编程的时候，我们也会先调用connect, 那么调用connect的作用是什么呢?主要有以下几点:  

1. 选定了对端，内核只会将绑定对象的对端发来的数据报传给套接口，因此在一定环境下可以提升安全性  
2. 会返回异步错误，如果对端没启动，默认情况下发送的包对应的ICMP回射包不会给调用进程，如果用了connect，则会回射ICMP
3. 发送两个包间不要先断开再连接，提升了效率

tcp连接时序图参考: [tcp详细分析](http://coderworm.com//%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B/2013/11/11/TCP-conn-status-analysis.html)

###二十四、select和epoll的区别，epoll有哪些触发模式(水平触发和边缘触发的区别以及边缘触发在编程中要做哪些更多的确认)

###二十五、 tcp结束链接怎么握手，time\_wait状态是什么? 为什么会有time\_wait? 哪一方会有time\_wait状态? 如何避免time\_wait状态占用资源?  

主动关闭连接的一端调用close或shutdown，使得tcp协议栈发送FIN分节，对端tcp收到这个FIN分节后,传回应用进程同时并回射ack；此时，应用进程内的read函数返回0，得知它的对端以关闭连接，所以它也要调用close关闭连接，这样，对端tcp同样也发送FIN,主动关闭的这端收到这个FIN并响应ack，此时，主动关闭的这端tcp进入time_wait状态。  

参考这个图:
![img3](https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/tcp-zhuangtai.gif)

为什么需要time\_wait呢？   
因为如果不进入time_wait而直接关闭tcp连接的话，如果最后一个ack因为某些原因没到到达，这可能导致对端tcp重传FIN，那么当重传的FIN到达的时候，因为这个tcp连接已经被关闭了，所以直接响应一个RST，这样就导致重传FIN的这端不是进入有序终止状态而是进入错误状态。

为"离群的段"提供从网络中消失的时间，考虑在同一个端口上的新连接。如果离群的端重新在新连接上被接受，则可能破坏新连接。

###二十六、 对socket套接字设置非阻塞，如果select返回套接字可读，但返回0，什么情况?
对端关闭了链接(调用了close或者shutdown),tcp协议栈发送FIN分节。

###二十七、 常用tcp选项

	TCP_DEFER_ACCEPT	

###二十八、 TCP包头信息
TCP协议头最少20个字节，包括以下的区域：

TCP源端口(Source Port)：16位的源端口其中包含初始化通信的端口。源端口和源IP地址的作用是
标示报问的返回地址。  

TCP目的端口(Destination port)：16位的目的端口域定义传输的目的。这个端口指明报文接收计算
机上的应用程序地址接口。  

TCP序列号（序列码,Sequence Number）：32位  

TCP应答号(Acknowledgment Number)：32位的序列号由接收端计算机使用，重组分段的报文成最初形式。，如果设置了ACK控制位，这个值表示一个准备接收的包的序列码。  

数据偏移量(HLEN)：4位包括TCP头大小，指示何处数据开始。  

保留(Reserved)：6位值域，这些位必须是0。为了将来定义新的用途所保留。  

标志(Code Bits)：6位标志域。表示为：紧急标志、有意义的应答标志、推、重置连接标志、同步序列号标志、完成发送数据标志。按照顺序排列是：URG、ACK、PSH、RST、SYN、FIN。  

窗口(Window)：16位，用来表示想收到的每个TCP数据段的大小。  

校验位(Checksum)：16位TCP头。源机器基于数据内容计算一个数值，收信息机要与源机器数值结果完全一样，从而证明数据的有效性。  

优先指针（紧急,Urgent Pointer）：16位，指向后面是优先数据的字节，在URG标志设置了时才有效。如果URG标志没有被设置，紧急域作为填充。加快处理标示为紧急的数据段。   

选项(Option)：长度不定，但长度必须以字节。如果 没有 选项就表示这个一字节的域等于0。  

数据（Date）：应用程序的数据。  

![img5][tcp-header]

###二十九、 函数入栈的顺序

由于栈地址是从高往低分配的，所以栈底是高地址，栈顶是低地址。函数从右向左入栈。  

###三十、 webproxy多个worker进程产生唯一的序列号

	信号量的工作原理
		
	信号量是一个特殊变量，只允许对它进行等待(wait)和发送信号(signal)这两种操作
	P(信号量变量): 用于等待
	V(信号量变量): 用于发送信号

	PV原语是对整数计数器信号量sv的操作。一次P操作使sv减一，而一次V操作使加一。
	
	P(sv)  
		如果 sv的值 >  零, 就给它减去1,并获得临界区的执行权限。
        如果sv的值 == 零, 就挂起该进程的执行。
	V(sv)  
		如果有其它进程因等待sv而被挂起, 就让它恢复运行;
        如果没有进程因等待sv而被挂起,   就给它加1。

	采用共享内存的方式，生成唯一的序列号

	linux信号量接口:
	int semget(key_t key, int num_sems, int sem_flags); 
	int semop(int sem_id, struct sembuf *sem_opa, size_t num_sem_ops); 
	int semctl(int sem_id, int sem_num, int command, ...);  

### 三十一、 printf的实现原理
printf为可变参数函数，参数从右向左入栈，由于x86的栈空间向下增长，所以只要根据第一个格式参数format的地址向上移动，就可以得到其他参数的地址。  
c语言中相关的宏:

	va_list
	va_start
	va_arg
	va_end  

printf得到参数个数是通过解析格式化字符串的"%"来实现的。 

[linux-process-memory-layout]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-c/linux-process-memory-layout.png  
[process-and-thread]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-c/process-and-thread.png
[tcp-header]: https://raw.github.com/yuxingfirst/blog/gh-pages/_images/linux-network-program/tcp-header.gif

-EOF-
