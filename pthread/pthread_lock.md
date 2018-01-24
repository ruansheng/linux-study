## pthread锁
NTPL提供了pthread_rwlock_t类型来表示读写锁，和互斥量一样，也提供了初始化方法

###初始化和销毁读写锁
```
pthread_rwlock_t  m_rw_lock;
pthread_rwlock_init(pthread_rwlock_t * ,pthread_rwattr_t *);
pthread_rwlock_t m_rw_lock = PTHREAD_RWLOCK_INITIALIZER;

pthread_rwlock_destroy(pthread_rwlock_t* );
```

### 读锁上锁
```
pthread_rwlock_rdlock(pthread_rwlock_t *);
pthread_rwlock_tryrdlock(pthread_rwlock_t *);
pthread_rwlock_timedrdlock(pthread_rwlock_t *, const struct timespec *abstime);
```

### 写锁上锁
```
pthread_rwlock_wrlock(pthread_rwlock_t *);
pthread_rwlock_trywrlock(pthread_rwlock_t *);
pthread_rwlock_timedwrlock(pthread_rwlock_t *, const struct timespec *abstime);
```

### 释放锁
```
无论是读锁还是写锁，锁的释放都是同一个接口:
pthread_rwlock_unlock(pthread_rwlock_t *);
```

### 读写锁的竞争策略
```
读写锁的属性是:pthread_rwlockattr_t类型
属性中有lockkind 和 pshared

int pthread_rwlockattr_setkind_np(pthread_rwlockattr_t *attr, int pref);
int pthread_rwlockattr_getkind_np(const pthread_rwlockattr_t *attr, int *pref);

enum
{
  PTHREAD_RWLOCK_PREFER_READER_NP,
  PTHREAD_RWLOCK_PREFER_WRITER_NP,
  PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP,
  PTHREAD_RWLOCK_DEFAULT_NP = PTHREAD_RWLOCK_PREFER_READER_NP
};


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