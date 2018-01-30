## 进程的状态
| 进程状态        | 说明 |
 | :--------   | :-----|
| TASK_RUNNING   | 可运行状态，但未必正在使用CPU，也许在等待调度  |
| TASK_INTERRUPTIBLE | 可中断的睡眠状态，在等待某个条件的完成  |
| TASK_UNINTERRUPTIBLE | 不可中断的睡眠状态，与可中断的睡眠类似，但是不会被信号中断 |
| TASK_STOPED | 暂停状态，进程收到某信号，运行被停止 |
| TASK_TRACED | 被跟踪状态，和暂停状态有些类似，进程被停止，被另一个进程跟踪  |
| EXIT_ZOMBIE | 僵尸状态，进程已已经退出，但是尚未被父进程或init进程收养  |
| EXIT_DEAD | 真正死亡的状态，进程停留在该状态的时间很短，很难被观察到 |

### 可运行状态 TASK_RUNNING
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

### 可中断睡眠状态和不可中断睡眠状态
```
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
```