## pthread_cancel
linux提供了pthread_cancel函数来控制取消线程
一个线程调用该方法向另一个线程发送取消请求，该函数不是阻塞的，调用完成立即返回
对于glibc而言，调用pthread_cancel会向目标线程发送一个SIGCANCEL信号，这个信号就是被NPTL征用的32号信箱
(不建议使用这个函数)

### pthread_cancel的弊端
```
线程取消是一种外部的强行终止线程的做法，由于无法预知目标线程的内部情况，
可能互斥量没有解锁，内存没有释放 可能带来毁灭性的结果
```

### 取消线程
```
int pthread_setcancelstate(int state,   int *oldstate)
设置本线程对Cancel信号的反应，state有两种值：PTHREAD_CANCEL_ENABLE（缺省）和PTHREAD_CANCEL_DISABLE，
分别表示收到信号后设为CANCLED状态和忽略CANCEL信号继续运行；old_state如果不为NULL则存入原来的Cancel状态以便恢复。

int pthread_setcanceltype(int type, int *oldtype)   
设置本线程取消动作的执行时机，type由两种取值：PTHREAD_CANCEL_DEFFERED和PTHREAD_CANCEL_ASYCHRONOUS，仅当Cancel状态为Enable时有效，分别表示收到信号后继续运行至下一个取消点再退出和立即执行取消动作（退出）；oldtype如果不为NULL则存入运来的取消动作类型值。  

void pthread_testcancel(void)
是说pthread_testcancel在不包含取消点，但是又需要取消点的地方创建一个取消点，以便在一个没有包含取消点的执行代码线程中响应取消请求.
线程取消功能处于启用状态且取消状态设置为延迟状态时，pthread_testcancel()函数有效。
如果在取消功能处处于禁用状态下调用pthread_testcancel()，则该函数不起作用。
请务必仅在线程取消线程操作安全的序列中插入pthread_testcancel()。除通过pthread_testcancel()调用以编程方式建立的取消点意外，pthread标准还指定了几个取消点。测试退出点,就是测试cancel信号.
```

### 线程取消点
```
线程取消的方法是向目标线程发Cancel信号，但如何处理Cancel信号则由目标线程自己决定，或者忽略、或者立即终止、或者继续运行至Cancelation-point（取消点），由不同的Cancelation状态决定。
线程接收到CANCEL信号的缺省处理（即pthread_create()创建线程的缺省状态）是继续运行至取消点，也就是说设置一个CANCELED状态，线程继续运行，只有运行至Cancelation-point的时候才会退出。

pthreads标准指定了几个取消点，其中包括：
(1)通过pthread_testcancel调用以编程方式建立线程取消点。 
(2)线程等待pthread_cond_wait或pthread_cond_timewait()中的特定条件。 
(3)被sigwait(2)阻塞的函数 
(4)一些标准的库调用。通常，这些调用包括线程可基于阻塞的函数。 

缺省情况下，将启用取消功能。有时，您可能希望应用程序禁用取消功能。如果禁用取消功能，则会导致延迟所有的取消请求，
直到再次启用取消请求。  
根据POSIX标准，pthread_join()、pthread_testcancel()、pthread_cond_wait()、pthread_cond_timedwait()、sem_wait()、sigwait()等函数以及
read()、write()等会引起阻塞的系统调用都是Cancelation-point，而其他pthread函数都不会引起Cancelation动作。
```