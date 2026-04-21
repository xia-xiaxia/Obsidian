# 介绍

首先，我们在下载好了xv6-labs-2022代码后，运行一下他：
```bash
make clean
make qume
```

此时，xv6内核运行起来了。

### ls命令

我们运行ls命令：
输出：

```sh
.              1 1 1024
..             1 1 1024
README         2 2 2227
xargstest.sh   2 3 93
cat            2 4 34704
echo           2 5 33488
forktest       2 6 16816
grep           2 7 38256
init           2 8 33864
kill           2 9 33416
ln             2 10 33208
ls             2 11 36824
mkdir          2 12 33472
rm             2 13 33464
sh             2 14 57176
stressfs       2 15 34328
usertests      2 16 188488
grind          2 17 49952
wc             2 18 35736
zombie         2 19 32728
console        3 20 0
$ 
```

屏幕中显示了根目录下的文件列表，这些主要由 **系统文件**、**基础命令（工具）** 和 **测试程序** 组成。

xv6 `ls` 命令的输出格式通常是：`[文件名] [文件类型] [inode号] [文件大小]`。

​	**文件类型**：`1` 代表目录 (Directory)，`2` 代表普通文件 (File)，`3` 代表设备 (Device)。

下面是这些文件和接口的详细分类介绍：
#### 📁 核心系统与特殊文件

- **. (当前目录)** 和 **.. (父目录)**：类型为 `1`。标准的 Unix 目录导航符。
- **README**：类型为 `2`。包含 xv6 项目的基本说明和如何编译运行的文档。
- **init**：系统启动后运行的**第一个用户态程序**（PID 恒为 1）。它的主要工作是初始化控制台，然后启动 shell (`sh`)。如果它退出，系统通常会直接 panic。
- **console**：类型为 `3`（设备文件）。在 Unix 哲学中“一切皆文件”，这个文件代表了系统的控制台输入和输出（键盘和显示器）。`init` 程序会将标准输入（文件描述符 0）、标准输出（1）和标准错误（2）都绑定到这个设备上。

#### 🛠️ 基础用户命令 (Utilities)

这些是我们在 Linux/Unix 系统中最常用的基础工具，在第一个实验（Lab: Xv6 and Unix utilities）中，你可能需要自己实现或修改其中的一部分：

- **sh**：xv6 的 Shell 程序（命令行解释器）。它负责读取你输入的命令，解析它们，并通过 `fork` 和 `exec` 系统调用来运行下面的这些程序。
- **cat**：读取文件内容并将其打印到标准输出（屏幕）。
- **echo**：将其接收到的参数打印回屏幕，常用于脚本或测试标准输出。
- **grep**：文本搜索工具，用于在文件中查找匹配特定模式（正则表达式/字符串）的行。
- **ls**：列出目录中的文件信息（也就是你刚刚运行的命令）。
- **mkdir**：创建新的目录。
- **rm**：删除指定的文件或目录。
- **ln**：创建硬链接（Hard link），让多个文件名指向同一个物理文件内容。
- **kill**：向指定的进程发送终止信号以结束其运行。
- **wc**：Word Count，统计并输出文件的行数、单词数和字节数。

#### 🧪 实验与系统测试程序

这些程序是专门为了验证 xv6 内核实现是否正确、或者用于对你的 Lab 代码进行自动评分的测试集：

- **usertests**：**这是最重要的测试程序**。它包含了 xv6 官方编写的一大套综合测试用例。当你修改了内核代码（比如改了内存管理或进程调度）后，运行 `usertests` 可以检查你是否破坏了系统原有的功能。
- **forktest**：专门用于高强度测试 `fork` 系统调用功能的程序，验证进程创建机制是否稳定。
- **xargstest.sh**：一个 shell 脚本，专门用于测试你在 Lab 1 中要求编写的 `xargs` 命令是否正确工作。
- **stressfs**：文件系统压力测试程序。它会并发地执行大量的文件读写操作，用于测试文件系统的锁机制和稳定性。
- **grind**：另一个强力系统压力测试工具，通常用于模拟多进程高负载环境下的系统运行状态，揪出并发 bug。
- **zombie**：这是一个用于创建“僵尸进程”的测试程序。它能帮助你测试/观察父进程调用 `wait` 系统调用的行为，以及操作系统的进程清理（回收）机制是否正常。

