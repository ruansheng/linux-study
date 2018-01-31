## 进程的状态
| 进程状态        | 说明 |
 | :--------   | :-----|
| TASK_RUNNING   | 可运行状态，但未必正在使用CPU，也许在等待调度  |
| TASK_INTERRUPTIBLE | 可中断的睡眠状态，在等待某个条件的完成  |
| TASK_UNINTERRUPTIBLE | 不可中断的睡眠状态，与可中断的睡眠类似，但是不会被信号中断 |
| TASK_STOPPED | 暂停状态，进程收到某信号，运行被停止 |
| TASK_TRACED | 被跟踪状态，和暂停状态有些类似，进程被停止，被另一个进程跟踪  |
| EXIT_ZOMBIE | 僵尸状态，进程已已经退出，但是尚未被父进程或init进程收养  |
| EXIT_DEAD | 真正死亡的状态，进程停留在该状态的时间很短，很难被观察到 |
| TASK_KILLABLE | 内核2.6.25引入，和TASK_UNINTERRUPTIBLE类似，区别是可以响应致命信号SIGKILL |

### TASK_RUNNING状态(可运行状态)
```
这个描述不爱太准确，应该说是可运行状态，并非一定是在占有CPU运行，可能叫 TASK_RUNABLE会更好
处于这个状态的进程有2种情况:
1. 正在RUNNING中的(可能在执行用户态代码、也可能在执行内核态代码)
2. 处于READY中的，可随时投入运行，由于CPU资源有限，调度器暂时未选中它运行

time 命令可查看进程在用户态或内核态执行的时间

CPU利用率 = ((user time) + (sys time)) / (real time)

# cat /proc/27812/stat
27812 (phantomjs) S 27805 27805 26404 0 -1 1077944320 450504 0 0 0 4829 415 0 0 20 0 18 0 1071005045 2158919680 45848 18446744073709551615 4194304 70025304 140721656213040 140721656211728 140360911137961 0 0 16781312 1260 18446744073709551615 0 0 17 0 0 0 0 0 0 72122464 74221048 85569536 140721656218992 140721656219050 140721656219050 140721656221653 0

说明：以下只解释对我们计算Cpu使用率有用相关参数
参数                                 解释
pid=27812                            进程号
utime=415                        该任务在用户态运行的时间，单位为jiffies
stime=4829                       该任务在核心态运行的时间，单位为jiffies
cutime=0                          所有已死线程在用户态运行的时间，单位为jiffies
cstime=0                          所有已死在核心态运行的时间，单位为jiffies
进程的总Cpu时间processCpuTime = utime + stime + cutime + cstime，该值包括其所有线程的cpu时间

时钟滴答(clock tick):
#cat /boot/config-3.10.0-514.21.1.el7.x86_64  | grep CONFIG_HZ
# CONFIG_HZ_PERIODIC is not set
# CONFIG_HZ_100 is not set
# CONFIG_HZ_250 is not set
# CONFIG_HZ_300 is not set
CONFIG_HZ_1000=y
CONFIG_HZ=1000
表示1秒钟有1000个时钟滴答，增加一个时钟滴答(内核的jiffies++)
```

