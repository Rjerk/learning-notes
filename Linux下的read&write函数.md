# Linux中的 read & write 函数

read & write 函数原型：

```
#include <unistd.h>

ssize_t read(int fd, void* buf, size_t count);
ssize_t write(int fd, const void* buf, size_t count);
```

## read

read从文件描述符fd 中读取count 字节数据到以buf 为起点的缓存区中，如果已经到达文件末尾，就返回0。

读操作从文件当前的偏移量开始，在成功返回前，偏移量将增加实际读到的字节数。

返回值为带符号整数(ssize_t)，保证能返回正整数字节数、0（文件尾端）或 -1（出错）。出错时返回-1，同时errno 被设置以指出错误类型：
- EAGAIN 或 EWOULDBLOCK：文件或socket 为非阻塞模式，而且读操作被阻塞，没有数据可读。
- EBADF：fd 不是一个有效的文件描述符或没有为读操作而打开。
- EFAULT：buf 在可访问地址空间之外。
- EINTR：在读取数据前该调用被信号打断。
- EIO：I/O 错误。
- ...

所以对于read的返回值处理应该这样使用：

```
while (true) {
    int ret = read(fd, buf, count);
    if (ret == -1) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            //handle nonblock;
            continue;
        } else if (errno == EINTR) {
            continue;
        }
        // other error happended, deal with it.
        // ...
        exit(-1);
    } else if (ret == 0) {
        close(fd);
    } else {
        break;
    }
}
```

因为read 读取字节流，返回的字节数无法预计，所以要多次调用read 直到接受到整个报文。

```
char* p = buf;
ssize_t total = 0;

while (count > 0) {
    ssize_t loaded = read(fd, p, count);
    if (loaded < 0) {
        if (errno == EINTR) {
            loaded = 0;
            continue;
        } else if (...) {
            // handle errno
        }
    } else if (loaded == 0) {
        close(fd);
        return total;
    }
    total += loaded;
    p += loaded;
    count -= loaded;
}

```

## write

write 从以buf 为起点的缓存区中向文件描述符fd 中写入count 字节的数据。

如果调用成功，返回实际写入的字节数。写入的字节数可能小于count，称为“部分写”，对磁盘来说造成“部分写”的原因可能是由于磁盘已满，或是进程资源对文件大小的限制，或是在写入小于count字节时调用被信号打断。

对于普通文件，写操作从文件当前的偏移量开始，如果打开文件时指定了O_APPEND，则每次写操作前将偏移量设为文件当前结尾处。在一次成功写后，文件偏移量增加实际写的字节数。

发生错误时，返回-1，并设置errno:
- EAGIN
- EBADF：fd 不是一个有效的文件描述符或未打开
- EINTR：在写数据前调用被信号打断
- EIO：I/O 错误
- EFALUT：buf 在可访问地址空间之外
- EPIPE：fd 所连接的管道或socket 的读端被关闭

write 的使用也和read 类似

```
const char* p = buf;
ssize_t total = 0;

while (count > 0) {
    ssize_t written = write(fd, p, count);
    if (written < 0) {
        if (errno == EINTR) {
            written = 0;
            continue;
        } else if (...) {
            // handle error
        }
    } else if (writen == 0) {
        close(fd);
        return total;
    }
    total += written;
    p += written;
    count -= written;    
}
```
