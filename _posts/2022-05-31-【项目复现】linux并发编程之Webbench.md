---

layout:     post
title:      「项目复现」linux并发编程之Webbench
subtitle:   Linux系统
date:       2022-05-31
author:     MRL Liu,WJ
header-img: img/post-bg-hacker.png
catalog: true
tags:
    - 项目复现
---

​	   Web bench（[网页工作台](http://home.tiscali.cz/~cz210552/webbench.html)）是一款Linux系统上用C语言编写的专用的轻量级的网站压力测试工具，最多可以对一个网站模拟3w左右的并发请求，对中小型网站有明显的效果。

## 1、Web bench的安装

web bench是一个压缩包（webbench-1.5.tar.gz），可以手动从[网页工作台](http://home.tiscali.cz/~cz210552/webbench.html)下载，也可以使用命令行下载：

```bash
wget http://www.ha97.com/code/webbench-1.5.tar.gz # 用wget下载webbench-1.5.tar.gz
tar xf  webbench-1.5.tar.gz # 解压webbench-1.5.tar.gz中的所有文件
yum install gcc*  ctags* -y # 安装gcc编译环境
make && make install # 编译并且安装编译程序
```

## 2、Web bench的使用

在终端使用如下命令行启动测试

```bash
webbench -c 100 -t 30 http://127.0.0.1:1316/
```

其中`-c`表示设置客户端访问的数量，可省略，默认为1；`-t`表示设置客户端访问的时间，可省略，默认为30s；

得到立即反馈：

```bash
Benchmarking:GET http://127.0.0.1:1316/
100 clients,running 30 sec.
```

执行完毕后得到类似反馈：

```bash
Speed=513172 pages/min,27651526 bytes/sec.
Requests: 256586 susceed,0 failed.
```

#### **使用的注意事项**

1）webbench 做压力测试时，自身也会消耗CPU和内存资源，为了测试准确，请将 webbench 安装在别的服务器上。

2）**测试时并发应当由小逐渐加大，比如并发100时观察一下网站负载是多少、打开页面是否流畅，并发200时又是多少、网站打开缓慢时并发是多少、网站打不开时并发又是多少；**

## 3、Web bench的工作流程

webbench-1.5.tar.gz中的文件大致分为如下三类：安装好后，就会得到如下3个

> 源码文件：socket.c、webbench.c
>
> 脚本文件：MakeFile
>
> 编译文件：webbench、webbench.o

其中脚本文件MakeFile表明如何编译源码

编译文件是通过脚本文件MakeFile编译后生成的，即在终端切换到该文件夹所在路径，执行make，就会生成webbench、webbench.o文件。

可以看到，该工具十分简单，源码只有两个文件：socket.c、webbench.c。

其程序执行的工作流程如下：

```C++
int main(){
   1、处理终端输入的命令行参数，检查参数是否合法
   2、调用build_request()，输入URL路径，构造一个HTTP请求，存储在全局字符数组request中
   3、打印创建HTTP请求信息,例如"Benchmarking:GET http://127.0.0.1:1316/"等
   4、调用bench():
    	1、尝试创建Socket对象连接服务端程序
        2、尝试创建管道pipe
        3、通过fork()创建client个子进程
            1、在每个子进程中，调用benchcore()在测试时间内不断向服务器建立连接并发送http请求：
            	1、设置闹钟alarm(benchtime)，达到规定时间后产生SIGALRM信号，执行timerexpired=1
            	2、进入死循环while(1):
    				1、检查timerexpired是否为1，若为1，退出循环
                    2、创建一个Socket对象连接服务端程序，更新访问结果
                    3、借助Socket对象向服务端发送HTTP请求，更新访问结果
                    4、接收服务端返回的HTTP相应数据，更新访问结果
            2、将访问结果通过fdopen(,"w")写入管道
        4、通过fdopen(,"r")从管道中读取子进程写入的数据，累计全部数据
        5、打印结果数据
}
```

`socket.c`中只有一个方法`int Socket`，用来建立与目标的TCP连接，并返回客户端连接使用的套接字，对命令行参数的处理，构建请求，创建子进程进行压力测试等在`webbench.c`文件中完成。

## 4、Web bench源码的知识点

## （1）父进程创建多个子进程

#### fork()的使用

Linux中通过fork()来创建子进程

在执行fpid=fork()之前，只有当前的父进程在执行这段代码

在执行fpid=fork()之后，当前的父进程和新复制出的子进程都开始并发执行之后的代码

所以为了父进程和子进程执行不同的代码，需要根据fpid的值分别处理，否则就会执行相同的代码。

子进程的fork返回值是0，父进程的fork返回值>0，这也是区分父进程和子进程的方法，至于其他的内容，在fork之前的东西两个进程的一样的。

#### fork()的底层

在调用fork()时，Linux的内核程序实际上只会复制父进程的页表以及给子进程创建一个进程描述符，并不会立即开辟新的内存空间来复制一份父进程的数据，此时父进程和子进程共享同一份数据资源，只有父进程或者子进程发生写的操作时，子进程才会复制父进程的数据到一个新的内存空间，这种技术就是写时拷贝技术。

但是在程序员看来，fork()就是立即克隆出了一个和父进程完全相同的子进程，只不过其底层使用了写时拷贝技术来优化。

#### 父子进程的关闭

假如父进程先于子进程关闭，那么子进程就成为孤儿进程，被操作系统接管

假如子进程先于父进程终止，而父进程又没有调用wait或waitpid函数，父进程结束后，子进程成为僵尸进程，此时子进程虽然执行完毕但是始终占有内核资源，同时也减少了系统可以创建的最大进程数。

假如子进程先于父进程终止，而父进程调用了wait或waitpid函数，那么父进程会等待子进程结束后再结束，子进程不会成为僵尸进程。

#### 父进程创建一个子进程

```C++
int fd = fork();
if(fd < 0){
	exit(1);
} else if(fd == 0){
	// child code
} else {
	// father code
}
```

#### 父进程创建2个不同的子进程

```C++
int fd1,fd2;

fd1 = fork();
if(fd < 0){
	exit(1);
} else if(fd == 0){
	// child 1 code
} else {
	fd2 = fork();
	if(fd2 < 0){
		exit(1);
	} else if(fd2 == 0){
		// child 2 code
	} else {
		// father code
	}
}
```

#### 父进程创建N个相同的子进程

```C++
// 父进程创建管道
int mypipe[2];
int rt=pipe(mypipe);
FILE *f;
// 父进程创建n个子进程
pid_t pid;
for(int i = 0; i < n; i++){
	pid=fork();
	if(pid<=0){
	   sleep(1);
	   break;// 必不可少
	} 
}
// 不同的进程执行不同的代码
if(pid < 0){
	exit(1);
} 
else if(pid == 0){
	// child code
} 
else {
	// father code
}
```

## （2）父子进程通信：匿名管道

这里使用的是Linux的匿名管道（因为其创建管道使用了pipe(),命名管道使用mkfifo()），其基本用法是

匿名管道的特点如下：

> 1、匿名管道是一种**半双工通信**，数据只能从写端流入读端。
>
> 2、匿名管道读写的是**内核程序**的缓冲区的两端fd[0]和fd[1]。
>
> 3、匿名管道只能用于**父子进程通信**。这是因为匿名管道没有标识符，其他无亲缘关系的进程无法访问到该管道，只有创建它的父进程和使用它的子进程可以访问。**在进行父子进程间通信时，一定要父进程先创建管道，再创建子进程**，这样子进程才能访问匿名管道。
>
> 4、匿名管道默认是**阻塞**的，父子进程抢占式运行，如果父进程或者子进程先读，发现是空管道时就会阻塞，等待写入；某个进程写入一次后，读进程抢占到才能读入。
>
> 5、管道**读取数据是将数据剪切**，所以多个进程读管道，无法拿到相同的数据
>
> 6、管道的读写具有**原子性**，即读操作或写操作要么完成，要么没有开始，不存在边读边写，或者同时写
>
> 7、管道的读写是**字节流**，用户write两次写入字符串，对于管道来说两次输入的数据没有明显边界，read可以读任意字节大小，在读取时无法区分两次输入，前后两次输入粘连在一起，read的返回值是读取到数据的字节数。

```C++
// 父进程创建管道
int mypipe[2];
int rt=pipe(mypipe);
FILE *f;
// 父进程创建client个子进程
pid_t pid;
for(int i = 0; i < n; i++){
	pid=fork();
	if(pid<=0){
	   sleep(1);
	   break;// 必不可少
	} 
}
if(pid<0){
   sys_err("fork err");
}
// 子进程向管道写数据
else if (pid == 0){
    fdopen(mypipe[1],"w");//打开管道的写端
    fprintf(f,"%d %d %d\n",speed,failed,bytes);// 子进程每次写入一行格式化的数据
    fclose(f);
}
// 父进程从管道读数据
else{
    fdopen(mypipe[0],"r");//打开管道的读端
    while(client--){
        fscanf(f,"%d %d %d",&i,&j,&k);// 父进程每次读入一行格式化的数据
    }
    fclose(f);
}
```

## （3）HTTP请求格式

```
"GET /index.html  HTTP/1.1
connection:close
Host:www.google.cn"
```

通过HTTP1.1版本协议请求index.html页面；connection: close指明使用短连接，即服务器返回后就断开连接；Host字段指明页面所在的主机名。





