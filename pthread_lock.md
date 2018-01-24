## pthread互斥量
互斥量就是互相排斥之意，用来保护共享数据，保证操作是原子性的

### 

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
#define LOOP_TIMES 10000000
#define NR_THREAD 4
pthread_rwlock_t rwlock;
int global_cnt = 0;
#define ERRBUF_LEN 4096

void *thread_work(void *param)
{
    int i;
    pthread_rwlock_rdlock(&rwlock);
    for(i = 0; i < LOOP_TIMES; i++)
    {
	global_cnt++;
    }
    pthread_rwlock_unlock(&rwlock);
    return NULL;
}

int main(int argc, char *argv[])
{
    pthread_t tid[NR_THREAD];
    char errbuf[ERRBUF_LEN];
    int i,ret;
    ret = pthread_rwlock_init(&rwlock, NULL);
    if(ret)
    {
	fprintf(stderr, "init lock failed, return %d (%s)\n", ret, strerror_r(ret, errbuf, sizeof(errbuf)));
	exit(0);
    }

    pthread_rwlock_wrlock(&rwlock);

    for(i= 0; i < NR_THREAD; i++)
    {
	ret = pthread_create(&tid[i], NULL, thread_work, NULL);
	if(ret != 0)
	{
	    fprintf(stderr, "create thread failed, return %d (%s)\n", ret, strerror_r(ret, errbuf, sizeof(errbuf)));
	}
    }

    pthread_rwlock_unlock(&rwlock);

    for(i= 0; i < NR_THREAD; i++)
    {
	ret = pthread_join(tid[i], NULL);
    }

    pthread_rwlock_destroy(&rwlock);
    printf("thread num       :%d\n", NR_THREAD);
    printf("loop per thread  :%d\n", LOOP_TIMES);
    printf("expect resutl    :%d\n", LOOP_TIMES*NR_THREAD);
    printf("actual result    :%d\n", global_cnt);
    exit(0);
}
```

### i++ 剖析
```
反汇编操作:
#objdump -d pthread_lock > pthread_lock.objdump
#cat pthread_lock.objdump
  ...
  40098c:	8b 05 1a 07 20 00    	mov    0x20071a(%rip),%eax        # 6010ac <global_cnt>
  400992:	83 c0 01             	add    $0x1,%eax
  400995:	89 05 11 07 20 00    	mov    %eax,0x200711(%rip)        # 6010ac <global_cnt>
  ...

可以看出i++的操作并非是原子操作，而是对应3条汇编指令

```