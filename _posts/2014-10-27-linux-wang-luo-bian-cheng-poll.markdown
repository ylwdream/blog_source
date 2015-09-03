---
layout: post
title: "linux 网络编程 poll"
date: 2014-10-27 21:11:31 +0800
comments: true
toc: false
categories: linux
tags:
- socket
---

对linux的select函数已经看不懂了，这个poll函数和epoll函数都看不懂，主要是对tcp/ip下面的socket到底是怎么搞得不清楚，先记录下吧，回来有时间继续看。。(是不是太笨了。。。)
<!--more-->

下面是客户端和服务器端的程序

> 客户端程序 

``` c client
#include  <unistd.h>
#include  <sys/types.h>       /* basic system data types */
#include  <sys/socket.h>      /* basic socket definitions */
#include  <netinet/in.h>      /* sockaddr_in{} and other Internet defns */
#include  <arpa/inet.h>       /* inet(3) functions */
#include <netdb.h> /*gethostbyname function */

#include <stdlib.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>

#define MAXLINE 1024

void handle(int connfd);

int main(int argc, char **argv)
{
    char * servInetAddr = "127.0.0.1";
    int servPort = 6888;
    char buf[MAXLINE];
    int connfd;
    struct sockaddr_in servaddr;

    if (argc == 2) {
        servInetAddr = argv[1];
    }
    if (argc == 3) {
        servInetAddr = argv[1];
        servPort = atoi(argv[2]);
    }
    if (argc > 3) {
        printf("usage: echoclient <IPaddress> <Port>\n");
        return -1;
    }

    connfd = socket(AF_INET, SOCK_STREAM, 0);

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(servPort);
    inet_pton(AF_INET, servInetAddr, &servaddr.sin_addr);

    if (connect(connfd, (struct sockaddr *) &servaddr, sizeof(servaddr)) < 0) {
        perror("connect error");
        return -1;
    }
    printf("welcome to echoclient\n");
    handle(connfd);     /* do it all */
    close(connfd);
    printf("client exit\n");
    exit(0);
}

void handle(int sockfd)
{

    int n;
    for (;;) {
        char sendline[MAXLINE] = {0}, recvline[MAXLINE] = {0};
        
        if (fgets(sendline, MAXLINE, stdin) == NULL) {
            break;//read eof
        }
        /*
        //也可以不用标准库的缓冲流,直接使用系统函数无缓存操作
        if (read(STDIN_FILENO, sendline, MAXLINE) == 0) {
            break;//read eof
        }
        */
        
        if (strncmp("exit", sendline, 4) == 0) {
            // close(connfd);
            // printf("client exit, close socket(%d)\n", connfd);
            break;
        }

        n = write(sockfd, sendline, strlen(sendline));
        n = read(sockfd, recvline, MAXLINE);
        if (n == 0) {
            printf("echoclient: server terminated prematurely\n");
            break;
        }
        write(STDOUT_FILENO, recvline, n);
        //如果用标准库的缓存流输出有时会出现问题
        //fputs(recvline, stdout);
    }
}

```

> 服务器端的程序

``` c "server poll"

#include  <unistd.h>
#include  <sys/types.h>       /* basic system data types */
#include  <sys/socket.h>      /* basic socket definitions */
#include  <netinet/in.h>      /* sockaddr_in{} and other Internet defns */
#include  <arpa/inet.h>       /* inet(3) functions */

#include <stdlib.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>


#include <poll.h> /* poll function */
#include <limits.h>

#define MAXLINE 10240

#ifndef OPEN_MAX
#define OPEN_MAX 40960
#endif

void handle(struct pollfd* clients, int maxClient, int readyClient);

int  main(int argc, char **argv)
{
    char * servInetAddr = "0.0.0.0";
    int servPort = 6888;
    int listenq = 1024;
    int listenfd, connfd;
    struct pollfd clients[OPEN_MAX];
    int  maxi;
    socklen_t socklen = sizeof(struct sockaddr_in);
    struct sockaddr_in cliaddr, servaddr;
    char buf[MAXLINE];
    int nready;

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
    
    bzero(&servaddr, socklen);
    servaddr.sin_family = AF_INET;
    // servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    inet_pton(AF_INET, servInetAddr, &servaddr.sin_addr);
    servaddr.sin_port = htons(servPort);

    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd < 0) {
        perror("socket error");
    }

    int opt = 1;
    if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) < 0) {
        perror("setsockopt error");
    }

    if(bind(listenfd, (struct sockaddr *) &servaddr, socklen) == -1) {
        perror("bind error");
        exit(-1);
    }
    if (listen(listenfd, listenq) < 0) {
        perror("listen error");    
    }

    clients[0].fd = listenfd;
    clients[0].events = POLLIN;
    int i;
    for (i = 1; i< OPEN_MAX; i++) 
        clients[i].fd = -1; 
    maxi = listenfd + 1;

    printf("echo server [poll] startup, listen on port:%d\n", servPort);
    printf("max connection is %d\n", OPEN_MAX);

    for ( ; ; )  {
        nready = poll(clients, maxi + 1, -1);
        //printf("nready is %d\n", nready);
        if (nready == -1) {
            perror("poll error");
        }
        if (clients[0].revents & POLLIN) {
            connfd = accept(listenfd, (struct sockaddr *) &cliaddr, &socklen);
            sprintf(buf, "new socket(%d) accept client form %s:%d\n", connfd, inet_ntoa(cliaddr.sin_addr), cliaddr.sin_port);
            printf(buf, "");

            for (i = 0; i < OPEN_MAX; i++) {
                if (clients[i].fd == -1) {
                    clients[i].fd = connfd;
                    clients[i].events = POLLIN;
                    break;
                }
            }

            if (i == OPEN_MAX) {
                fprintf(stderr, "too many connection, more than %d\n", OPEN_MAX);
                close(connfd);
                continue;
            }

            if (i > maxi)
                maxi = i;

            --nready;
        }

        handle(clients, maxi, nready);
    }
}

void handle(struct pollfd* clients, int maxClient, int nready) {
    int connfd;
    int i, nread;
    char buf[MAXLINE + 1] = {0};

    if (nready == 0)
        return;

    for (i = 1; i< maxClient; i++) {
        connfd = clients[i].fd;
        if (connfd == -1) 
            continue;
        if (clients[i].revents & (POLLIN | POLLERR)) {
            nread = read(connfd, buf, MAXLINE);//读取客户端socket流
            if (nread < 0) {
                perror("read error");
                close(connfd);
                clients[i].fd = -1;
                continue;
            }
            if (nread == 0) {
                printf("client close the connection, socket(%d)\n", connfd);
                close(connfd);
                clients[i].fd = -1;
                continue;
            }
            
            buf[nread] = '\0';
            printf("sockte(%d) recv : %s", connfd, buf);
            
            write(connfd, buf, nread);//响应客户端  
            if (--nready <= 0)//没有连接需要处理，退出循环
                break;
        }
    }
}
```
