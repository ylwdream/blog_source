---
layout: post
title: "linux 网络编程 epoll"
date: 2014-10-27 21:57:24 +0800
comments: true
toc: false
categories: linux
tags:
- socket
---
进阶路线之epoll，这个函数是系统函数，非root用户还不可以执行。
<!--more-->

客户端和上一个介绍的poll函数的客户端程序一样。


> 服务器端的程序

``` c "server poll"

#include <unistd.h>
#include <sys/types.h>          /* basic system data types */
#include <sys/socket.h>         /* basic socket definitions */
#include <netinet/in.h>         /* sockaddr_in{} and other Internet defns */
#include <arpa/inet.h>          /* inet(3) functions */
#include <sys/epoll.h>          /* epoll function */
#include <fcntl.h>              /* nonblocking */
#include <sys/resource.h>       /* setrlimit */

#include <stdlib.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>



#define MAXEPOLLSIZE 10000
#define MAXLINE 1024
int handle(int connfd);
int setnonblocking(int sockfd)
{
    if (fcntl(sockfd, F_SETFL, fcntl(sockfd, F_GETFD, 0)|O_NONBLOCK) == -1) {
        return -1;
    }
    return 0;
}

int main(int argc, char **argv)
{
    char * servInetAddr = "0.0.0.0";
    int  servPort = 6888;
    int listenq = 1024;

    int listenfd, connfd, kdpfd, nfds, n, nread, curfds,acceptCount = 0;
    struct sockaddr_in servaddr, cliaddr;
    socklen_t socklen = sizeof(struct sockaddr_in);
    struct epoll_event ev;
    struct epoll_event events[MAXEPOLLSIZE];
    struct rlimit rt;
    char buf[MAXLINE];
    
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

    /* 设置每个进程允许打开的最大文件数 */
    rt.rlim_max = rt.rlim_cur = MAXEPOLLSIZE;
    if (setrlimit(RLIMIT_NOFILE, &rt) == -1) 
    {
        perror("setrlimit error");
        return -1;
    }


    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET; 
    // servaddr.sin_addr.s_addr = htonl (INADDR_ANY);
    inet_pton(AF_INET, servInetAddr, &servaddr.sin_addr);
    servaddr.sin_port = htons (servPort);

    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        perror("can't create socket file");
        return -1;
    }

    int opt = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    if (setnonblocking(listenfd) < 0) {
        perror("setnonblock error");
    }

    if (bind(listenfd, (struct sockaddr *) &servaddr, sizeof(struct sockaddr)) == -1)
    {
        perror("bind error");
        return -1;
    } 
    if (listen(listenfd, listenq) == -1) 
    {
        perror("listen error");
        return -1;
    }
    /* 创建 epoll 句柄，把监听 socket 加入到 epoll 集合里 */
    kdpfd = epoll_create(MAXEPOLLSIZE);
    ev.events = EPOLLIN | EPOLLET;
    ev.data.fd = listenfd;
    if (epoll_ctl(kdpfd, EPOLL_CTL_ADD, listenfd, &ev) < 0) 
    {
        fprintf(stderr, "epoll set insertion error: fd=%d\n", listenfd);
        return -1;
    }
    curfds = 1;

    printf("echo server [epoll] startup,port %d, max connection is %d, backlog is %d\n", servPort, MAXEPOLLSIZE, listenq);

    for (;;) {
        /* 等待有事件发生 */
        nfds = epoll_wait(kdpfd, events, curfds, -1);
        if (nfds == -1)
        {
            perror("epoll_wait");
            continue;
        }
        /* 处理所有事件 */
        for (n = 0; n < nfds; ++n)
        {
            if (events[n].data.fd == listenfd) 
            {
                connfd = accept(listenfd, (struct sockaddr *)&cliaddr,&socklen);
                if (connfd < 0) 
                {
                    perror("accept error");
                    continue;
                }

                sprintf(buf, "accept form %s:%d\n", inet_ntoa(cliaddr.sin_addr), cliaddr.sin_port);
                printf("%d:%s", ++acceptCount, buf);

                if (curfds >= MAXEPOLLSIZE) {
                    fprintf(stderr, "too many connection, more than %d\n", MAXEPOLLSIZE);
                    close(connfd);
                    continue;
                } 
                if (setnonblocking(connfd) < 0) {
                    perror("setnonblocking error");
                }
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = connfd;
                if (epoll_ctl(kdpfd, EPOLL_CTL_ADD, connfd, &ev) < 0)
                {
                    fprintf(stderr, "add socket '%d' to epoll failed: %s\n", connfd, strerror(errno));
                    return -1;
                }
                curfds++;
                continue;
            } 
            // 处理客户端请求
            if (handle(events[n].data.fd) < 0) {
                epoll_ctl(kdpfd, EPOLL_CTL_DEL, events[n].data.fd,&ev);
                curfds--;


            }
        }
    }
    close(listenfd);
    return 0;
}
int handle(int connfd) {
    int nread;
    char buf[MAXLINE];
    nread = read(connfd, buf, MAXLINE);//读取客户端socket流

    if (nread == 0) {
        printf("client close the connection\n");
        close(connfd);
        return -1;
    } 
    if (nread < 0) {
        perror("read error");
        close(connfd);
        return -1;
    }    
    
    buf[nread] = '\0';
    printf("sockte(%d) recv : %s", connfd, buf);
    
    write(connfd, buf, nread);//响应客户端  
    return 0;
}
```
