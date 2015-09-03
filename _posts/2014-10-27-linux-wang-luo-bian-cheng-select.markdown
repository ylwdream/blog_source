---
layout: post
title: "linux 网络编程 select"
date: 2014-10-27 20:29:48 +0800
comments: true
toc: false
categories: linux
tags:
- socket 
---
> linux中的`accept`函数，接受一个客户端发来的链接请求。 
``` c
	SOCKET accept(	
	       SOCKET             　　s,
	       struct sockaddr FAR　　*addr,
		   int FAR　　　　　　　　　*addrlen
			);
```
<!--more-->
返回值是一个新的套接字描述符，它代表的是和客户端的新的连接，可以把它理解成是一个客户端的socket,这个socket包含的是客户端的ip和port信息 。（当然这个new_socket会从sockfd中继承 服务器的ip和port信息，两种都有了），而参数中的SOCKET   s包含的是服务器的ip和port信息 本函数从s的等待连接队列中抽取第一个连接，创建一个与s同类的新的套接口并返回句柄。如果队列中无等待连接，且套接口为阻塞方式，则accept()阻塞调用进程直至新的连接出现。如果套接口为非阻塞方式且队列中无等待连接，则accept()返回一错误代码。已接受连接的套接口不能用于接受新的连接，原套接口仍保持开放。

----
基本的服务端的方式
``` c
	int socket(int domain, int type, int protocol);
```
参数说明：
　　
domain：协议域，又称协议族（family）。常用的协议族有AF_INET、AF_INET6、AF_LOCAL（或称AF_UNIX，Unix域Socket）、AF_ROUTE等。协议族决定了socket的地址类型，在通信中必须采用对应的地址，如AF_INET决定了要用ipv4地址（32位的）与端口号（16位的）的组合、AF_UNIX决定了要用一个绝对路径名作为地址。

type：指定Socket类型。常用的socket类型有SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET等。流式Socket（SOCK_STREAM）是一种面向连接的Socket，针对于面向连接的TCP服务应用。数据报式Socket（SOCK_DGRAM）是一种无连接的Socket，对应于无连接的UDP服务应用。

protocol：指定协议。常用协议有IPPROTO_TCP、IPPROTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC等，分别对应TCP传输协议、UDP传输协议、STCP传输协议、TIPC传输协议。
注意：type和protocol不可以随意组合，如SOCK_STREAM不可以跟IPPROTO_UDP组合。当第三个参数为0时，会自动选择第二个参数类型对应的默认协议。

-----

每次accept只能处理等待的socket一个链接。
select 模型
``` c "select server"
#include  <unistd.h>
#include  <sys/types.h>       /* basic system data types */
#include  <sys/socket.h>      /* basic socket definitions */
#include  <netinet/in.h>      /* sockaddr_in{} and other Internet defns */
#include  <arpa/inet.h>       /* inet(3) functions */
#include <sys/select.h>       /* select function*/

#include <stdlib.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>

#define MAXLINE 10240

void handle(int * clientSockFds, int maxFds, fd_set* pRset, fd_set* pAllset);

int  main(int argc, char **argv)
{
    char * servInetAddr = "0.0.0.0";
    int  servPort = 6888;
    int listenq = 1024;

    int  listenfd, connfd;
    struct sockaddr_in cliaddr, servaddr;
    socklen_t socklen = sizeof(struct sockaddr_in);
    int nready, nread;
    char buf[MAXLINE];
    int clientSockFds[FD_SETSIZE];
    fd_set allset, rset;
    int maxfd;

    // 使用参数适配 IP:Port
    if (argc == 2) {
        servInetAddr = argv[1];
    }
    if (argc == 3) {
        servInetAddr = argv[1];
        servPort = atoi(argv[2]);
    }
    if (argc > 3) {
        printf("usage: exename <IPaddress> <Port>\n");
        return -1;
    }
    
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd < 0) {
        perror("socket error");
        return -1;
    }

    int opt = 1;
    if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt error");    
    }  

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    // servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    inet_pton(AF_INET, servInetAddr, &servaddr.sin_addr);
    servaddr.sin_port = htons(servPort);


    if(bind(listenfd, (struct sockaddr*)&servaddr, socklen) == -1) {
        perror("bind error");
        exit(-1);
    }

    if (listen(listenfd, listenq) < 0) {
        perror("listen error");
        return -1;
    }

    int i = 0;
    for (i = 0; i< FD_SETSIZE; i++) 
        clientSockFds[i] = -1; 
    FD_ZERO(&allset);
    FD_SET(listenfd, &allset); 
    maxfd = listenfd;    

    printf("echo server [select] startup, listen on port %d\n", servPort);
    printf("max connection: %d\n", FD_SETSIZE);

    for ( ; ; )  {
        rset = allset;
        nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
        if (nready < 0) {
            perror("select error");
            continue;
        }
        if (FD_ISSET(listenfd, &rset)) {
            connfd = accept(listenfd, (struct sockaddr*) &cliaddr, &socklen);
            if (connfd < 0) {
                perror("accept error");
                continue;
            }

            sprintf(buf, "new socket(%d) accept form %s:%d\n", connfd, inet_ntoa(cliaddr.sin_addr), cliaddr.sin_port);
            printf(buf, "");

            for (i = 0; i< FD_SETSIZE; i++) {
                if (clientSockFds[i] == -1) {
                    clientSockFds[i] = connfd;
                    break;
                }
            }
            if (i == FD_SETSIZE) {
                fprintf(stderr, "too many connection, more than %d\n", FD_SETSIZE);
                close(connfd);
                continue;
            }
            if (connfd > maxfd)
                maxfd = connfd;

            FD_SET(connfd, &allset);
            if (--nready <= 0)
                continue;
        }

        handle(clientSockFds, maxfd, &rset, &allset);
    }
}


void handle(int * clientSockFds, int maxFds, fd_set* pRset, fd_set* pAllset) {
    int nread;
    int i;
    char buf[MAXLINE];
    for (i = 0; i< maxFds; i++) {
        if (clientSockFds[i] != -1) {
            if (FD_ISSET(clientSockFds[i], pRset)) {
                nread = read(clientSockFds[i], buf, MAXLINE);//读取客户端socket流
                if (nread < 0) {
                    perror("read error");
                    close(clientSockFds[i]);
                    FD_CLR(clientSockFds[i], pAllset);
                    clientSockFds[i] = -1;
                    continue;
                }
                if (nread == 0) {
                    printf("client close the connection\n");
                    close(clientSockFds[i]);
                    FD_CLR(clientSockFds[i], pAllset);
                    clientSockFds[i] = -1;
                    continue;
                } 
                
                buf[nread] = '\0';
                printf("sockte(%d) recv : %s", clientSockFds[i], buf);
                write(clientSockFds[i], buf, nread);//响应客户端  有可能失败，暂不处理
            }
        }
    }

}
```
