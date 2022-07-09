解析```zmalloc.c```文件



## ```zmalloc```

```redis```对```malloc```函数进行了封装

如果要分配```size```个字节大小的空间, 那么```zmalloc```会为其分配```size+sizeof(size_t)```个字节的空间

因为```zmalloc```会在该头地址开始的```sizeof(size_t)```个字节的空间中记录下```size```这个类型为```size_t```的数

所以```zmalloc```申请的空间首地址为```p```, 实际返回的指针却是```(char*)p+sizeof(size_t)``` (这里用```char*```转的目的是```char```只占一个字节,可以精确表示首地址的含义)

```c
static void zmalloc_default_oom(size_t size) {
    fprintf(stderr, "zmalloc: Out of memory trying to allocate %zu bytes\n",
        size);
    fflush(stderr); 
    abort(); // 终止程序
}

static void (*zmalloc_oom_handler)(size_t) = zmalloc_default_oom;  // zmalloc_oom_handler是一个函数指针

void *zmalloc(size_t size) {
    void *ptr = malloc(size+PREFIX_SIZE);  // 申请size+sizeof(size_t)的空间(PREFIX_SIZE是一个宏)

    if (!ptr) zmalloc_oom_handler(size);  // 如果分配不成功, 打印错误信息, 结束程序
#ifdef HAVE_MALLOC_SIZE  // 该宏的定义可以在zmalloc.h中找到, 在windows中不会执行该分支.(为了考虑不同系统兼容性)
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else  // 走这个分支
    *((size_t*)ptr) = size;  // 在起始的sizeof(size_t)个字节保存,分配多少空间
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);  
    return (char*)ptr+PREFIX_SIZE;  // 返回的地址,是首地址往移sizeof(size_t)个字节
#endif
}
```

上面注意最后返回指针前, 都会执行一个叫```update_zmalloc_stat_alloc```的宏

它作用就是记录下```redis```目前已经分配了多大的空间

```
#define update_zmalloc_stat_add(__n) do { \
    pthread_mutex_lock(&used_memory_mutex); \
    used_memory += (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)

#define update_zmalloc_stat_alloc(__n) do { \
    size_t _n = (__n); \
    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
    if (zmalloc_thread_safe) { \  // 开启了线程安全, 所以对_n的修改需要加锁
        update_zmalloc_stat_add(_n); \
    } else { \
        used_memory += _n; \  // 没开线程安全, 直接加
    } \
} while(0)
```

相信其他语句都不难理解, 唯独这一句

```c
if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1));
```

我们知道```malloc```会按照```sizeof(long)```字节进行对齐(如果不是```sizeof(long)```的整数倍会补齐,我的运行环境中```sizeof(long)=4```)

而```redis```需要统计的是实际使用内存, 所以肯定需要考虑这些多分配的空间.

这里使用位运算快速得到了需要补齐的大小, 挺有技巧性的.



## ```zcalloc```

与```zmalloc```的用途一致, 但是会初始化分配的内存, 因为底层分配空间使用的是```calloc```

```calloc```函数原型

```c
void* calloc（unsigned int num，unsigned int size）；
```

它可以在内存的动态存储区中分配```num```个长度为```size```的连续空间

```calloc```在动态分配完内存后，自动初始化该内存空间为零，而```malloc```不做初始化，分配到的空间中的数据是随机数据。

因为```calloc()```函数会清空分配的内存，而```malloc()```函数不会，所以可以调用以```1```作为第一个实参的```calloc()```函数，为任何类型的数据项分配空间。比如：

```c
struct point{ int x, y;} *pi;
pi = calloc(1, sizeof(struct point));
```

接下来看以下```zcalloc```的源码

```c
void *zcalloc(size_t size) {
    void *ptr = calloc(1, size+PREFIX_SIZE);  // 仍然是申请size+sizeof(size_t)的空间，这次使用calloc来初始化

    if (!ptr) zmalloc_oom_handler(size);  // 输出错误信息
#ifdef HAVE_MALLOC_SIZE    // 不走该分支
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size;  // 开头位置的sizeof(size_t)字节存放分配的空间数size
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);  // redis记录已分配的内存数
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```



## ```zrealloc```

前面的```zmalloc,zcalloc```都是分配内存使用的, ```zrealloc```则是对```realloc```的封装




