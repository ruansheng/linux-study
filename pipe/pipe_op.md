## 无名管道pipe
无名管道因为没有实体文件与之关联，靠的是时代相传的文件描述符，所以只能用在有共同祖先的各个进程之间

### 创建无名管道
```
在Linux下，可以使用如下接口来创建管道:
#include<unistd.h>
int pipe(int pipefd[2]);
成功: 返回 0
失败: 返回 -1 并设置errno
     EMFILE  该进程使用的文件描述符已已经多于MAX_OPEN-2
     ENFILE  系统中同时打开的文件已经超过了系统的限制
     EFAULT  pipefd参数不合法

成功调用pipe函数后，会返回2个打开的文件描述符:
pipefd[0] 读取端描述符
pipefd[1] 写入端描述符  

管道没有文件名与之关联，因此只能通过文件描述符来访问
```

### 管道写数据
```
write(pipefd[1], wbuf, count);
```

### 管道读数据
```
read(pipefd[0], rbuf, count);
当管道为空时，read会被阻塞(如果没有设置O_NONBLOCK标志位)
```

### 常规操作
```
父子进程通过管道通信，fork之后父子进程都同时拥有了pipefd[2]
可以父进程关闭写文件描述符，子进程关闭读文件描述符
int pipefd[2];
pipe(pipefd);
switch(fork()) {
    case -1:
        /* fork failed, error handler here */
    case 0: /* 子进程 */
        close(pipefd[0];
        break;
    default: /* 父进程 */
        close(pipefd[1];
        break;
}
```

### 管道的性质
```
1. 只有当所有的写入端描述符都已关闭，且管道中的数据都被读出，对读取端描述符调用read函数才会返回0(读到EOF标志)
2. 如果所有读取端描述符都已关闭，此时进程再次往管道里面写入数据，写操作会失败，errno被设置为EPIPE，同时内核会向进程发送一个SIGPIPE信号
3. 当所有的读取端和写入端都关闭后，管道才能被销毁
```

### example
```
#include<unistd.h>
#include<sys/types.h>
#include<errno.h>
#include<stdio.h>
#include<stdlib.h>

int main() {
    int pipe_fd[2];
    pid_t pid;
    char r_buf[4096];
    char w_buf[4096];
    int writenum;
    int rnum;

    memset(r_buf, 0, sizeof(r_buf));
    if(pipe(pipe_fd) < 0) {
	    printf("[PARENT] pipe create error!\n");
	    return -1;
    }
    printf("\n");
    if((pid = fork()) == 0) { // 子进程
	    close(pipe_fd[1]);
	    while(1) {
	        rnum = read(pipe_fd[0], r_buf, 1000);
	        printf("[CHILD] readnum is %d\n", rnum);
	        if(rnum == 0) { // EOF
		        printf("[CHILD] all the writer of pipe are closed. break and exit.\n");
		        break;
	        }
	    }
	    close(pipe_fd[0]);
	    exit(0);
    } else if(pid > 0) {  // 父进程
	    printf("[PARENT] fork success pid:%d \n", pid);
	    close(pipe_fd[0]);
	    memset(w_buf, 0, sizeof(w_buf));
	    if(writenum = write(pipe_fd[1], w_buf, 1024) == -1) {
	        printf("[PARENT] write to pipe error\n");
	    } else {
	        printf("[PARENT] the bytes write to pipe is %d \n");
	    }
	    sleep(15);
	    printf("[PARENT] I will close the write end of pipe.\n");
	    close(pipe_fd[1]);
	    sleep(2);
	    return 0;
    }
}

从proc fs中可以看到父进程打开的管道
[root@iz2ze2vve1jzbbft74642sz ~]# cd /proc/27831/fd
[root@iz2ze2vve1jzbbft74642sz fd]# ll
总用量 0
lrwx------ 1 root root 64 2月  23 15:06 0 -> /dev/pts/0
lrwx------ 1 root root 64 2月  23 15:06 1 -> /dev/pts/0
lrwx------ 1 root root 64 2月  23 15:06 2 -> /dev/pts/0
lr-x------ 1 root root 64 2月  23 15:06 3 -> pipe:[34948360]

用lsof也可以看到进程打开的管道
[root@iz2ze2vve1jzbbft74642sz fd]# lsof | grep FIFO | grep 27854
1         27854          root    3r     FIFO                0,8       0t0   34962078 pipe
```

### 管道大小
```
管道的本质是对应一片内存区域，内存区域就有大小的区分，从Linux 2.6.11版本起，管道的默认大小是65536字节，可以调用fcntl来获取和修改这个值的大小

获取管道大小
pipe_capacity = fcntl(fd, ?F_GETPIPE_SZ);

设置管道大小
ret = fcntl(fd, ?F_SETPIPE_SZ, size);

管道内存区域大小必须在页面大小(PAGE)和上限值之间，上限记录在
#cat /proc/sys/fs/pipe-max-size
1048576

在使用管道时，管道有大小，写入须谨慎，不能连续地写入大量的内容，一旦管道满了，写入就会被阻塞；
对于读取端，要及时读取，防止管道被写满，造成写入阻塞
```