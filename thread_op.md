## thread 操作
线程可以由主线程创建，也可以由被创建出来的子线程再创建，但是一个进程中的所有线程都是同等的关系，线程没有父子之分，这也大大简化了线程的处理

### 线程的操作函数
| POSIX函数        | 功能描述 |
| pthread_create   | 创建一个线程  |
| pthread_exit        | 退出线程    |
| pthread_self        | 获取线程ID   |
| pthread_equal | 检查2个线程id是否相等  |
| pthread_join | 等待线程的退出  |
| pthread_detach | 设置线程状态为分离状态  |
| pthread_cancel | 线程的取消 |
| pthread_cleanup_push | 线程退出，清理函数注册和执行  |
| pthread_cleanup_pop | 线程退出，清理函数注册和执行     |

### 线程示例
```
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

void *thread_function(void *arg)
{
    int i;
    for(i = 0; i < 20; i++)
    {
	printf("Thread says hi!\n");
	sleep(1);
    }

    return NULL;
}

int main(void)
{
    pthread_t mythread;
    if(pthread_create(&mythread, NULL, thread_function, NULL))
    {
	printf("error creating thread.");
	abort();
    }
    if(pthread_join(mythread, NULL))
    {
	printf("error joining thread.");
	abort();
    }
    exit(0);
}
```

### pthread_create()
```
#include<pthread.h>
int pthread_create(pthread_t *tidp,const pthread_attr_t *attr,
(void*)(*start_rtn)(void*),void *arg);

参数说明:
pthread_t *tidp 线程id，也就是线程句柄
const pthread_attr_t *attr 用来设置线程属性
(void*)(*start_rtn)(void*) 函数指针，线程运行函数的起始地址
void *arg 运行函数的参数

返回值:
若线程创建成功，则返回0。若线程创建失败，则返回出错编号，并且*thread中的内容是未定义的

编译链接参数:
-lpthread

注意:
因为pthread并非Linux系统的默认库，而是POSIX线程库。在Linux中将其作为一个库来使用，因此加上 -lpthread（或-pthread）以显式链接该库。函数在执行错误时的错误信息将作为返回值返回，并不修改系统全局变量errno，当然也无法使用perror()打印错误信息。
```

### pthread_join()
```
#include <pthread.h>
int pthread_join(pthread_t thread, void **retval);

功能描述:
以阻塞的方式等待thread指定的线程结束。当函数返回时，被等待线程的资源被收回。如果线程已经结束，那么该函数会立即返回。并且thread指定的线程必须是joinable的。

参数说明:
pthread_t thread 线程id，也就是线程句柄
void **retval 用户定义的指针，用来存储被等待线程的返回值

返回值:
0代表成功。 失败，返回的则是错误号。
```

### 线程ID
```
linux提供了gettid系统调用来返回线程ID，但是glibc并没有将该系统调用封装起来，如果要获取线程ID，可以采用:
#include<sys/syscall.h>
int tid = syscall(SYS_gettid);

使用如下:
void *thread_function(void *arg)
{
    int i;
    for(i = 0; i < 20; i++)
    {
	int tid = syscall(SYS_gettid);
	printf("Thread id:%d says hi!\n", tid);
	sleep(1);
    }

    return NULL;
}
```