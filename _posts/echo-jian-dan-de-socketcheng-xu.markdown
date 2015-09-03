---
layout: post
title: "echo 简单的socket程序"
date: 2015-03-18 13:23:48 +0800
comments: true
toc: true
categories: 
- linux
tags:
- socket


---

看到深入理解计算机系统，socket一张，进而对web服务器有了进一步的认识，但是这里却没有提到关于信号，中断等更多的知识。

要学习网络编程，tcp/ip详解，和stevens的unix 网络编程是必须要看的。

下面是从数据结构到函数的一个介绍。觉得实际写的时候最好有个.thm的帮助文档最好了。  
要不那么多函数和参数很容易记错的。而且里面有大量的强制类型转换。

<!--more-->

## 通信过程 ##

首先客户端和主机通信时建立在socket上的高级i/o操作。
而这个通信线路由唯一的client ip：port和serverip:port确定。

下面这个图真的十分重要。

{% img /img/blog-img/client_server.png %}

下面的两个定义的结构体其实是一样的东西，都是16个字节，只是不同的函数要求传递的参数不同，tcp/io遗留下来的诟病。

``` c

	//ip address
	struct in_addr
	{
		unsigned int s_addr;
	};
	
	struct sockaddr
	{
		unsigned short sa_family;
		char sa_data[14];
	};

 	struct sockaddr_in
	{
		unsigned short sin_family; //总是AF_INT
		unsigned short sin_port;
		struct in_addr sin_addr;
		unsigned char  sin_zero[8]l //为了对齐要求
	}；

```

>* INADDR_ANY就是指定地址为0.0.0.0的地址，这个地址事实上表示不确定地址，或“所有地址”、“任意地址”。
>* 一般来说，在各个系统中均定义成为0值。

在linux下的定义

```
/usr/include/netinet/in.h
/* Address to accept any incoming messages. */
#define INADDR_ANY ((in_addr_t) 0x00000000)
```
定义好的函数

``` c

#include "tiny.h"

int open_clientfd(char *hostname, int port)
{
    int clientfd;
    struct hostent *hp;
    struct sockaddr_in serveraddr;

    // AF_INT 表示的因特网协议，SOCK_STREAM 表示这个套接字是因特网的一个端点
    // 返回的clientfd是部分打开的，还不能直接对套接字读写
    if((clientfd = socket(AF_INT, SOCK_STREAM, 0)) < 0)
        return -1; //创建套接字失败
    if((hp = gethostbyname(hostname)) == NULL)
        return -2; //DNS错误

    bzero((char*)&serveraddr, sizeof(serveraddr));

    serveraddr.sin_family = AF_INT;
    //成员选择的优先级高于类型转换
    //设置服务器地址，已经是大端格式
    bcopy((char*)hp->h_addr_list[0], (char*)serveraddr.sin_addr.s_addr, hp->h_length);
    serveraddr.sin_port = htons(port);

    if(connect(clientfd, (SA *)&serveraddr, sizeof(SA)) < 0)
        return -3; //链接失败
    return clientfd;
}

int open_listenfd(int port)
{
	int listenfd, optval = 1;
	struct sockaddr_in serveraddr;

	if((listenfd = sock(AF_INT, SOCK_STREAM, 0)) < 0)
		return -1;  //创建socket失败


	/* 能够使两个套接字和一个地址绑定，SO_RESUMEADDR */
	if(setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR,
				  (const void *)&optval, sizeof(int)) < 0)
		return -1;
	
	bzero((char*)&serveraddr, sizeof(serveraddr));
	serveraddr.sin_family = AF_INT;
	serveraddr.sin_addr.s_addr = htonl(INADDR_ANY); /* 表示服务器任意ip地址都可以 */
	serveraddr.sin_port = htons(port);
	
	if(bind(listenfd, (SA *)&serveraddr, sizeof(serveraddr)) < 0)
		return -2; // bind函数错误

	if(listen(listenfd, LISTENQ) <0)
	{
		errno = -3;
		fprintf(stderr, "error: %d, listen error\n", errno);
	}
	
	return listenfd;
}

```
## 客户端程序

>* 下面是对应的客户端程序。

``` c echo_client.c

#include "tiny.h"

int main(int argc, char *argv[])
{
	int clientfd, port;
	char *host, buf[MAXLINE];
	rio_t rio;

	if(argc != 3)
	{
		fprintf(stderr, "usage: %s <host> <port>\n", argv[0]);
		exit(0);
	}

	host = argv[1];
	port = atoi(argv[2]);

	clientfd = Open_clientfd(host, port);
	Rio_readinitb(&rio, clientfd);
	
	while(Fgets(buf, MAXLINE, stdin) != NULL)
	{
		Rio_writen(clientfd, buf, strlen(buf));
		Rio_readlineb(&rio, buf, MAXLINE);
		Fputs(buf, stdout);
	}
	close(clientfd);
	return 0;
}

```

## 服务器程序

``` c echo_server.c

#include "tiny.h"

int main(int argc, char **argv)
{
	int lisenfd, connfd, port, clientlen;
	struct sockaddr_in clientaddr;
	struct hostent *hp; /* dns条目 */
	char *haddrp;

	if(argc != 2)
	{
		sprintf(stderr, "usage: %s  <port>\n", argv[0]);
		exit(0);
	}

	port = atoi(argv[1]);
	lisenfd = Open_listenfd(port);
	
	while(1)
	{
		clientlen = sizeof(clientaddr);
		connfd = accept(lisenfd, (SA *)&clientaddr, &clientlen);

		hp = gethostbyaddr((const char *)&clientaddr.sin_addr.s_addr,
							sizeof(clientaddr.sin_addr.s_addr), AF_INET);
		haddrp = inet_ntoa(clientaddr.sin_addr);
		printf("server connected to %s (%s) \n", hp->h_name, haddrp);

		echo(connfd);
		close(connfd);

	}
	exit(0);

}

void echo(int connfd)
{
	size_t n;
	char buf[MAXLINE];
	rio_t rio;

	Rio_readinitb(&rio, connfd);
	
	while((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0)
	{
		printf("server received %d bytes\n", n);
		Rio_writen(connfd, buf, n);
	}
}

```
