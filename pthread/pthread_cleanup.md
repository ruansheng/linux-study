## pthread_cleanup_push和pthread_cleanup_pop
设置清理函数
pthread_cleanup_push 和 pthread_cleanup_pop 必须同时出现在同一代码块中

### 函数原型
```
void pthread_cleanup_push(void (*routine) (void  *),  void *arg)
void pthread_cleanup_pop(int execute)
```