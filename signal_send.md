## signal 发送
发送信号的主要函数有：kill()、raise()、 sigqueue()、alarm()、setitimer()以及abort()

### kill()
```
#include <sys/types.h>
#include <signal.h>
int kill(pid_t pid,int signo)

该系统调用可以用来向任何进程或进程组发送任何信号。参数pid的值为信号的接收进程
pid>0 进程ID为pid的进程
pid=0 同一个进程组的进程
pid<0 pid!=-1 进程组ID为 -pid的所有进程
pid=-1 除发送进程自身外，所有进程ID大于1的进程

sinno是信号值，当为0时（即空信号），实际不发送任何信号，但照常进行错误检查，因此，可用于检查目标进程是否存在，以及当前进程是否具有向目标发送信号的权限（root权限的进程可以向任何进程发送信号，非root权限的进程只能向属于同一个session或者同一个用户的进程发送信号）。

Kill()最常用于pid>0时的信号发送。该调用执行成功时，返回值为0；错误时，返回-1，并设置相应的错误代码errno。下面是一些可能返回的错误代码：
EINVAL：指定的信号sig无效。
ESRCH：参数pid指定的进程或进程组不存在。注意，在进程表项中存在的进程，可能是一个还没有被wait收回，但已经终止执行的僵死进程。
EPERM： 进程没有权力将这个信号发送到指定接收信号的进程。因为，一个进程被允许将信号发送到进程pid时，必须拥有root权力，或者是发出调用的进程的UID 或EUID与指定接收的进程的UID或保存用户ID（savedset-user-ID）相同。如果参数pid小于-1，即该信号发送给一个组，则该错误表示组中有成员进程不能接收该信号。
```

### raise()
```
#include <signal.h>
int raise(int signo)

向进程本身发送信号，参数为即将发送的信号值。调用成功返回 0；否则，返回 -1。
```

### sigqueue()
```
#include <sys/types.h>
#include <signal.h>
int sigqueue(pid_t pid, int sig, const union sigval val)
调用成功返回 0；否则，返回 -1

sigqueue()是比较新的发送信号系统调用，主要是针对实时信号提出的（当然也支持前32种），支持信号带有参数，与函数sigaction()配合使用。
sigqueue的第一个参数是指定接收信号的进程ID，第二个参数确定即将发送的信号，第三个参数是一个联合数据结构union sigval，指定了信号传递的参数，即通常所说的4字节值。

typedef union sigval {
    int  sival_int;
    void *sival_ptr;
}sigval_t;

sigqueue()比kill()传递了更多的附加信息，但sigqueue()只能向一个进程发送信号，而不能发送信号给一个进程组。如果signo=0，将会执行错误检查，但实际上不发送任何信号，0值信号可用于检查pid的有效性以及当前进程是否有权限向目标进程发送信号。
在调用sigqueue时，sigval_t指定的信息会拷贝到对应sig 注册的3参数信号处理函数的siginfo_t结构中，这样信号处理函数就可以处理这些信息了。由于sigqueue系统调用支持发送带参数信号，所以比kill()系统调用的功能要灵活和强大得多。
```

### alarm()
```
#include <unistd.h>
unsigned int alarm(unsigned int seconds)

系统调用alarm安排内核为调用进程在指定的seconds秒后发出一个SIGALRM的信号。如果指定的参数seconds为0，则不再发送 SIGALRM信号。后一次设定将取消前一次的设定。该调用返回值为上次定时调用到发送之间剩余的时间，或者因为没有前一次定时调用而返回0。

注意，在使用时，alarm只设定为发送一次信号，如果要多次发送，就要多次使用alarm调用
```

### setitimer()
```
现在的系统中很多程序不再使用alarm调用，而是使用setitimer调用来设置定时器，用getitimer来得到定时器的状态，这两个调用的声明格式如下：

int getitimer(int which, struct itimerval *value);
int setitimer(int which, const struct itimerval *value, struct itimerval *ovalue);

在使用这两个调用的进程中加入以下头文件：
#include <sys/time.h>

该系统调用给进程提供了三个定时器，它们各自有其独有的计时域，当其中任何一个到达，就发送一个相应的信号给进程，并使得计时器重新开始。三个计时器由参数which指定，如下所示：
TIMER_REAL：按实际时间计时，计时到达将给进程发送SIGALRM信号。
ITIMER_VIRTUAL：仅当进程执行时才进行计时。计时到达将发送SIGVTALRM信号给进程。
ITIMER_PROF：当进程执行时和系统为该进程执行动作时都计时。与ITIMER_VIR-TUAL是一对，该定时器经常用来统计进程在用户态和内核态花费的时间。计时到达将发送SIGPROF信号给进程

定时器中的参数value用来指明定时器的时间，其结构如下：
struct itimerval {
     struct timeval it_interval; /* 下一次的取值 */
     struct timeval it_value; /* 本次的设定值 */
};

该结构中timeval结构定义如下：
struct timeval {
     long tv_sec; /* 秒 */
     long tv_usec; /* 微秒，1秒 = 1000000 微秒*/
};

在setitimer 调用中，参数ovalue如果不为空，则其中保留的是上次调用设定的值。定时器将it_value递减到0时，产生一个信号，并将it_value的值设定为it_interval的值，然后重新开始计时，如此往复。当it_value设定为0时，计时器停止，或者当它计时到期，而it_interval 为0时停止。调用成功时，返回0；错误时，返回-1，并设置相应的错误代码errno：
EFAULT：参数value或ovalue是无效的指针。
EINVAL：参数which不是ITIMER_REAL、ITIMER_VIRT或ITIMER_PROF中的一个。
```

### abort()
```
#include <stdlib.h>
void abort(void);

向进程发送SIGABORT信号，默认情况下进程会异常退出，当然可定义自己的信号处理函数。即使SIGABORT被进程设置为阻塞信号，调用abort()后，SIGABORT仍然能被进程接收。该函数无返回值。
```
