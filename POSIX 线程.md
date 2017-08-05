# POSIX 线程

线程在繁忙使用的网络服务器上的一个优势是，创建一个新线程通常比使用 fork 派生一个新进程快得多。

另外，同一进程内的所有线程共享相同的全局内存，这使得线程之间易于共享信息。

## 基本线程函数：创建和终止

当一个程序由 exec 启动执行时，单个线程就被创建，称为初始线程或主线程。其余线程可以通过 pthread_create 创建。

```
#include <pthread.h>

int pthread_create(pthread_t* tid, const pthread_attr_t* attr,
                   void* (*func)(void *), void* arg);
// 成功返回 0，出错返回错误码。 
```

tid 返回新创建的线程ID；在创建时可以初始化一个取代默认设置的 pthread_attr_t 来指定线程属性，比如优先级、初始栈大小、指定守护线程等；线程执行函数及参数由 func 和 arg 指定。

线程终止有三种方式：
1. 调用 pthread_exit 函数
2. 从 func 返回
3. 如果进程的 main 函数或任何线程调用了 exit，整个进程就终止，其中包括它的任何线程

```
#include <pthread.h>

void pthread_exit(void* status);
```

pthread_exit 函数终止调用线程并且通过 status 返回一个值。status 不能指向局部于调用线程的对象，因为线程终止时这样的对象也消失。

```
#include <pthread.h>

int pthread_join(pthread_t* tid, void** status);
// 成功返回 0，出错返回错误码。 
```

可以调用 pthread_join 等待一个给定的线程终止（类似 waitpid）。tid 指定的线程必须是可加入的（joinable）。如果 status 非空，来自所等待线程的返回值（一个指向某个对象的指针）将存入 status 指向的位置。

```
#include <pthread.h>

pthread_t pthread_self(void);
// 返回调用线程的线程 ID。
```

每个线程使用 pthread_self 获取自身的线程 ID（类似于 getpid）。

```
#include <pthread.h>

int pthread_detach(pthread_t tid);
// 成功返回 0，出错返回错误码。
```

当一个可加入的线程终止时，它的线程 ID 和退出状态将留存到另一个线程对它调用 pthread_join。脱离的线程却像守护进程，当它们终止时，所有相关资源都被释放，我们不能等待它们终止。

如果线程想让自己脱离，可以这样做：

```
pthread_detach(pthread_self());
```

## 线程特定数据

使用线程特定数据可以使得先有函数变为线程安全。

每个系统支持有限数量的线程特定数据元素（POSIX 要求不小于 128 个）。系统为每个进程维护一个我们称之为 Key 结构的结构数组。

典型的线程特有数据实现如下：

```
key[0]  |    标志   |
        |析构函数指针|
key[1]  |    标志   |
        |析构函数指针|
        |          |
        |    ...   |
        |          |
key[127]|    标志   |
        |析构函数指针|
```

Key 结构中的标志指示这个数组元素是否正在使用，所有的标志初始化为 “不在使用”。当一个线程调用 pthread_key_create 创建一个新的线程特定元素时，系统搜索其 Key 结构数组找出第一个不在使用的元素，将该元素的索引（0~127）返回给调用线程（通过 pthread_key_create 的第一个参数 pthread_key_t* keyptr）。

除了进程范围的 Key 结构数组外，系统还在进程内维护每个线程的多条信息，这些特定于线程的信息我们称之为 Pthread 结构，其中的 pkey 的指针数组内元素被初始化为 NULL，其内 128 个指针是和进程内 128 个可能的索引逐一关联的值。

```
             线程 0             线程 1 ...
            Pthread{}
         —————————————
         |           |
         |           |
         |           |
         |其他线程信息|
         |           |
         |           |
         |           |
         ————————————
pkey[0]  |    指针   | NULL
pkey[1]  |    指针   | NULL
         |    ...    |
         |           |
pkey[127]|    指针   | NULL
         ————————————
```

当调用 pthread_key_create 创建一个 Key 时，系统返回一个索引。每个线程可以随后为该索引存储一个析构函数指针，这个指针通常是每个线程通过调用 malloc 获得的。真正的线程特定数据就是这个指针所指向的任何内容。

当一个线程终止时，系统将会扫描该线程的 pkey 数组，为每个非空的 pkey 指针调用相应的析构函数。

```
#include <pthread.h>

int pthread_key_create(pthread_key_t* keyptr, void (*destructor)(void* value));
int pthread_once(pthread_once_t* onceptr, void (*init)(void));
// 成功返回 0，错误返回错误码。
```

每当一个使用线程特定数据的函数被调用时，pthread_once 通常转而被该函数调用，pthread_once 使用 onceptr 指向变量中的值，确保 init 所指的函数在进程范围内只被调用一次。

在进程范围内对于一个给定键，pthread_key_create 只能被调用一次，所创建的键通过 keyptr 返回，如果 destructor 不为空，则其所指的函数将由为该键存放过某个值的每个线程在终止时调用。

```
#include <pthread.h>

void* pthread_getspecific(pthread_key_t key);
// 返回指向特定线程数据的指针。

int pthread_setspecific(pthread_key_t key, const void* value);
// 成功返回 0，出错返回错误码。
```

## 互斥锁

涉及到多个线程更改一个共享变量，需要用互斥锁来保护这个共享变量；访问该变量的前提条件就是拥有这个互斥锁。如果一个线程试图上锁一个已经被另一个线程锁住的互斥锁，这个线程就会被阻塞，直到互斥锁被解锁。

上锁和解锁：

```
#include <pthread.h>

int pthread_mutex_lock(pthread_mutex_t* mptr);
int pthread_mutex_unlock(pthread_mutex_t* mptr);
// 成功返回 0，出错返回错误码。
```

如果某个互斥锁变量是静态分配的，必须把它初始化为常值 PTHREAD_MUTEX_INITIALIZER。

## 条件变量

互斥锁提供一种防止多线程同时访问某个共享变量的互斥机制，条件变量则提供一种在等待某个条件发生期间让线程睡眠的信号机制。

使用条件变量：

```
#include <pthread.h>

int pthread_cond_wait(pthread_cond_t* cptr, pthread_mutex_t* mptr);
int pthread_cond_signal(pthread_cond_t* cptr);

int pthread_cond_broadcast(pthread_cond_t* cptr);
int pthread_cond_timedwait(pthread_cond_t* cptr, pthread_mutex_t* mptr, const struct timespec* abstime);
// 成功返回 0，出错返回错误码。
```

pthread_cond_signal 通常唤醒等在相应条件变量上的单个线程，pthread_cond_broadcast 则可以唤醒等在相应条件变量上的所有线程，pthread_cond_timedwait 允许线程设置一个阻塞时间的限制。这个时间是绝对时间，而不是时间增量，即指定返回时刻的系统时间 —— 从 1970 年 1 月 1 日 UTC 时间以来的秒数和纳米数。通常使用 gettimeofday 获取当前时间(struct timeval)，把它复制到一个 tiemspec 结构中，再加上期望的时间限制。

```
struct timeval tv;
struct timespec ts;

if (gettimeofday(&tv, NULL) < 0) {
    err_sys("gettimeofday error");
}

ts.tv_sec = tv.tv_sec + 5;
ts.tv_nsec = tv.tv_usec * 1000;

pthread_cond_timewait(..., &ts);
```

其优点是，如果函数过早返回（比如收到某个信号），则不必改动 timespec 参数的内容就可以再次调用该函数；缺点是首次调用该函数时不得不调用 gettimeofday。