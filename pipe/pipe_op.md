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