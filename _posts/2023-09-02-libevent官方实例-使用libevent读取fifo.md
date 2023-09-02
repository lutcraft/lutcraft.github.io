---
layout: post
title: libevent——读取fifo
categories: reactor libevent RPC 多进程 信号量
description: 
keywords: 
---

## libevent

测试了libevent的一个官方示例代码，这个代码的服务器采用epoll后端，监控了`SIGINT`信号和一个有名fifo上的事件。自己编写了一个客户端，循环地向管道写入数据，测试libevnet。

因为管道是基于流的，且在后端被设置为非block的，这导致在libevent进行回调的过程中，客户端可能又写入了数据，导致服务器获得多条数据，甚至半截数据。所以改进了示例程序和客户端，增加了PV信号量来控制读写。

此外还有两个小细节需要注意：
1. 在epoll的回调中，必须进行read操作读取epoll监控的hd，且必须成功（read返回>0）否则，这个hd会保持在就绪态，被epoll不断触发，造成cpu冲高。
2. 在关闭socket时，描述符hd也会收到一个RV_READ，只是这个event的数据长度为0. 在hd回调中，需要进行特殊处理。在示例代码中，我们在发现socket被关闭时，将这个event从后端的监控列表中清除了出去，并且停止了后端的event_loop.


但还有一点一问，就是在libevent的回调中，无法获取在main中初始化的信号量指针semi_writer，直接取总是null。这个还需要后续研究研究。

## libevent后端
```
/*
 * This sample code shows how to use Libevent to read from a named pipe.
 * XXX This code could make better use of the Libevent interfaces.
 *
 * XXX This does not work on Windows; ignore everything inside the _WIN32 block.
 *
 * On UNIX, compile with:
 * cc -I/usr/local/include -o event-read-fifo event-read-fifo.c \
 *     -L/usr/local/lib -levent
 * 
 * gcc -I/usr/local/include -g -o event-read-fifo event-read-fifo.c -L/usr/local/lib -levent -lpthread
 * 
 */

#include <event2/event-config.h>

#include <sys/types.h>
#include <sys/stat.h>
#ifndef _WIN32
#include <sys/queue.h>
#include <unistd.h>
#include <sys/time.h>
#include <signal.h>
#else
#include <winsock2.h>
#include <windows.h>
#endif
#include <fcntl.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <semaphore.h>

#include <event2/event.h>

#ifdef __cplusplus
extern "C" {
#endif


sem_t *semi_writer;
sem_t *semi_reader;

//无论是否阻塞，都可能收到半截的数据，简单缓冲器就是这样的
static void
fifo_read(evutil_socket_t fd, short event, void *arg)
{

	char buf[255];
	int len;
	struct event *ev = arg;
#ifdef _WIN32
	DWORD dwBytesRead;
#endif
	// //看看属性 		//这个调用拖延了时间，导致read有时间可以完成
	// int flags = fcntl(fd, F_GETFL, 0);
	// if (flags & O_NONBLOCK)
	// {
	// 	fprintf(stdout, "No Block hd!\n");
	// }else
	// {
	// 	fprintf(stdout, "Block hd!\n");
	// }
	// fprintf(stderr, "fifo_read called with fd: %d, event: %d, arg: %p\n",
	//     (int)fd, event, arg);
#ifdef _WIN32
	len = ReadFile((HANDLE)fd, buf, sizeof(buf) - 1, &dwBytesRead, NULL);

	/* Check for end of file. */
	if (len && dwBytesRead == 0) {
		fprintf(stderr, "End Of File");
		event_del(ev);
		return;
	}

	buf[dwBytesRead] = '\0';
#else
	fprintf(stdout, "get alarm!\n");
    sem_t *semi_writer = sem_open("semi_writer", 0);
    sem_t *semi_reader = sem_open("semi_reader", 0);
        printf("semi_reader:%p\n", semi_reader);

        int a = 111;
        sem_getvalue(semi_writer,&a);
        printf("semi_writer:%d\n", a);

	//V-semi2 等待可读
    sem_wait(semi_reader);
	sleep(2);
	//如果read到的是空的，则读无法完成，epoll_wait会不断返回
	len = read(fd, buf, sizeof(buf) - 1);
	//P-semi1
    sem_post(semi_writer);
	fprintf(stdout, "read length:%u\n", len);
	if (len <= 0) {
		if (len == -1)
		{
			perror("read -1 err!");
		}
		else if (len == 0)
		{
			if (dup2(fd,fd)<0)
			{
				fprintf(stderr, "Close!\n");
			}
			else
			{
				fprintf(stderr, "not close!!!\n");
			}
			
		}
		fprintf(stderr, "del!!!\n");
		event_del(ev);
		event_base_loopbreak(event_get_base(ev));
		return;
	}

	buf[len] = '\0';
#endif
	// fprintf(stdout, "Read: %s\n", buf);
	for (int i = 0; i < len; i++)
	{
		putchar(buf[i]);
	}
	
}

/* On Unix, cleanup event.fifo if SIGINT is received. */
#ifndef _WIN32
static void
signal_cb(evutil_socket_t fd, short event, void *arg)
{
	struct event_base *base = arg;
	event_base_loopbreak(base);
}
#endif

int
main(int argc, char **argv)
{
	sem_unlink("semi_writer");	//需要销毁
	sem_unlink("semi_reader");
    sem_t *semi_writer = sem_open("semi_writer", O_CREAT, 0666, 1);
    sem_t *semi_reader = sem_open("semi_reader", O_CREAT, 0666, 0);
        int a = 111;
        sem_getvalue(semi_writer,&a);
        printf("main semi_writer:%d\n", a);

	struct event *evfifo;
	struct event_base* base;
#ifdef _WIN32
	HANDLE socket;
	/* Open a file. */
	socket = CreateFileA("test.txt",	/* open File */
			GENERIC_READ,		/* open for reading */
			0,			/* do not share */
			NULL,			/* no security */
			OPEN_EXISTING,		/* existing file only */
			FILE_ATTRIBUTE_NORMAL,	/* normal file */
			NULL);			/* no attr. template */

	if (socket == INVALID_HANDLE_VALUE)
		return 1;

#else
	struct event *signal_int;
	struct stat st;
	const char *fifo = "event.fifo";
	int socket;

	if (lstat(fifo, &st) == 0) {
		if ((st.st_mode & S_IFMT) == S_IFREG) {
			errno = EEXIST;
			perror("lstat");
			exit(1);
		}
	}

	unlink(fifo);
	if (mkfifo(fifo, 0600) == -1) {
		perror("mkfifo");
		exit(1);
	}

	// socket = open(fifo, O_RDONLY, 0);	//阻塞io没问题，会等到fifo被写好
	socket = open(fifo, O_RDONLY|O_NONBLOCK, 0);

	if (socket == -1) {
		perror("open");
		exit(1);
	}

	fprintf(stderr, "Write data to %s\n", fifo);
#endif
	/* Initialize the event library */
	base = event_base_new();

	/* Initialize one event */
#ifdef _WIN32
	evfifo = event_new(base, (evutil_socket_t)socket, EV_READ|EV_PERSIST, fifo_read,
                           event_self_cbarg());
#else
	/* catch SIGINT so that event.fifo can be cleaned up */
	signal_int = evsignal_new(base, SIGINT, signal_cb, base);
	event_add(signal_int, NULL);

	evfifo = event_new(base, socket, EV_READ|EV_PERSIST, fifo_read,
                           event_self_cbarg());
#endif

	/* Add it to the active events, without a timeout */
	event_add(evfifo, NULL);



	event_base_dispatch(base);
	event_base_free(base);
#ifdef _WIN32
	CloseHandle(socket);
#else
	close(socket);
	unlink(fifo);
	sem_destroy(semi_writer);
	sem_destroy(semi_reader);
#endif
	libevent_global_shutdown();
	return (0);
}

#ifdef __cplusplus
}
#endif
```

