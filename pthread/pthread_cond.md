## 条件变量
条件等待是线程同步的另一种方法
NPTL使用pthread_cond_t类型的量来表示条件变量
条件变量不是一个值，无法给条件变量赋值

### 初始化 和 销毁
```
int pthread_cond_init(pthread_cond_t *cv,const pthread_condattr_t *cattr);

pthread_cond_t cond = PTHREAD_COND_INITIALIZER;   // 不需要销毁

int pthread_cond_destroy(pthread_cond_t *cv);
```