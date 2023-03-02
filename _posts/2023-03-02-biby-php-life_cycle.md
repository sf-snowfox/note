# 生命周期
### 信号处理
unix信号

### SAPI
Server Application Programimg Interface 服务端应用编程接口。

相当于PHP外部环境的代理器。

核心数据结构 \_sapi\_module\_struct。

CLI和FPM都是基于SAPI的实现。

### CLI模式
Command Line Interface 命令行接口

#### PHP执行流程的5大阶段
1. 模块初始化

对于FPM模式，进程启动后只会进行一次模块初始化，进而进入循环（休眠阻塞等到accept请求）。

2. 请求初始化
3. 执行阶段
4. 请求关闭

请求关闭后，FPM模式会循环等待请求到来，继续进行请求的初始化。

而CLI模式将进入模块关闭阶段。

5. 模块关闭

### 模式
FastCGI Process Manager，FastCGI进程管理器

支持平滑重启PHP以及重载PHP配置

#### FPM生命周期
FPM模式的生命周期也是5个阶段。

FPM是常驻内存的进程，其模块初始化只做一次，便进入循环，模块关闭也只是在进程退出时做一次。

请求次数大于max\_requests，进程退出。

1. 模块初始化

调用 php\_module\_startup 之后，进入循环，调用 fcgi\_accept\_requet（实际调用accept），阻塞等待请求，如果请求进来，会被唤起，进入php\_request\_startup，初始化请求。

2. 请求初始化
3. 执行阶段
4. 请求关闭

销毁所有全局变量(super-globals) 释放所有request全局变量 fpm在此等待请求到来，cli模式进入到模块关闭阶段。

5. 模块关闭

#### 进程模型
PHP-FPM是多进程服务，其中有一个master进程（做管理工作）和多个worker进程（处理数据请求）。

##### master进程跟worker进程创建
PHP-FPM进程设置的三种方式

1. static
2. dynamic
3. ondemand

##### 启动过程
calling process(fork master 后退出) -> fork master -> worker

创建完成之后，请求的处理工作由worker进程进行，master进程负责对worker进程进行监控和管理。

##### 进程间如何通信
通过Unix信号

##### Worker进程是如何工作的
###### 网络编程

1. Socket创建

在Linux中，Nginx服务器和PHP-FPM通信可以通过TCP Socket 和 Unix Socket 两种方式实现。

Unix Socket是一种终端，可以使同一台操作系统上的两个或多个进程进行数据通信。

这种方式需要在Nginx配置文件中填写PHP-FPM的pid文件位置，效率比TCP Socket高。

TCP Socket的优点是可以跨服务器。当Nginx跟PHP-FPM不在同一台机器上时，只能使用这种方式。

* 配置如下：

```bash
    location ～ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name; ;
        fastcgi_pass 127.0.0.1:9000; //TCP socket
        #fastcgi_pass unix:/var/run/php7-fpm.sock; //UNIX socket
        fastcgi_index index.php;
    }
```
2. accept请求。

worker进入循环，当没有请求时，会阻塞在 fcgi\_accept\_request ，让出CPU资源，成为空闲进程。

当请求到达时，会有一个worker进程抢到并处理，进入FastCGI的处理阶段。

##### 小结
master进程创建Socket，worker进程通过创建的fd来accept请求。抢占式接收。

FROM模式是多进程模式。首先由 calling process 进程fork出master进程，master进程创建Socket，然后fork出worker进程，worker进程会在accept处阻塞等待，请求过来时，由其中一个worker进程处理，按照FastCGI模式进行各阶段的读取，然后解析PHP并执行，最后按照FastCGI协议返回数据，继续进入accept处阻塞等待。

另外，FPM建立了计分榜机制，可以关注全局和每个worker的工作情况，方便使用者监控。

### CGI模式
CGI将Web服务器和PHP执行程序连接起来，把接收的指令传递给PHP执行程序，再把服务器执行程序的结果返还给Web服务器。

对于每一个用户请求，都会先创建CGI的子进程，然后处理请求，处理完后结束这个子进程，这就是fork-and-execute模式。

用户请求数量非常多会大量挤占系统的资源（如内存、CPU时间等），造成效率低下。

对于采用CGI模式的服务器，有多少连接请求，就会有多少CGI子进程，子进程反复加载也是导致CGI性能低下的主要原因，这也是FastCGI出现的原因。

### Embed模式
允许在C/C++语言中调用PHP/ZE提供的函数

### PHPDBG模式
PHPDBG 提供类似GDB功能，支持但不调试，可以灵活打断点