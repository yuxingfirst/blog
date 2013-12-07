---
title: posix信号处理
layout: post
category: [linux-network-program]
tags: [network]
---


###信号(signal)

就是通知某个进程发生了某个事件，有时也称为软件中断(software interrupt)。信号通常是异步发生的，也就是说进程预先不知道信号准确发生的时刻。

信号可以：

1. 由一个进程发送给另一个进程(或自身)。

2. 由内核发给某个进程。

 每个信号都有一个与之关联的处置(disposition)，也称为行为(action)。我们通过sigaction函数来设定一个信号的处置，并有三种选择。

1. 我们提供一个函数，他将在特定信号发生的任何时刻被调用。
2. 我们可以发某个信号设置为SIG_IGN来忽略(ignore)它。SIGKILL和SIGSTOP这两个信号不能被忽略。
3. 我们可以把某个信号的处置设定为SIG_DFL来启用他的缺省(default)处理。(缺省处置通常是在收到信号后终止进程，有些信号可能会产生一个core image, 内存影像)。  

### signal函数

建立信号处理的POSIX方法就是调用sigaction函数，不过这个函数有点复杂，它需要接受一个结构参数；比较简单的方法是调用signal函数，signal函数的第一个参数为信号名，第二个参数可以是一个指向函数的指针，也可以是一个常值，如：SIG_IGN，SIG_DFL。不过signal是早于POSIX出现的悠久函数，调用它时，不同的实现提供不同的信号语义以达成后向兼容，而POSIX则明确规定调用sigaction时的信号语义。为了方便，我们一般会定义一个自己的signal函数，在这个函数里边调用sigaction函数，这样就以所期望的POSIX语义提供了一个简单的接口。   

下边我们看来自《UNIX网络编程》卷一的实现：

	sigfunc* signal( int signal, sigfunc *func ) {

	    struct sigaction act, oact;
	    act.sa_handler = func;
	    sigemptyset( &act.sa_mask );
	    act.sa_flags = 0;
	    if ( signo == SIGALRM ) {
		#ifdef   SA_INTERRUPT
		        act.sa_flags |= SA_INTERRUPT;    /* SunOS 4.x */
		#endif
		    } else {
		#ifdef   SA_RESTART
		        act.sa_flags |= SA_RESTART;    /* SVR4, 4.4BSD */
		#endif
	    }
	    if ( sigaction(signo, &act, &oact) < 0 ) {
	        return SIG_ERR;
	    }
	    return oac.sa_handler;

	}  

 在上边这段代码中，sigfunc为一个函数指针，在由signal指定的信号发生的时候调用的处理函数；  

	  sigemptyset( &act.sa_mask );  //设置处理函数的信号掩码  

POSIX允许我们指定这样一组信号，他们在信号处理函数被调用时阻塞(这里的阻塞是指阻塞某个信号或某个信号集，防止它们在阻塞期间递交；不同于系统调用的阻塞)，任何阻塞的信号都不能递交给进程。我们把sa_mask成员设置为空集，意味着在该信号处理函数运行期间，不阻塞额外的信号。POSIX保证被捕获的信号在其信号处理函数运行期间总是阻塞的(其他的信号不能递交给进程)。  

 	“if ( signo == SIGALRM ) { "   // if语句中设置SA_RESTART标志  

SA_RESTART标志是可选的。
> 如果设置，由相应信号中断的系统调用将有内核自动重启。  
> 如果被捕获的信号不是SIGALRM且SA_RESTART有定义，我们就设置该标志，一些较早期的系统(如SunOS 4.x)缺省设置成自动重启被中断的系统调用，并定义了与SA_RESTART互补的SA_INTERRUPT标志，如果定义了该标志，我们就在被捕获的信号是SIGALRM时设置它。假设一个进程正在进行accept系统调用，此时收到一个信号，如果在4.4BSD下，则内核会自动重启被中断的系统调用，accept不会返回错误；如果是在Solaris9下， 由于SA_RESTART标志并没有设置，那么accept会返回一个EINTR错误(被中断的系统调用)。  

最后我们调用sigaction函数，并将相应信号旧的行为作为signal的返回值。   

符合POSIX的系统信号处理总结：  

1. 一旦安装了信号处理函数，它便一直安装者（较早期的系统是每执行一次就将其拆除）。
2. 在一个信号处理函数运行期间，正被递交的信号是阻塞的。
3. 如果一个信号在被阻塞期间产生了一次或多次，那么该信号被解阻塞之后通常只递交一次，也就是说Unix信号缺省是不排队的。
4. 利用sigprocmask函数选择性地阻塞或解阻塞一组信号是可能的。这使得我们可以做到在一段临界区代码执行期间，防止捕获某些信号，以此保护这段代码。    

