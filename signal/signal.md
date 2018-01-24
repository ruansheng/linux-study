## signal 分析
linux信号继承自unix信号，unix信号是不可靠信号，后面linux新增了可靠信息，信号的可靠与不可靠只与信号值有关，与信号的发送及安装函数无关
不可靠信号:内核用位图来记录，内核可能丢弃，因为内核投递信息到进程的时候判断该信号是否是未决状态，就会丢弃该信号
可靠信号:内核内部用队列来维护，收到信号进挂到相应的队列中，但是挂起的信号个数也是有限制的

### linux查看所有信号
```
kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX

1-31属于不可靠信息
SIGRTMIN-SIGRTMAX 属于可靠信号

32、33 信号被NPTL这个线程库征用
```

### 实时信号(可靠信号)的限制
```
ulimit -a 查看实时信号最多挂载量
...
pending signals                 (-i) 7277
...
```

### 触发信号
```
1. 硬件触发(比如我们按下了键盘或者其它硬件故障)
2. 软件触发(最常用发送信号的系统函数是kill, raise, alarm和setitimer以及sigqueue函数，软件来源还包括一些非法运算等操作)
```

### signal函数
```
#include<signal.h>
typedef void (*sighandler_t)(int);
__sighandler_t signal (int sig, __sighandler_t handler)
{
  __set_errno (ENOSYS);
  return SIG_ERR;
}
```

### 演化的signal 函数
```
#include<signal.h>
typedef void (*sighandler_t)(int);
__sighandler_t sysv_signal (int __sig, __sighandler_t __handler) // System V 风格的 signal函数
__sighandler_t bsd_signal (int __sig, __sighandler_t __handler)  // BSD风格的 signal函数
```

### 四类signal函数
```
syscall(SYS_signal, signo, func)  // signal系统调用
signal()                          // glibc的signal函数
sysv_signal()                     // System V风格的signal函数
bsd_signal()                      // BSD风格的signal函数

System V 风格的signal函数:每次处理完之后要想重新触发，必须再次安装信号处理函数
```

### 预定于宏
```
// glibc-2.26/bits/signal.h
# define NSIG	_NSIG

// glibc-2.26/bits/signum-generic.h
#define _NSIG		(__SIGRTMAX + 1)
#define __SIGRTMIN	32
#define __SIGRTMAX	__SIGRTMIN
```