---

# Lab1

Lab 1 一共包含以下 6 个具体要求/任务：

**1. Boot xv6 (启动 xv6)**

- **要求**：这不是一个写代码的任务，而是要求你配置好实验环境，编译并成功运行 xv6。你需要学会使用 `make qemu` 启动系统，并在 shell 中运行一些基础命令（比如你上一条提到的 `ls`），最后学会通过快捷键 `Ctrl-a x` 退出 QEMU。

**2. sleep (实现 sleep 命令)**

- **要求**：编写程序 `user/sleep.c`。
- **功能**：让程序暂停指定的“滴答数”（ticks）。用户在命令行输入类似 `sleep 10` 的命令。
- **考察点**：处理命令行参数（`argc` 和 `argv`），将字符串转换为整数（使用 `atoi`），并调用系统内置的 `sleep`系统调用。

**3. pingpong (实现 pingpong 通信)**

- **要求**：编写程序 `user/pingpong.c`。
- **功能**：使用 Unix 管道（pipes）在两个进程之间传递一个字节的数据。
  - 父进程通过管道向子进程发送一个字节，子进程收到后打印 `<pid>: received ping`。
  - 子进程再通过另一个管道将字节发回给父进程，父进程收到后打印 `<pid>: received pong`。
- **考察点**：熟练掌握 `fork`（创建进程）、`pipe`（创建管道）、`read`、`write` 以及 `getpid`（获取进程 ID）系统调用的组合使用。

**4. primes (实现并发素数筛)**

- **要求**：编写程序 `user/primes.c`。
- **功能**：使用管道网络实现一个并发的素数生成器（基于贝尔实验室的 Sieve of Eratosthenes 算法）。你需要创建一个进程管道（pipeline），每个进程负责筛选掉特定素数的倍数，然后将剩下的数字传递给下一个新创建的子进程。
- **考察点**：这是 Lab 1 中**最难**的一个任务。深度考察对 `fork` 和递归管道流的理解。你需要特别注意在不需要管道读/写端时及时关闭它们（`close`），否则程序会耗尽文件描述符或导致读取阻塞。

**5. find (实现简易 find 命令)**

- **要求**：编写程序 `user/find.c`。
- **功能**：在指定的目录树中查找具有特定名称的所有文件。例如 `find . b` 会在当前目录及其所有子目录下寻找名为 `b` 的文件，并打印出它们的相对路径。
- **考察点**：理解 xv6 的文件系统结构。你需要学习如何打开目录、读取目录条目（`dirent`），以及如何使用递归遍历子目录。官方提示你可以参考现有的 `user/ls.c` 源码来学习如何读取目录。

**6. xargs (实现简易 xargs 命令)**

- **要求**：编写程序 `user/xargs.c`。
- **功能**：从标准输入（standard input）逐行读取数据，然后为每一行数据运行指定的命令，并将该行数据作为命令的附加参数。
  - 例如：`echo "1\n2" | xargs echo line` 应该输出： `line 1` `line 2`
- **考察点**：考察对标准输入（文件描述符 0）的循环读取、字符串处理，以及熟练使用 `fork`、`exec`（执行新程序）和 `wait` 系统调用。

我们来逐步完成上面的任务，就从任务2开始吧：
	不过我们还有些需要注意的事情，我们写的脚本实际上是在xv6这个系统里面运行的，所以不能用clang/gcc等编译器编译。

