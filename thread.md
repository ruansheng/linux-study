## thread 机制
POSIX threads标准确立于1995年，Unix原本不支持线程，引入线程给Unix带来了一些麻烦，因为很多函数都不是线程安全的(thread-safe)，需要重新定义

### Linux中线程的实现
目前Linux中线程的实现是Native POSIX Thread Library，简称NPTL，每一个用户态的线程，在内核中都对应一个调度实体，也拥有自己的进程描述符(task_struct结构体)
```
// /include/linux/sched.h
struct task_struct {
    ...
	pid_t				pid;
	pid_t				tgid;
    ...
    struct task_struct		*group_lead
    struct pid_link			pids[PIDTYPE_MAX];
	struct list_head		thread_group;
    ...
};

这里 pid其实是线程id，可通过gettid(void) 获取
这里 tgid其实是进程id，可通过getpid(void) 获取

linux中查看pid
#ps -eLf
UID        PID  PPID   LWP  C NLWP STIME TTY      STAT   TIME CMD
mysql     1280     1  1280  0   39  2017 ?        Sl     0:45 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
mysql     1280     1  1287  0   39  2017 ?        Sl     0:00 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
mysql     1280     1  1294  0   39  2017 ?        Sl     4:41 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
mysql     1280     1  1295  0   39  2017 ?        Sl     6:14 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
mysql     1280     1  1296  0   39  2017 ?        Sl     5:33 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
mysql     1280     1  1297  0   39  2017 ?        Sl     5:00 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
mysql     1280     1  1298  0   39  2017 ?        Sl     5:26 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

这里LWP就是gettid()系统调用的返回值
这里NLWP就是线程组内线程的个数

cd /proc/$PID/task/ 可以看到一个进程中的线程信息
```

### 进程与线程的关系
一个进程中可以创建多个线程，多个线程之间的关系的对等的，不想进程与子进程的关系，同一个进程内的多个线程共享进程的内存地址空间；而同一个进程的多个子进程则不能共享，虽然有许多不同种类的本地 IPC (进程间通信），但还是有一定的局限性

### 线程组
POSIX保准要求进程内的所有线程调用getpid()函数返回相同的进程ID，所以引入了线程组(Thread Group)的概念
内核在创建第一个线程(也就主线程)的时候，会把线程组ID的值设置成第一个线程的线程ID，group_leader指针指向自身，所以可以看到线程组内总有一个线程的ID等于进程ID，也就是主线程
通过group_leader指针，每个线程都能轻松的找到主线程，通过主线程也可以轻松的遍历组内的所有线程(通过链表)