### TASK_INTERRUPTIBLE状态 和 TASK_INTERRUPTIBLE状态
```
(可中断睡眠状态和不可中断睡眠状态)

进程不总是处于运行状态，有些进程需要和慢速的设备交互，就需要消耗很长的时间
1. 比如进程和磁盘交互消耗的时间是很长的，这时候进程就需要等待这些耗时的操作完成之后才能执行接下来的指令
2. 有些进程需要等待某些特定条件(等待子进程退出、等待socket连接、尝试获得锁、等待信号量等)，满足条件才可以继续执行，往往等待的时间是不可估量的

上面的情况如果进程依旧占用CPU就不合适了，对CPU资源是极大的浪费，这时候内核就会调整进程的状态，把进程从CPU的运行队列中移除，同时CPU选择其他进程来使用CPU

可中断睡眠状态和不可中断睡眠状态的区别在于能否响应收到的信号

内核提供了hung task检测机制，启动一个名为khungtaskd的内核线程来检测处于TASK_UNINTERRUPTIBLE状态的进程是否已经失控，khungtaskd定期被唤醒(默认120秒)，khungtaskd会遍历所有处于TASK_UNINTERRUPTIBLE状态的进程
这里的120秒可以修改，查看:
# sysctl kernel.hung_task_timeout_secs
kernel.hung_task_timeout_secs = 120
关于khungtaskd的源码在kernel/hung_task.c中

查看不可中断状态的进程停在什么位置或等待什么资源:(procfs的wchan提供了这些信息，wchan是wait channel的意思，ps命令也可以通过wchan获得这些信息)
下面是查看当前bash正在等待子进程的退出:
# echo $$
2778
# cat /proc/2778/wchan
do_wait
# ps -p 2778 -o pid,wchan,cmd
PID WCHAN  CMD
2778 wait   -bash
# cat /proc/2778/stack
[<ffffffff8108baf3>] do_wait+0x1f3/0x250
[<ffffffff8108cbf0>] SyS_wait4+0x80/0x110
[<ffffffff816975c9>] system_call_fastpath+0x16/0x1b
[<ffffffffffffffff>] 0xffffffffffffffff

等待队列:睡眠进程是通过等待队列(wait queue)实现的
内核使用双向链表来实现等待队列，每个等待队列都可以用等待队列头来标识，等待队列头的定义如下:
// /include/linux/wait.h
struct __wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
解释:
spinlock_t lock 在对task_list与操作的过程中，使用该锁实现对等待队列的互斥访问
struct list_head task_list 双向循环链表，存放等待的进程
直接定义并初始化:
wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);
init_waitqueue_head()函数会将自旋锁初始化为未锁，等待队列初始化为空的双向循环链表
定义并初始化:
DECLARE_WAIT_QUEUE_HEAD(my_queue);

进程需要休眠的时候需要定义一个等待队列元素，将元素挂入合适的等待队列中，等待队列元素定义如下:
typedef struct __wait_queue wait_queue_t;
struct __wait_queue {
    unsigned int flags;
#define WQ_FLAG_EXCLUSIVE   0x01
    void *private;
    wait_queue_func_t func;
    struct list_head task_list;
};
其中flags域指明该等待的进程是互斥进程还是非互斥进程。其中0是非互斥进程，WQ_FLAG_EXCLUSIVE(0×01)是互斥进程
宏初始化:
DECLARE_WAITQUEUE(name,tsk);
此处是定义一个wait_queue_t类型的变量name，并将其private与设置为tsk
函数初始化:
static inline void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)
{
    q->flags = 0;
    q->private = p;
    q->func = default_wake_function;
}
static inline void init_waitqueue_func_entry(wait_queue_t *q,wait_queue_func_t func)
{
    q->flags = 0;
    q->private = NULL;
    q->func = func;
}

将等待队列元素添加到合适的等待队列:
设置等待的进程为非互斥进程，并将其添加进等待队列头(q)的队头中
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
    unsigned long flags;
    wait->flags &= ~WQ_FLAG_EXCLUSIVE;
    spin_lock_irqsave(&q->lock, flags);
    __add_wait_queue(q, wait);
    spin_unlock_irqrestore(&q->lock, flags);
}
该函数也和add_wait_queue()函数功能基本一样，只不过它是将等待的进程(wait)设置为互斥进程
void add_wait_queue_exclusive(wait_queue_head_t *q, wait_queue_t *wait)
{
    unsigned long flags;
    wait->flags |= WQ_FLAG_EXCLUSIVE;
    spin_lock_irqsave(&q->lock, flags);
    __add_wait_queue_tail(q, wait);
    spin_unlock_irqrestore(&q->lock, flags);
}

将等待队列元素从等待队列移除:
在等待的资源或事件满足时，进程被唤醒，使用该函数被从等待头中删除
void remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
    unsigned long flags;
    spin_lock_irqsave(&q->lock, flags);
    __remove_wait_queue(q, wait);
    spin_unlock_irqrestore(&q->lock, flags);
}

内核封装了一些宏来完成添加到等待队列功能:
wait_event(wq, condition);
wait_event_timeout(wq, condition, timeout);
wait_event_interruptible(wq, condition);
wait_event_interruptible_timeout(wq, condition, timeout);
wait_event_killabel(wq, condition);  // 添加为TASK_KILLABLE状态
第一个参数是等待队列头部，表示该进程会睡眠在该等待队列上

从等待队列上唤醒进程:
wake_up(x);
wake_up_nr(x, nr);
wake_up_all(x);
wake_up_interruptible(x);
wake_up_interruptible_nr(x, nr);
wake_up_interruptible_all(x);
```

### TASK_KILLABLE状态
```
内核2.6.25引入，和TASK_UNINTERRUPTIBLE类似，区别是可以响应致命信号SIGKILL
添加了一个宏来添加为TASK_KILLABLE状态，SIGKILL信号可以将其唤醒
wait_event_killabel(wq, condition);
```

### TASK_STOPPED状态 和 TASK_TRACED状态
```
TASK_STOPPED状态:
进程收到 SIGSTOP、SIGTSTP、SIGTTIN、SIGTTOU 信号都会将进程暂停，进程暂停就会进入该状态
进程收到 收到 SIGCONT 信号可以从 TASK_STOPPED状态 恢复

TASK_TRACED状态:
被跟踪状态，进程会停下来等待跟踪它的进程对它进行进一步的操作，例如gdb调试程序，当进程再断点处停下来时

这2个状态类似之处是都处于暂停状态，不同之处是TASK_TRACED状态不会被SIGCONT信号唤醒，只有调试进程通过ptrace系统调用，下达PTRACE_CONT、PTRACE_DETACH等指令，或者调试进程退出，被跟踪进程才能恢复TASK_RUNNING状态
```

### EXIT_ZOMBIE状态 和 EXIT_DEAD状态
```
EXIT_ZOMBIE状态 和 EXIT_DEAD状态 2种都是退出状态，处于这2种状态的进程都已经死了

如果子进程退出，父进程没有将SIGCHLD信号的处理函数重设为SIG_IGN，或者没有为SIGCHLD设置SA_NOCLDWAIT标志位，那么子进程退出后会进入EXIT_ZOMBIE状态等待父进程或init进程来收尸

进程的退出是非常快的，很难观察到一个进程处于EXIT_DEAD状态
```