​	因为 xv6 是一个独立的操作系统。如果你直接用宿主机（比如你的 Ubuntu 或 macOS）的编译器，它会默认链接宿主机的 C 标准库（glibc/libc）。而你的程序是要跑在 xv6 系统里的，xv6 有它自己精简版的系统调用接口和库（存放在 `user/user.h` 中），底层架构也是 RISC-V。所以，必须使用 xv6 提供的**交叉编译（Cross-compilation）**工具链。

​	不过不用担心，xv6的环境已经为你配置好了自动化构建脚本，所以我们只需要在`user/` 下面写代码就好了，以下是需要注意的东西（**xv6编程规范**）：

1. **头文件**：不能引入 `<stdio.h>` 等标准库，必须引入 xv6 的系统库：

   ```c
   #include "kernel/types.h"
   #include "kernel/stat.h"
   #include "user/user.h"
   ```

2. **获取参数**：通过 `main(int argc, char *argv[])` 获取命令行参数。注意 `argv[0]` 是程序名（"sleep"），`argv[1]` 才是你要睡眠的时间（比如 "10"）。

3. **类型转换**：用 `atoi()` 函数将字符串类型的数字转成整型。

4. **系统调用**：调用系统提供的 `sleep(ticks)` 函数。

5. **退出**：程序结束时必须调用 `exit(0);`，不能只是 `return 0;`

6. **user.h**：

   ```c
   struct stat;
   
   // system calls
   int fork(void);
   int exit(int) __attribute__((noreturn));
   int wait(int*);
   int pipe(int*);
   int write(int, const void*, int);
   int read(int, void*, int);
   int close(int);
   int kill(int);
   int exec(const char*, char**);
   int open(const char*, int);
   int mknod(const char*, short, short);
   int unlink(const char*);
   int fstat(int fd, struct stat*);
   int link(const char*, const char*);
   int mkdir(const char*);
   int chdir(const char*);
   int dup(int);
   int getpid(void);
   char* sbrk(int);
   int sleep(int);
   int uptime(void);
   
   // ulib.c
   int stat(const char*, struct stat*);
   char* strcpy(char*, const char*);
   void *memmove(void*, const void*, int);
   char* strchr(const char*, char c);
   int strcmp(const char*, const char*);
   void fprintf(int, const char*, ...);
   void printf(const char*, ...);
   char* gets(char*, int max);
   uint strlen(const char*);
   void* memset(void*, int, uint);
   void* malloc(uint);
   void free(void*);
   int atoi(const char*);
   int memcmp(const void *, const void *, uint);
   void *memcpy(void *, const void *, uint);
   
   ```

   

### sleep

这个比较简单，因为`user.h`中实现了`sleep()`这个函数，我们只需要处理输入输出并且调用它：
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h" // 这个文件里面包含了 xv6 所有的系统调用声明（如 sleep,exit,atoi等）

int main(int argc, char *argv[]) {
// 1.检查命令行参数的数量
// argc 代表的是参数的个数，argv[0] -> "sleep" argv[1] -> "num"
if(argc != 2) {
// 如果参数输入错误，打印错误提示
fprintf(2,"Usage: sleep <ticks>\n");
// 参数错误，非正常退出，通常传入非 0 值
exit(1);
}

// 2.转换参数 使用 xv6 提供的 atoi 函数将字符串转换为整数。
int ticks = atoi(argv[1]);
if(ticks < 0) {
// ticks必须为正数
fprintf(2,"Usage: ticks can't be negative");
exit(1);
}

sleep(ticks);
exit(0);
}
```

注意，写完之后我们要在`Makefile`下的`UPROGS`列表下面添加`$U/sleep\`，然后我们还可以用官方的评分脚本单独测试这个程序：
```bash
./grade-lab-util sleep
```

![image-20260409212522986](/Users/xiatian/Library/Application Support/typora-user-images/image-20260409212522986.png)

最终得倒上述结果。

### pingpong

首先我们需要了解只是需要实现这个pingpong功能，而不是一个命令行驱动的命令，实际上系统内自会调用。
同时，该实验只是需要我们