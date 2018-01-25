## 命名管道FIFO

### 接口定义
```
#include<sys/types.h>
#include<sys/stat.h>
int mkfifo(const char * pathname, mode_t mode);

参数:
const char * pathname 管道文件的文件名
mode_t mode FIFO文件的读写执行权限，和chmod的参数一样
```

### 命令行模式下的管道
```
mkfifo [-m mode] pathname
或者
mknod [-m mode] pathname p    // 这里末尾的p表示要创建命名管道

ls -l 查看到的第一个字母是p，表示是命名管道文件
```

### 判断文件是FIFO文件
```
shell编程中判断是否是FIFO文件
-p file

c语音中判断是否是FIFO文件
通过S_ISFIFO宏判断，不过要先通过stat或者fstat函数来获取文件的属性
#include<sys/types.h>
#include<sys/stat.h>
#include<unistd.h>
int stat(const char *path, struct stat *buf);
int stat(int fd, struct stat *buf);
S_IS_FIFO(buf->st_mode);
```

### 写入限制
POSIX标准规定PIPE_BUF最少为512字节，对于linux而言，PIPE_BUF是4096，一个页面的大小

### 打开FIFO文件
