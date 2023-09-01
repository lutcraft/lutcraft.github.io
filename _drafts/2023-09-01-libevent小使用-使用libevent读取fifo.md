---
layout: post
title: 使用libevent读取fifo
categories: reactor libevent RPC
description: 
keywords: 
---


# 后端
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
 * gcc -I/usr/local/include -o event-read-fifo event-read-fifo.c -L/usr/local/lib -levent -lpthread
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


//无论是否阻塞，都可能收到半截的数据，简单缓冲器就是这样的
static void
fifo_read(evutil_socket_t fd, short event, void *arg)
{
    sem_t *semi_writer = sem_open("semi_writer", O_CREAT, 0666, 1);
    sem_t *semi_reader = sem_open("semi_reader", O_CREAT, 0666, 0);
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
	sleep(0);
	//V-semi2 等待可读
    sem_wait(semi_reader);
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
#endif
	libevent_global_shutdown();
	return (0);
}


```

# 客户端

```

//gcc fifowrite.c -levent -o fifowrite  -lpthread

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
    int fd;
    int n, i;
    char buf[BUFS];
    time_t tp;
    sem_t *semi_writer = sem_open("semi_writer", O_CREAT, 0666, 1);
    sem_t *semi_reader = sem_open("semi_reader", O_CREAT, 0666, 0);

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
        sem_wait(semi_writer);
        if ((write(fd, buf, n + 1)) < 0)
        {
            printf("Write failed!\n");
            close(fd);
            exit(1);
        }
        //P-semi2
        sem_post(semi_reader);
    }
    //客户端在这里关闭fd的时候，服务器也会收到一个读事件
    close(fd);
    sem_close(semi_writer);
    exit(0);
}

```