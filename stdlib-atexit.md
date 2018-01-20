## atexit
只有在进程正常退出时，其定义的退出函数才会执行，exit()退出或者main的return退出，如果是因为收到信号而退出，则不会被调用

### atexit运用
```
#include<stdio.h>
#include<stdlib.h>

static void callback1(void)
{
    printf("callback1\n");
}

static void callback2(void)
{
    printf("callback2\n");
}

static void callback3(void)
{
    printf("callback3\n");
}

int main(void)
{
    atexit(callback1);
    atexit(callback2);
    atexit(callback3);
    printf("main exit\n");
    return 0;
}

输出:
main exit
callback3
callback2
callback1
```

### atexit的局限
```
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

static void callback1(void)
{
    printf("callback1\n");
}

int main(void)
{
    atexit(callback1);
    while(1)
    {
	sleep(1);
    }
    printf("main exit\n");
    return 0;
}

测试:
killall atexit1
输出:
已终止
```
