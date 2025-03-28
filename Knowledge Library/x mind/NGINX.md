# NGINX

## 概念

### Nginx是一种高性能的开源Web服务器和反向代理服务器

- 拓展：正向代理与反向代理

	- 正向代理

		- 可以理解为代理客户端向原始服务器发送请求

	- 反向代理

		- 可以理解为代理服务端管理和处理客户端发送的请求，将请求转发给内网的服务器，或者进行负载均衡

## 特性

### 多进程/多线程模型

- 主进程（Master）负责1.读取并验证nginx.config文件，2.监听端口，接收新的连接请求。而工作进程（Worker）则负责实际处理连接和请求，每一个Worker进程都只维护一个线程，为了避免线程切换。

### 事件驱动架构

- Nginx采用了事件驱动的架构，通过使用异步、非阻塞的方式处理请求，能够更高效地处理大量并发连接

### 事件模块

- Nginx采用了Linux的epoll模型，epoll模型基于事件驱动机制，它可以监控多个事件是否准备完毕，如果准备完毕，那么放入epoll队列中，这个过程是异步的。Worker只需要从epoll队列循环处理即可。避免了传统多线程模型中频繁的上下文切换和线程管理开销

### 模块化架构

- Nginx的功能是通过模块来实现的，它的核心模块提供了基本的Web服务器功能，而其他模块则提供了诸如反向代理、负载均衡、缓存、安全性等功能。

### 高效的内存管理

- Nginx采用了自己的内存池管理机制，通过预先分配一定数量的内存池，并在需要时从内存池中分配内存，避免了频繁的内存分配和释放操作，从而提高了性能和稳定性。

## 配置文件

### 配置项说明

  user                    #设置nginx服务的系统使用用户
  worker_processes        #工作进程数 一般情况与CPU核数保持一致
  error_log               #nginx的错误日志
  pid                     #nginx启动时的pid
  
  events {
      worker_connections    #每个进程允许最大连接数
      use                   #nginx使用的内核模型
  }
  http {
   	sendfile  on                  #高效传输文件的模式 一定要开启
      keepalive_timeout   65        #客户端服务端请求超时时间
      log_format  main   XXX        #定义日志格式 代号为main
      access_log  /usr/local/access.log  main     #日志保存地址 格式代码 main
   	
  	#负载均衡
  	upstream domain {
      	server localhost:8000;
      	server localhost:8001;
  	}   
  	
      server {
          listen 80                          #监听端口;
          server_name localhost              #地址
          
          location / {                       #访问首页路径
              root /xxx/xxx/index.html       #默认目录
              index index.html index.htm     #默认文件
          }        
          
          error_page  500 504   /50x.html    #当出现以上状态码时从新定义到50x.html
          location = /50x.html {             #当访问50x.html时
              root /xxx/xxx/html             #50x.html 页面所在位置
          }        
      }
      
      server {
          ... ... 
      } 
  }
  
## 问题

### nginx如何做到热切换

- 修改配置文件nginx.conf后，重新生成新的worker进程，当然会以新的配置进行处理请求，而且新的请求必须都交给新的worker进程，至于老的worker进程，等把那些以前的请求处理完毕后，kill掉即可。

### nginx是如何处理请求的

- nginx的请求处理过程基本上都是worker进程进行处理的。在处理的过程中，基本的步骤如下： 
1、接收连接：Worker 进程负责接受客户端发起的连接请求，并建立连接。

2、解析请求：Worker 进程负责解析客户端发送的 HTTP 请求，包括请求方法、URL、请求头部等信息。

3、处理请求：Worker 进程根据配置的虚拟主机、URL 路由规则等，决定如何处理该请求。这可能涉及静态文件服务、反向代理、负载均衡等操作。

4、与后端服务通信：如果请求需要访问后端服务（如应用服务器、数据库等），Worker 进程负责与后端服务建立连接，并转发请求。它们还负责处理从后端服务返回的响应。

5、生成响应：Worker 进程根据处理结果生成 HTTP 响应，并将响应发送回客户端。

6、维护连接：Worker 进程负责维护与客户端和后端服务之间的连接，包括保持连接的活跃状态、处理连接的关闭等。  

	- 拓展

		- 如何建立连接

			- 监听端口：
在启动时，Nginx Worker 进程会打开一个或多个监听端口，这些端口用于接收客户端的连接请求。

