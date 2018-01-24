## pthread_join
pthread_join()函数，以阻塞的方式等待thread指定的线程结束。当函数返回时，被等待线程的资源被收回。如果线程已经结束，那么该函数会立即返回。并且thread指定的线程必须是joinable的。

### 函数原型
```
#include <pthread.h>
int pthread_join(pthread_t thread, void **retval);
```

### 示例
```
#define _GNU_SOURCE
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>
#include<string.h>
#include<errno.h>
#include<sys/syscall.h>
#include<sys/types.h>
#define NR_THREAD 1
#define ERRBUF_LEN 4096

void *thread_work(void *param)
{
    int TID = syscall(SYS_gettid);
    printf("thread-%d IN \n", TID);
    printf("thread-%d pthread_self return %p \n", TID, (void*)pthread_self());
    sleep(60);
    printf("thread-%d EXIT \n", TID);
    return NULL;
}

int main(int argc, char *argv[])
{
    pthread_t tid[NR_THREAD];
    pthread_t tid_2[NR_THREAD];
    char errbuf[ERRBUF_LEN];
    int i,ret;
    for(i= 0; i < NR_THREAD; i++)
    {
	ret = pthread_create(&tid[i], NULL, thread_work, NULL);
	if(ret != 0)
	{
	    fprintf(stderr, "create thread failed, return %d (%s)\n", ret, strerror_r(ret, errbuf, sizeof(errbuf)));
	}
    }

#ifdef NO_JOIN
    sleep(30);
#else
    printf("join thread Begin\n");
    for(i = 0; i < NR_THREAD; i++)
    {
	pthread_join(tid[i], NULL);
    }
#endif

    for(i= 0; i < NR_THREAD; i++)
    {
	ret = pthread_create(&tid_2[i], NULL, thread_work, NULL);
	if(ret != 0)
	{
	    fprintf(stderr, "create thread failed, return %d (%s)\n", ret, strerror_r(ret, errbuf, sizeof(errbuf)));
	}
    }
    sleep(1000);
    exit(0);
}

编译:
gcc -o pthread_no_join 3.c -DNO_JOIN -lpthread
gcc -o pthread_has_join 3.c -lpthread

没有pthread_join的情况:
thread-22126 IN
thread-22126 pthread_self return 0x7f7dc0d65700
thread-22130 IN
thread-22130 pthread_self return 0x7f7dc0564700
thread-22126 EXIT
thread-22130 EXIT

有pthread_join的情况:
join thread Begin
thread-22164 IN
thread-22164 pthread_self return 0x7f3a6e8bd700
thread-22164 EXIT
thread-22170 IN
thread-22170 pthread_self return 0x7f3a6e8bd700
thread-22170 EXIT

由上可见:
未执行pthread_join的情况下，线程退出后，线程资源无法释放，线程栈空间出现了泄露
执行pthread_join的情况下，线程退出后，线程资源释放了，线程地址空间是被重复利用了
```