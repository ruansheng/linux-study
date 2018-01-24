## pthread互斥量
互斥量就是互相排斥之意，用来保护共享数据，保证操作是原子性的

### 临界区
```
临界区的大小设定很重要:
设置小了可能达不到预想的效果，保护不了共享数据
设置大了可能影响性能
```

### 互斥量初始化
```
POSIX提供了2种初始化互斥量的方法:
方法一:
#include<pthread.h>
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

方法二:
#include<pthread.h>
int pthread_mutex_init(pthread_mutex_t *restrict mutex,const pthread_mutexattr_t *restrict attr)
第二个参数是用来设置互斥量的属性的，大部分情况默认传NULL即可
```

### 互斥量销毁
```
使用PTHREAD_MUTEX_INITIALIZER初始化的互斥量不需要销毁
不要销毁一个已加锁的互斥量，或者正在配合条件变量使用的互斥量
已销毁的互斥量确保后面不会再加锁

#include<pthread.h>
int pthread_mutex_destroy(pthread_mutex_t *mutex); 
```

### 互斥量的加锁解锁
```
#include<pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock( pthread_mutex_t *mutex );
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```