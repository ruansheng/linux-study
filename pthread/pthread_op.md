## thread 操作
线程可以由主线程创建，也可以由被创建出来的子线程再创建，但是一个进程中的所有线程都是同等的关系，线程没有父子之分，这也大大简化了线程的处理

### 线程的操作函数
| POSIX函数        | 功能描述 |
 | :--------   | :-----|
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
const pthread_attr_t *attr 用来设置线程属性，比如新建线程栈的大小，调度策略等，NULL表示默认
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

### pthread_self 和 SYS_gettid 的区别
```
pthread_self()是POSIX的实现，它的返回值是pthread_t，pthread_t在linux中实际是无符号长整型，即unsigned long。
gettid是系统调用，它的返回值是pid_t，在linux上是一个无符号整型。
pthread_self是为了区分同一进程种不同的线程, 是由thread的实现来决定的，而gettid获取的线程id和pid是有关系的，因为在linux中线程其实也是一个进程(clone)，所以它的线程ID也是pid_t类型。在一个进程中，主线程的线程id和进程id是一样的，该进程中其他的线程id则在linux系统内是唯一的，因为linux中线程就是进程，而进程号是唯一的。gettid是不可移植的。

pthread_self返回的是同一个进程中各个线程之间的标识号，对于这个进程内是唯一的，而不同进程中，每个线程返回的pthread_self可能返回的是一样的。而gettid是用来系统内各个线程间的标识符，由于linux采用轻量级进程实现的，它其实返回的应该是pid号。
```

### 给线程取名字
```
#include <sys/prctl.h>  
int prctl(int option, unsigned long arg2, unsigned long arg3, unsigned long arg4, unsigned long arg5);

对于"option"的选项有很多，我们这里只关注的是"PR_SET_NAME"这个选项
当入参"option"值为"PR_SET_NAME"时，prctl()会为 调用此函数的 线程设置新名字，新名字设置为参数(char *)arg2。其中，线程名字的参数arg2字符串的最大长度为16 bytes(包括字符串终止符)。如果arg2长度超过16 bytes，它将会被截断为16 bytes

除了使用“PR_SET_NAME”参数可以设置线程名字外，另外一个函数pthread_setname_np()也可以实现此功能。设置完成后我们可以通过线程的tid来查看它的新名字
```

### 线程名称的操作
```
#include<stdio.h>
#include<pthread.h>
#include<sys/prctl.h>

void *tmain(void *arg)
{
    char name[32];
    prctl(PR_SET_NAME, (unsigned long)"pthread-1");
    prctl(PR_GET_NAME, (unsigned long)name);
    printf("thread name:%s\n", name);
    while(1)
    {
	sleep(1);
    }
}

int main(void)
{
    pthread_t tid;
    pthread_create(&tid, NULL, tmain, NULL);
    pthread_join(tid, NULL);

    return 0;
}

#gcc -o mypthread mypthread.c -lpthread
#ps -L -p 17816
  PID   LWP TTY          TIME CMD
17816 17816 pts/1    00:00:00 mypthread
17816 17817 pts/1    00:00:00 pthread-1
```

### 设置线程属性
```
pthread_create 的第二个参数是 const pthread_attr_t *attr 
pthread_attr_init()函数可以重置线程的属性为默认值
使用示例:
pthread_attr_t attr
pthread_attr_init(&attr)
```

### 线程栈
```
查看线程栈大小
# ulimit -s
8192

设置stack大小
#ulimit –s value

#include<pthread.h>
// 返回线程的基地址和栈的大小
int pthread_attr_getstacksize(pthread_attr_t *attr, size_t *stacksize); 

// 设置线程的基地址和栈的大小
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize)
```

### 线程退出
```
退出当前线程
#include  <pthread.h>
void  pthread_exit(void  *retval)
其中void  *retval是线程退出的传递参数，不传参数可以传NULL，return返回值也可作为传递的参数
可以通过pthread_join取出返回值，注意这个参数不要用函数栈内数据
```