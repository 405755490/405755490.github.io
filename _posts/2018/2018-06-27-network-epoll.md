---
layout: post
title:      "Epoll"
excerpt:    "epoll示例"
category:   c++
tags:       [c++]
mathjax:    false
date:       2018-06-27
author:     "Undefined"
---

## 1.分析

STDIN_FILENO：接收键盘的输入

STDOUT_FILENO：向屏幕输出


> epoll步骤

参考：IO多路复用之epoll总结 [https://www.cnblogs.com/Anker/p/3263780.html](https://www.cnblogs.com/Anker/p/3263780.html)

	#include <sys/epoll.h>
	int epoll_create(int size);
	int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
	int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

+ 调用epoll_create()建立一个epoll对象(在epoll文件系统中为这个句柄对象分配资源)

+ 调用epoll_ctl向epoll对象中添加这100万个连接的套接字

	EPOLL_CTL_ADD：注册新的fd到epfd中；
	EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
	EPOLL_CTL_DEL：从epfd中删除一个fd；
	
+ 调用epoll_wait收集发生的事件的连接


关于ET、LT两种工作模式

LT：`水平触发`，效率会低于ET触发，尤其在大并发，大流量的情况下。但是LT对代码编写要求比较低，不容易出现问题。LT模式服务编写上的表现是：只要有数据没有被获取，内核就不断通知你，因此不用担心事件丢失的情况。

ET：`边缘触发`，效率非常高，在并发，大流量的情况下，会比LT少很多epoll的系统调用，因此效率高。但是对编程要求高，需要细致的处理每个请求，否则容易发生丢失事件的情况。
	
	ev.events = EPOLLIN; //，默认LT模式
	ev.events = EPOLLIN | EPOLLET;  表示ET模式

	
	
	
## 2.服务端代码

	#include <stdio.h>
	#include <stdlib.h>
	#include <string.h>
	#include <errno.h>

	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <arpa/inet.h>
	#include <sys/epoll.h>
	#include <unistd.h>
	#include <sys/types.h>

	#define IPADDRESS "127.0.0.1"
	#define PORT 6666
	#define MAXSIZE 1024
	#define LISTENQ 5 
	#define FDSIZE 1000
	#define EPOLLEVENTS 100

	int socket_bind(const char*ip, int port);
	void  do_epoll(int listenfd );
	void handle_events(int epollfd ,struct epoll_event* events ,int num ,int listenfd ,char*buf);
	void handle_accpet(int epolled ,int listenfd);
	void do_read(int epolled , int fd ,char* buf);
	void do_write(int epolled ,int fd ,char *buf);
	void add_event(int epolled ,int fd, int state );
	void modify_event(int epolled ,int fd ,int state );
	void delete_event(int epolled ,int fd, int state );


	int main(int argc , char* argv[])
	{
		int listenfd;
		listenfd = socket_bind(IPADDRESS,PORT);
		listen(listenfd, LISTENQ);
		do_epoll(listenfd);
		return 0 ;
	}

	int socket_bind(const char*ip, int port){
		int listenfd;
		struct sockaddr_in servaddr;
		listenfd = socket(AF_INET, SOCK_STREAM,0);
		if (listenfd == -1 ){
			perror("socket error:");
			exit(1);
		}

		bzero(&servaddr ,sizeof(servaddr));
		servaddr.sin_family = AF_INET;
		inet_pton(AF_INET,ip,&servaddr.sin_addr);
		servaddr.sin_port = htons(port);
		if ( bind (listenfd, (struct sockaddr*)&servaddr ,sizeof(servaddr)) == -1){
			perror("bind error: ");
			exit(1);
		}
		return listenfd;
	}

	void do_epoll(int listenfd){
		int epollfd;
		struct epoll_event events[EPOLLEVENTS];
		int ret;
		char buf[MAXSIZE];
		memset(buf,0,sizeof(buf));

		epollfd = epoll_create(FDSIZE);

		add_event(epollfd, listenfd, EPOLLIN);

		while(1){
			ret = epoll_wait(epollfd ,events ,EPOLLEVENTS ,-1);
			handle_events(epollfd, events ,ret ,listenfd, buf);
		}
		close(epollfd);
	}

	void handle_events(int epollfd, struct epoll_event *events ,int nums, int listenfd, char*buf){
		int i , fd ;

		for (i = 0 ; i < nums; i++){
			fd = events[i].data.fd;
			if ((fd == listenfd) && (events[i].events & EPOLLIN))
				handle_accpet(epollfd ,listenfd);
			else if (events[i].events & EPOLLIN )
				do_read(epollfd, fd, buf);
			else if ( events[i].events & EPOLLOUT)
				do_write(epollfd, fd ,buf);
		}
	}

	void handle_accpet(int epollfd ,int listenfd){
		int clifd ;
		struct sockaddr_in cliaddr;
		socklen_t cliaddrlen;
		clifd = accept(listenfd ,(struct sockaddr*) &cliaddr ,&cliaddrlen);
		if(clifd == -1 )
			perror("accpet error");
		else{
			printf("accept a new client: %s:%d\n ", inet_ntoa(cliaddr.sin_addr),cliaddr.sin_port);
			add_event(epollfd ,clifd ,EPOLLIN);
		}
	}

	void do_read(int epollfd ,int fd ,char*buf){
		int nread ;
		nread = read (fd, buf ,MAXSIZE);
		if(nread == -1 ){
			perror("read error:");
			close(fd);
			delete_event(epollfd,fd ,EPOLLIN);
		}else if(nread == 0 ){
			fprintf(stderr ,"client close.\n");
			close(fd);
			delete_event(epollfd, fd, EPOLLIN);
		}else{
			printf("read message is :%s",buf);
			modify_event(epollfd, fd ,EPOLLOUT);
		}
	}

	void do_write(int epollfd ,int fd ,char*buf){
		int nwrite ;
		nwrite = write(fd, buf ,strlen(buf));
		if (nwrite ==-1 ){
			perror("write error:");
			close (fd);
			delete_event(epollfd , fd ,EPOLLOUT);
		}else{
			modify_event(epollfd ,fd ,EPOLLIN);
		}
		memset(buf, 0 , MAXSIZE);
	}

	void add_event(int epollfd , int  fd ,int state ){
		struct epoll_event ev ;
		ev.events = state ;
		ev.data.fd = fd ;
		epoll_ctl(epollfd, EPOLL_CTL_ADD ,fd, &ev);
	}

	void delete_event(int epollfd ,int fd ,int state ){
		struct epoll_event ev;
		ev.events = state ;
		ev.data.fd = fd ;
		epoll_ctl(epollfd ,EPOLL_CTL_DEL,fd,&ev);
	}

	void modify_event(int epollfd ,int fd ,int state){
		struct epoll_event ev;
		ev.events = state ;
		ev.data.fd = fd ;
		epoll_ctl(epollfd ,EPOLL_CTL_MOD, fd, &ev);
	}



## 3.客户端代码

	#include <netinet/in.h>
	#include <sys/socket.h>
	#include <stdio.h>
	#include <string.h>
	#include <stdlib.h>

	#include <sys/epoll.h>
	#include <time.h>
	#include <unistd.h>
	#include <sys/types.h>
	#include <arpa/inet.h>

	#define MAXSIZE 1024 
	#define IPADDRESS  "127.0.0.1"
	#define SERV_PORT 6666
	#define FDSIZE 1024
	#define EPOLLEVENTS 20

	void handle_connection(int sockfd);
	void handle_events(int epollfd ,struct epoll_event *events,int num ,int sockfd ,char*buf);
	void do_read(int epollfd , int fd ,int sockfd, char* buf);
	void do_write(int epollfd ,int fd ,int sockfd ,char* buf);
	void add_event(int epollfd ,int fd ,int state );
	void delete_event(int epollfd, int fd ,int state );
	void modify_event(int epollfd ,int fd ,int state );
	int count = 0 ;

	int main( int argc ,char* argv[]){
		int sockfd;
		struct sockaddr_in servaddr;
		sockfd = socket(AF_INET , SOCK_STREAM, 0 );
		bzero(&servaddr , sizeof (servaddr));
		servaddr.sin_family = AF_INET;
		servaddr.sin_port = htons(SERV_PORT);
		inet_pton(AF_INET, IPADDRESS , &servaddr.sin_addr);
		if ( 0 !=  connect(sockfd, (struct sockaddr*)&servaddr ,sizeof(servaddr)) ){
			printf("connect %s-%d faild \n", IPADDRESS, SERV_PORT);
			close (sockfd);
			return -1;
		}
		handle_connection(sockfd);
		close(sockfd);
		return 0 ;
	}

	void handle_connection(int sockfd){
		int epollfd ;
		struct epoll_event events[EPOLLEVENTS];
		char buf[MAXSIZE];
		int ret;
		epollfd = epoll_create(FDSIZE);
		add_event(epollfd ,STDIN_FILENO,EPOLLIN);
		while(1){
			ret = epoll_wait(epollfd, events ,EPOLLEVENTS ,-1);
			handle_events(epollfd ,events ,ret, sockfd ,buf);
		}
		close(epollfd);
	}


	void handle_events(int epollfd ,struct epoll_event *events ,int num ,int sockfd, char*buf){
		int fd ,i ;
		for( i = 0 ; i < num ;i ++){
			fd = events[i].data.fd;
			if(events[i].events & EPOLLIN)
				do_read(epollfd ,fd ,sockfd ,buf);
			else if( events[i].events & EPOLLOUT)
				do_write(epollfd ,fd ,sockfd ,buf);
		}
	}

	void do_read(int epollfd ,int fd ,int sockfd ,char*buf){
		int nread ;
		nread = read(fd, buf ,MAXSIZE);
		if(nread == -1 ){
			perror("read error:");
			close(fd);
		}else if(nread == 0 ){
			fprintf(stderr, "server close \n");
			close (fd);
		}else{
			if (fd == STDIN_FILENO) // STDIN_FILENO：接收键盘的输入
				add_event(epollfd ,sockfd ,EPOLLOUT);
			else{ //从服务器读来的数据 删除读，打开写
				delete_event(epollfd, sockfd ,EPOLLIN);
				add_event(epollfd ,STDOUT_FILENO, EPOLLOUT); //往屏幕输出
			}
		}
	}


	void do_write(int epollfd ,int fd ,int sockfd ,char*buf){
		int nwrite ;
		char temp[100];
		buf[strlen(buf) -1 ] = '\0';
		snprintf(temp, sizeof(temp), "%s_%02d\n",buf ,count++);
		nwrite = write(fd ,temp ,strlen(temp));

		if(nwrite == -1 ){
			perror("write error: ");
			close(fd);
		}else{
			if( fd == STDOUT_FILENO)//STDOUT_FILENO：向屏幕输出
				delete_event(epollfd, fd ,EPOLLOUT);
			else
				modify_event(epollfd ,fd, EPOLLIN);
		}
		memset(buf, 0 ,sizeof(buf));
	}

	void add_event(int epollfd ,int fd ,int state){
		struct epoll_event ev;
		ev.events = state ;
		ev.data.fd = fd ;
		epoll_ctl(epollfd , EPOLL_CTL_ADD ,fd ,&ev);
	}


	void delete_event(int epollfd ,int fd ,int state )
	{
		struct epoll_event ev ;
		ev.events = state ;
		ev.data.fd  = fd ;
		epoll_ctl(epollfd ,EPOLL_CTL_DEL, fd, &ev);

	}

	void modify_event(int epollfd ,int fd ,int state ){
		struct epoll_event ev ;
		ev.events = state ;
		ev.data.fd = fd ;
		epoll_ctl(epollfd, EPOLL_CTL_MOD ,fd, &ev);
	}