等待连接：
一旦监听端口打开，Worker 进程开始等待客户端的连接请求。它们通过调用操作系统提供的 I/O 多路复用机制来等待连接事件。

客户端发起连接请求：
当客户端希望与 Nginx 建立连接时，它会向 Nginx 监听的端口发送连接请求。这通常是通过 TCP 协议的三次握手来建立连接的。

接受连接请求：
当有客户端连接请求到达监听端口时，操作系统会通知 Nginx Worker 进程。Worker 进程随后调用 accept 系统调用来接受连接，并为每个连接创建一个新的stock。

建立连接：
一旦连接建立成功，Nginx 和客户端之间就建立了一条通信通道。此时客户端可以向 Nginx 发送 HTTP 请求，并期待获取响应。

处理请求：
建立连接后，Nginx Worker 进程会开始处理该连接上的请求。这包括读取客户端发送的请求数据、解析请求头部、处理请求、生成响应等操作。

		- TCP三次握手和四次挥手

			- 三次握手（TCP 连接建立）
客户端发送 SYN：
客户端向服务端发送一个 SYN（同步）标志的数据包，表示客户端请求建立连接，并选择一个初始序列号（Sequence Number）作为起始通信序列号。

服务端发送 SYN-ACK：
服务端收到客户端的 SYN 后，会回复一个 SYN-ACK（同步-确认）标志的数据包，表示接受客户端的连接请求，并选择自己的初始序列号，并且确认收到了客户端的 SYN 数据包。

客户端发送 ACK：
客户端收到服务端的 SYN-ACK 后，会向服务端发送一个确认标志的 ACK（确认）数据包，表示客户端接受服务端的确认，连接建立成功。

四次挥手（TCP 连接终止）
客户端发送 FIN：
客户端想要关闭连接时，它发送一个 FIN（结束）标志的数据包给服务端，表示不再发送数据，但仍可以接收数据。

服务端发送 ACK：
服务端收到客户端的 FIN 后，会发送一个 ACK（确认）标志的数据包给客户端，确认收到客户端的 FIN。

服务端发送 FIN：
当服务端不再发送数据时，它也会发送一个 FIN 标志的数据包给客户端，表示服务端关闭了输出流，但仍然可以接收数据。

客户端发送 ACK：
客户端收到服务端的 FIN 后，会发送一个 ACK 数据包给服务端，表示客户端接受了服务端的关闭请求，连接终止。

## 作用

### 负载均衡

- 概念

	- 在服务器集群中，Nginx 可以将接收到的客户端请求“均匀地”（严格讲并不一定均匀，可以通过设置权重）分配到这个集群中所有的服务器上。这个就叫做负载均衡。

- 算法

	- 轮询

		- 轮询算法是最简单和最常见的负载均衡算法，将每个新的请求依次分配给下一个服务器，循环往复。这样可以保证请求在各个服务器之间均匀分布，适用于负载均衡要求不高的场景。

	- weight轮询

		- 加权轮询算法在轮询算法的基础上引入了权重的概念，根据服务器的权重来确定每个服务器接收请求的比例。权重越高的服务器，接收到的请求越多。这样可以根据服务器的性能和负载情况，调整请求的分配比例。

	- IP哈希

		- IP哈希算法根据客户端的IP地址计算哈希值，并根据哈希值来确定将请求分配给哪个后端服务器。这样可以保证同一个客户端的请求始终被分配到同一个后端服务器，适用于需要保持会话一致性的场景。

	- 最少链接

		- 最少连接算法将请求分配给当前连接数最少的服务器，以保证后端服务器的负载相对均衡。这样可以避免因为某些服务器连接数过多而导致性能下降的情况。

	- fair

		- 基于响应时间算法根据服务器的响应时间来动态调整请求的分配比例，将请求分配给响应时间最短的服务器。这样可以确保将请求发送给性能最好的服务器，提高系统的整体性能和响应速度。

### 健康检查

- Nginx还带有健康检查（服务器心跳检查）功能，会定期轮询向集群里的所有服务器发送健康检查请求，来检查集群中是否有服务器处于异常状态。一旦发现某台服务器异常，那么在这以后代理进来的客户端请求都不会被发送到该服务器上（直健康检查发现该服务器已恢复正常），从而保证客户端访问的稳定性。

### 动静分离

- 为了加快服务器的解析速度，可以把动态页面和静态页面交给不同的服务器来解析，加快解析速度，降低原来单个服务器的压力。

