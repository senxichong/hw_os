### Linux 下常见的3种系统调用方法包括： 
 (1)通过glibc提供的库函数调用
（2）使用syscall函数直接调用相应的系统调用 
 (3)通过int 80指令（32位系统）或者syscall指令（64位系统）的内联汇编调用 
 
请研究Linux(kernel>=2.6.24) getpid和open这两个系统调用的用法，分别使用上
述3种系统调用方法来执行。要求： 
1）针对同一个系统调用，记录和对比3种方法的运行时间，并尝试解释时间差异的
原因； 
2）对比getpid和open这两个系统调用在使用内联汇编实现调用时的运行时间，如
果有差异，尝试解释差异原因。若无差异，说明即可。 
### answer
* 函数选择：
    clock_gettime 更加准确，并且不会受到系统时间调整的影响。
* 系统环境:
```shell 
$  uname -a
Linux ubuntu 5.15.0-117-generic #127~20.04.1-Ubuntu SMP Thu Jul 11 15:36:12 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux

```
* 代码：
```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <time.h>

void measure_time(const char* description, void (*func)()) {
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);
    func();
    clock_gettime(CLOCK_MONOTONIC, &end);

    long ns = (end.tv_sec - start.tv_sec) * 1e9 + (end.tv_nsec - start.tv_nsec);
    printf("%s: %ld ns\n", description, ns);
}

void getpid_glibc() {
    getpid();
}

void getpid_syscall() {
    syscall(SYS_getpid);
}

void getpid_inline_asm() {
    int pid;
    asm volatile (
        "movl $39, %%eax;"  // SYS_getpid = 39
        "syscall;"          // 执行系统调用
        "movl %%eax, %0;"   // 将返回值存储到pid
        : "=r" (pid)
        :
        : "%eax"
    );
}

void open_glibc() {
    int fd = open("/tmp/testfile", O_RDONLY | O_CREAT, 0644);
    close(fd);
}

void open_syscall() {
    int fd = syscall(SYS_open, "/tmp/testfile", O_RDONLY | O_CREAT, 0644);
    close(fd);
}


void open_inline_asm() {
    long fd;  // 确保 fd 是 64 位
    const char *path = "/tmp/testfile";  // 字符串指针本来就是 64 位的
    long flags = O_RDONLY | O_CREAT;  // 确保 flags 是 64 位
    long mode = 0644;  // 确保 mode 是 64 位

    asm volatile (
        "mov $2, %%rax;"         // SYS_open = 2 (直接用mov让编译器选择合适的指令)
        "mov %1, %%rdi;"         // 文件路径
        "mov %2, %%rsi;"         // 打开标志
        "mov %3, %%rdx;"         // 权限
        "syscall;"               // 执行系统调用
        "mov %%rax, %0;"         // 将返回值存储到 fd
        : "=r" (fd)
        : "r" (path), "r" (flags), "r" (mode)
        : "%rax", "%rdi", "%rsi", "%rdx"
    );

    close(fd);
}


int main() {
    measure_time("getpid (glibc)", getpid_glibc);
    measure_time("getpid (syscall)", getpid_syscall);
    measure_time("getpid (inline asm)", getpid_inline_asm);

    measure_time("open (glibc)", open_glibc);
    measure_time("open (syscall)", open_syscall);
    measure_time("open (inline asm)", open_inline_asm);

    return 0;
}
```
*结果：
```shell
$ gcc test.c 
$ time ./test
getpid (glibc): 1333 ns
getpid (syscall): 1169 ns
getpid (inline asm): 574 ns
open (glibc): 13879 ns
open (syscall): 6084 ns
open (inline asm): 4735 ns

real	0m0.002s
user	0m0.000s
sys	0m0.002s
```


    glibc 方法：最慢，因为它包含了库函数的额外开销，例如参数检查和返回值处理。
    syscall 方法：稍快一些，因为它直接调用系统调用，省去了 glibc 的封装层。
    内联汇编方法：与 syscall 方法速度相近，略快一些，但差异不大。由于内联汇编减少了一些编译器优化的空间，速度提升有限。


    getpid 是一个非常简单的系统调用，仅返回当前进程的 PID，因此其时间开销非常小，差异不大。
    open 是一个较为复杂的系统调用，需要执行文件系统的操作，如查找文件路径、权限检查等，因此其时间开销相对较大，但不同调用方法之间的差异仍然主要由封装层开销决定。