## 客户端

```

//gcc fifowrite.c -g -o fifowrite  -lpthread -levent

#include <stdio.h>                                                             
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <limits.h>
#include <stdlib.h>
#include <time.h>
#include <semaphore.h>
 
#define BUFS PIPE_BUF
 
int main()
{
    int a;
    int fd;
    int n, i;
    char buf[BUFS];
    time_t tp;
    sem_t *semi_writer = sem_open("semi_writer", 0);
    sem_t *semi_reader = sem_open("semi_reader", 0);

    sem_getvalue(semi_writer,&a);
    printf("semi_writer:%d\n", a);

    printf("I am %d\n", getpid());
    if (access("event.fifo", F_OK) != 0)
    {
        // if (mkfifo("event.fifo", 0666))
        // {
        //     exit(1);
        // }

        printf("event.fifo not existed, and create event.fifo failed!\n");
    }
    if ((fd = open("event.fifo", O_WRONLY)) < 0)
    {
        printf("Open failed!\n");
        exit(1);
    }
    int count = 0;
    for (i = 0; i < 50; i++)
    {
        time(&tp);
        n = sprintf(buf, "c%d write_fifo %d sends %s", count++, getpid(), ctime(&tp));
        printf("Send msg:%s", buf);
        //V-semi1

        sleep(1);
        sem_wait(semi_writer);
        if ((write(fd, buf, n + 1)) < 0)
        {
            printf("Write failed!\n");
            close(fd);
            exit(1);
        }
        sleep(1);
        //P-semi2
        sem_post(semi_reader);
    }
    //客户端在这里关闭fd的时候，服务器也会收到一个读事件
    close(fd);
    sem_close(semi_reader);
    sem_close(semi_writer);
    exit(0);
}

```