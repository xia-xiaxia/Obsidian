这篇笔记用来系统整理 `xv6-labs-2022` 中 [`user.h`](#源文件) 暴露给用户程序的接口。

学习这份文件时，最重要的是先分清两类东西：

1. `// system calls`
这些是真正的系统调用。用户程序调用它们时，会从用户态陷入内核态，由内核帮我们完成工作。

2. `// ulib.c`
这些主要是用户态库函数。它们大多运行在用户空间，不一定进入内核；有些库函数会在内部再去调用系统调用。

---

## 源文件

`user.h` 中列出的接口如下：

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

---

## 先建立整体认识

### 1. 什么是系统调用

用户程序不能直接操作硬件，也不能随意访问内核中的数据结构。所以当程序需要：

- 创建进程
- 读写文件
- 申请更多地址空间
- 睡眠等待

就必须通过系统调用请求内核代办。

在 xv6 中，过程可以粗略理解为：

1. 用户程序调用 `fork()`、`read()`、`open()` 这样的函数。
2. 这些函数并不是直接干活，而是触发一次 `ecall`。
3. CPU 从用户态切换到内核态。
4. 内核根据系统调用号找到对应的 `sys_xxx` 函数。
5. 内核检查参数、执行操作、返回结果。
6. CPU 回到用户态，调用者拿到返回值。

所以，`user.h` 里这些“函数声明”其实是用户程序与内核交互的入口。

### 2. 什么是文件描述符 fd

很多接口都带有 `fd` 参数，例如：

```c
int read(int fd, void *buf, int n);
int write(int fd, const void *buf, int n);
int close(int fd);
int dup(int fd);
```

这里的 `fd` 是 file descriptor，文件描述符。它本质上是一个小整数，是“当前进程打开文件表”的下标。

通常约定：

- `0`：标准输入 stdin
- `1`：标准输出 stdout
- `2`：标准错误 stderr

你可以把它理解成：用户程序手里拿到的不是“文件对象本身”，而是一个编号，内核根据这个编号找到真正的打开文件结构。

### 3. `struct stat` 是什么

`struct stat` 用来描述文件的信息，例如：

- 文件类型
- inode 编号
- 链接数
- 文件大小

`fstat()` 和 `stat()` 都会把结果填进这个结构体里。

### 4. 返回值的一般规律

xv6 中大多数系统调用都遵循下面的风格：

- 成功：返回非负值，通常是 `0` 或一个有意义的结果
- 失败：返回 `-1`

所以写程序时，要养成检查返回值的习惯。

---

## 一、系统调用部分

---

### fork

```c
int fork(void);
```

#### 作用

创建一个当前进程的子进程。

调用成功后，系统里会出现两个几乎一样的进程：

- 父进程：`fork()` 返回子进程的 pid
- 子进程：`fork()` 返回 `0`

调用失败返回 `-1`。

#### 怎么用

```c
int pid = fork();
if(pid < 0){
  printf("fork failed\n");
} else if(pid == 0){
  printf("child\n");
} else {
  printf("parent, child pid=%d\n", pid);
}
```

#### 核心理解

`fork` 的关键不是“从头启动一个新程序”，而是“复制当前进程”。

子进程会继承父进程的大部分内容，例如：

- 用户地址空间内容
- 打开的文件描述符
- 当前工作目录
- 寄存器状态的大部分信息

但它有自己独立的：

- `pid`
- 内核中的进程结构
- 后续执行路径

#### 为什么父子返回值不同

因为内核在复制进程时，会故意修改子进程保存的返回寄存器，让子进程看到 `fork()` 返回 `0`，而父进程看到子 pid。

这就是我们区分父子分支的依据。

#### 常见搭配

- `fork + wait`：父进程等待子进程结束
- `fork + exec`：子进程创建后立刻装载新程序
- `fork + pipe`：父子进程通过管道通信

#### 易错点

- `fork()` 之后，父子都会从 `fork` 后一行继续执行
- 父子共享“文件对象”，但不共享彼此的用户变量修改结果
- 如果父进程不 `wait`，退出后的子进程可能暂时成为僵尸进程

---

### exit

```c
int exit(int status) __attribute__((noreturn));
```

#### 作用

终止当前进程，并把退出状态 `status` 留给父进程回收。

`__attribute__((noreturn))` 表示这个函数不会返回。

#### 怎么用

```c
if(error)
  exit(1);
else
  exit(0);
```

#### 原理

内核在处理 `exit` 时会做几件事：

- 关闭当前进程打开的文件
- 释放大部分资源
- 唤醒可能正在 `wait()` 的父进程
- 把当前进程标记为 zombie，等待父进程回收

注意：进程并不是立刻“完全消失”，而是先变成僵尸，直到父进程 `wait()`。

#### 易错点

- `exit()` 之后的代码不会执行
- xv6-labs-2022 里 `exit` 有状态码参数，不要和旧版 xv6 的 `exit(void)` 混淆

---

### wait

```c
int wait(int *status);
```

#### 作用

等待一个子进程退出，并回收它的资源。

成功时返回退出子进程的 pid，并通过 `*status` 取回该子进程的退出码。
如果没有子进程可等，则返回 `-1`。

#### 怎么用

```c
int status;
int pid = wait(&status);
if(pid >= 0){
  printf("child %d exit status %d\n", pid, status);
}
```

如果不关心退出状态，也常见写法：

```c
wait(0);
```

#### 原理

父进程调用 `wait()` 后：

- 如果有已经退出的子进程，立即回收并返回
- 如果有子进程但都还没退出，父进程睡眠等待
- 如果根本没有子进程，返回 `-1`

#### 为什么需要 wait

因为子进程退出后仍要保留一小部分信息给父进程读取，比如：

- pid
- 退出状态

这些信息只有等父进程 `wait` 之后才能彻底释放。

---

### pipe

```c
int pipe(int *fd);
```

#### 作用

创建一条管道，供进程间按字节流通信。

成功后：

- `fd[0]`：读端
- `fd[1]`：写端

失败返回 `-1`。

#### 怎么用

```c
int p[2];
pipe(p);

if(fork() == 0){
  close(p[0]);
  write(p[1], "A", 1);
  close(p[1]);
  exit(0);
} else {
  char ch;
  close(p[1]);
  read(p[0], &ch, 1);
  printf("recv %c\n", ch);
  close(p[0]);
  wait(0);
}
```

#### 原理

管道本质上是内核中的一个缓冲区，加上两个文件对象：

- 一个只负责读
- 一个只负责写

进程通过文件描述符访问它。

#### 阻塞行为

- 管道空了时，`read()` 可能阻塞
- 管道满了时，`write()` 可能阻塞
- 如果所有写端都关闭了，读端再读会返回 `0`，表示 EOF

#### 易错点

- 父子进程都要及时关闭自己不用的那一端
- 不关闭多余端口会导致读端等不到 EOF

---

### write

```c
int write(int fd, const void *buf, int n);
```

#### 作用

向文件描述符 `fd` 写入 `n` 个字节。

成功返回实际写入的字节数，失败返回 `-1`。

#### 怎么用

```c
write(1, "hello\n", 6);
```

也可以写文件：

```c
int fd = open("a.txt", 0x002 | 0x200);
write(fd, "abc", 3);
close(fd);
```

这里的打开标志请以你实验代码中的 `fcntl.h` 为准，常见有：

- `O_RDONLY`
- `O_WRONLY`
- `O_RDWR`
- `O_CREATE`
- `O_TRUNC`

#### 原理

内核会根据 `fd` 找到对应的打开文件对象，再决定：

- 如果它是普通文件，就写入 inode 对应的数据块
- 如果它是管道，就写入管道缓冲区
- 如果它是设备文件，就转给设备驱动处理

#### 易错点

- `write` 按字节工作，不会自动追加字符串结束符 `\0`
- 参数 `n` 必须是你想写出的字节数

---

### read

```c
int read(int fd, void *buf, int n);
```

#### 作用

从文件描述符 `fd` 读取最多 `n` 个字节到 `buf`。

返回值含义：

- `> 0`：实际读取到的字节数
- `0`：读到文件末尾 EOF
- `-1`：读取失败

#### 怎么用

```c
char buf[100];
int n = read(0, buf, sizeof(buf));
if(n > 0)
  write(1, buf, n);
```

#### 原理

和 `write` 类似，内核先根据 `fd` 找到打开文件对象，再根据对象类型决定读取方式。

对于普通文件，会从当前文件偏移处读数据，并推进偏移量。

#### 易错点

- `read` 不会自动加 `\0`
- 如果把读到的数据当作字符串，常常需要手动补上结束符

例如：

```c
char buf[20];
int n = read(fd, buf, 19);
if(n >= 0){
  buf[n] = '\0';
}
```

---

### close

```c
int close(int fd);
```

#### 作用

关闭一个文件描述符。

成功返回 `0`，失败返回 `-1`。

#### 原理

关闭的是“当前进程的这个 fd 引用”。

内核会：

- 清掉进程打开文件表中的这一项
- 将底层文件对象的引用计数减一
- 如果引用计数归零，再真正释放底层对象

#### 易错点

- `close(fd)` 后不要再使用这个 `fd`
- 管道场景中，不关闭多余端会引发阻塞问题

---

### kill

```c
int kill(int pid);
```

#### 作用

向指定 pid 的进程发送“终止请求”。

成功返回 `0`，失败返回 `-1`。

#### 原理

xv6 的 `kill` 比 Unix 里的信号系统简单得多。它不是发送各种复杂 signal，而是给目标进程打上“已被杀死”的标记。

目标进程在：

- 从睡眠中被唤醒
- 再次回到内核

等时机会检查这个标记，然后退出。

#### 易错点

- `kill` 不一定让目标“立刻马上”消失
- 它依赖进程后续再次进入合适的内核路径

---

### exec

```c
int exec(const char *path, char **argv);
```

#### 作用

用一个新的程序替换当前进程的用户地址空间。

如果成功，`exec()` 不会返回；如果失败，返回 `-1`。

#### 怎么用

```c
char *argv[] = { "ls", 0 };
exec("ls", argv);
printf("exec failed\n");
```

#### 最核心的一句话

`exec` 不会创建新进程，它只是“把当前进程变成另一个程序”。

所以经典组合一定是：

```c
if(fork() == 0){
  exec("ls", argv);
  exit(1);
}
wait(0);
```

父进程先 `fork` 出子进程，子进程再 `exec` 新程序。

#### 原理

内核执行 `exec` 时会：

- 打开目标可执行文件
- 读取 ELF 头
- 按程序段把代码和数据装入新的用户地址空间
- 重新布置用户栈，把 `argv` 拷进去
- 设置程序计数器到入口地址

完成后，这个进程虽然 pid 没变，但它的用户程序内容已经完全换掉了。

#### 易错点

- `argv` 必须以空指针 `0` 结尾
- `exec` 成功后原来的用户代码不会继续执行
- `exec` 不会改 pid

---

### open

```c
int open(const char *path, int mode);
```

#### 作用

打开文件，成功返回文件描述符 fd，失败返回 `-1`。

#### 怎么用

```c
int fd = open("README", 0);
if(fd < 0){
  printf("open failed\n");
}
```

常见模式一般定义在 `fcntl.h` 中，例如：

- `O_RDONLY`：只读
- `O_WRONLY`：只写
- `O_RDWR`：可读可写
- `O_CREATE`：文件不存在则创建
- `O_TRUNC`：打开时截断

例如：

```c
int fd = open("out", O_CREATE | O_WRONLY);
```

#### 原理

内核需要做几件事：

- 解析路径
- 找到对应 inode
- 检查打开模式是否合法
- 分配一个 file 结构
- 在当前进程的打开文件表中分配一个 fd

#### 易错点

- `open` 返回的是 fd，不是 inode
- 打开成功后要记得 `close`

---

### mknod

```c
int mknod(const char *path, short major, short minor);
```

#### 作用

创建设备文件。

这是一个比较“内核/文件系统实验味”的接口，普通应用程序平时不常直接使用。

#### 原理

设备文件本身不存储普通数据，它更像一个“名字到设备驱动”的入口。

其中：

- `major`：主设备号，用来标识应该交给哪个设备驱动
- `minor`：次设备号，用来区分同类设备中的具体实例

#### 例子理解

控制台设备可能会通过某个设备号暴露成特殊文件。打开它之后，对它 `read/write`，最终会进入设备驱动，而不是普通磁盘文件逻辑。

#### 易错点

- `mknod` 不是创建普通文件用的
- 普通文件更常通过 `open(path, O_CREATE|...)` 创建

---

### unlink

```c
int unlink(const char *path);
```

#### 作用

删除一个目录项。

成功返回 `0`，失败返回 `-1`。

#### 原理

`unlink` 删除的是“名字到 inode 的链接”，不是简单粗暴地立刻清空文件数据。

如果某个文件：

- 链接数变成 0
- 并且没有进程还打开着它

那么内核才会最终释放这个文件占用的资源。

#### 易错点

- `unlink` 针对路径名
- 它和 `close` 完全不是一回事：`close` 关闭 fd，`unlink` 删除目录项

---

### fstat

```c
int fstat(int fd, struct stat *st);
```

#### 作用

获取一个已经打开文件的元信息，写入 `*st`。

成功返回 `0`，失败返回 `-1`。

#### 怎么用

```c
struct stat st;
if(fstat(fd, &st) == 0){
  printf("size = %d\n", st.size);
}
```

#### 原理

内核根据 `fd` 找到打开文件，再把它关联 inode 的信息拷贝到用户提供的 `struct stat` 中。

#### 适用场景

- 已经有了 fd，想查看文件大小或类型
- 不想重新根据路径查找文件

---

### link

```c
int link(const char *old, const char *new);
```

#### 作用

为已有文件创建一个新的目录项名字，也就是创建硬链接。

成功返回 `0`，失败返回 `-1`。

#### 原理

硬链接的本质是：让两个路径名指向同一个 inode。

创建之后：

- `old` 和 `new` 地位平等
- 删除其中一个名字，不影响另一个名字继续使用

只要 inode 的链接计数还大于 0，文件内容就仍然存在。

#### 易错点

- `link` 创建的是硬链接，不是复制文件内容
- 两个名字对应的是同一份底层数据

---

### mkdir

```c
int mkdir(const char *path);
```

#### 作用

创建一个目录。

成功返回 `0`，失败返回 `-1`。

#### 原理

内核会：

- 分配新的目录 inode
- 建立 `.` 和 `..`
- 在父目录中添加对应目录项

#### 易错点

- 父目录必须已经存在
- 一般不能用它来创建普通文件

---

### chdir

```c
int chdir(const char *path);
```

#### 作用

修改当前进程的工作目录。

成功返回 `0`，失败返回 `-1`。

#### 怎么理解工作目录

如果当前工作目录是 `/a`，那么：

```c
open("b", O_RDONLY);
```

实际打开的是 `/a/b`。

#### 原理

每个进程内核中都保存了一个“当前工作目录 inode”。`chdir` 就是把它切换到新的目录。

#### 易错点

- `chdir` 影响当前进程，子进程会继承这个工作目录
- 路径解析中，相对路径是相对当前工作目录而言

---

### dup

```c
int dup(int fd);
```

#### 作用

复制一个已有的文件描述符，返回新的 fd。

成功返回新 fd，失败返回 `-1`。

#### 怎么用

```c
int fd2 = dup(1);
write(fd2, "hello\n", 6);
close(fd2);
```

#### 原理

`dup` 会让新的 fd 和旧的 fd 指向同一个底层打开文件对象。

所以它们共享：

- 文件偏移量
- 打开模式
- 底层引用

但它们是两个不同的“描述符编号”。

#### 典型用途

重定向。

例如 shell 常见思路是先关闭标准输出，再 `dup` 一个文件 fd，使其占据最小可用编号，从而变成新的标准输出。

#### 易错点

- `dup` 不是重新打开文件
- 两个 fd 的读写偏移是共享的

---

### getpid

```c
int getpid(void);
```

#### 作用

获取当前进程的 pid。

成功时返回当前进程 id。

#### 用途

- 调试
- 打印进程信息
- 配合 `kill(pid)`

#### 原理

当前 CPU 正在运行哪个进程，内核是知道的，所以直接从当前进程结构中取出 pid 返回即可。

---

### sbrk

```c
char* sbrk(int n);
```

#### 作用

把当前进程的数据段边界向高地址或低地址移动 `n` 字节。

成功时返回“扩展前”的旧地址，失败返回 `(char*)-1`。

#### 最常见用途

给 `malloc()` 提供底层内存来源。

#### 怎么理解

每个进程都有自己的用户地址空间。`sbrk(n)` 的意思近似是：

- 如果 `n > 0`：申请更多堆空间
- 如果 `n < 0`：缩小堆空间

#### 例子

```c
char *p = sbrk(4096);
if(p == (char*)-1){
  printf("sbrk failed\n");
} else {
  // p 指向新拿到的一页起始处
}
```

#### 原理

内核会调整进程大小，并为新增虚拟地址分配物理页，再建立页表映射。

#### 易错点

- `sbrk` 返回的是旧的 program break，不是新的结束地址
- 它是比较底层的接口，日常更常通过 `malloc/free` 间接使用

---

### sleep

```c
int sleep(int ticks);
```

#### 作用

让当前进程睡眠若干个时钟滴答数 `ticks`。

成功返回 `0`，如果被 kill 等异常打断，常见实现会返回 `-1`。

#### 怎么用

```c
sleep(100);
```

#### 原理

xv6 内核维护一个全局 tick 计数，由时钟中断不断增加。`sleep(ticks)` 会记录当前 tick，然后阻塞当前进程，直到经过足够多的 tick。

#### 易错点

- 这里单位不是“毫秒”，而是“时钟滴答数”
- 精确时间长度取决于 xv6 的时钟频率

---

### uptime

```c
int uptime(void);
```

#### 作用

返回系统从启动以来经历的 tick 数。

#### 用途

- 简单计时
- 调试和性能粗测

#### 原理

本质上就是读内核维护的全局 tick 计数。

#### 配合 `sleep`

```c
int t0 = uptime();
sleep(100);
int t1 = uptime();
printf("elapsed = %d ticks\n", t1 - t0);
```

---

## 二、用户态库函数部分

这一部分大多数不是“系统调用本体”，而是普通 C 风格工具函数。

其中有一类很重要：

### `stat()` 其实是对 `open + fstat + close` 的封装思路

也就是说，`stat(path, &st)` 是根据路径拿文件信息；而 `fstat(fd, &st)` 是根据已经打开的 fd 拿文件信息。

---

### stat

```c
int stat(const char *path, struct stat *st);
```

#### 作用

根据路径获取文件元信息。

成功返回 `0`，失败返回 `-1`。

#### 可能的实现思路

它通常不是单独的系统调用，而是用户态库函数：

1. `open(path, O_RDONLY)`
2. `fstat(fd, st)`
3. `close(fd)`

#### 什么时候用

- 只有路径，没有 fd
- 想查看文件大小、类型等属性

---

### strcpy

```c
char* strcpy(char *dst, const char *src);
```

#### 作用

把 `src` 字符串复制到 `dst`，包括结尾的 `\0`。

返回 `dst`。

#### 注意

调用者必须保证 `dst` 空间足够大。

#### 例子

```c
char buf[20];
strcpy(buf, "abc");
```

---

### memmove

```c
void *memmove(void *dst, const void *src, int n);
```

#### 作用

复制 `n` 个字节，允许源和目标区域重叠。

返回 `dst`。

#### 和 memcpy 的区别

- `memmove`：可安全处理重叠
- `memcpy`：通常假设不重叠

#### 原理

当检测到目标区和源区重叠时，`memmove` 会选择合适方向复制，避免数据被提前覆盖。

---

### strchr

```c
char* strchr(const char *s, char c);
```

#### 作用

在字符串 `s` 中查找字符 `c` 第一次出现的位置。

找到则返回指针，找不到返回 `0`。

#### 例子

```c
char *p = strchr("abc:def", ':');
```

---

### strcmp

```c
int strcmp(const char *a, const char *b);
```

#### 作用

按字典序比较两个字符串。

返回值通常表示：

- `0`：相等
- `< 0`：`a < b`
- `> 0`：`a > b`

#### 用法

```c
if(strcmp(argv[1], "quit") == 0){
  exit(0);
}
```

---

### fprintf

```c
void fprintf(int fd, const char *fmt, ...);
```

#### 作用

按格式化字符串输出到指定文件描述符。

#### 例子

```c
fprintf(2, "error: open failed\n");
```

这通常用于把错误打印到标准错误 `fd=2`。

#### 原理

它是用户态格式化函数，内部整理好输出内容后，再调用 `write()` 发给内核。

---

### printf

```c
void printf(const char *fmt, ...);
```

#### 作用

向标准输出打印格式化内容，本质上可以理解成对 `fprintf(1, ...)` 的封装。

#### 例子

```c
printf("pid = %d\n", getpid());
```

#### 常见格式

- `%d`：十进制整数
- `%x`：十六进制
- `%p`：指针
- `%s`：字符串
- `%c`：字符

---

### gets

```c
char* gets(char *buf, int max);
```

#### 作用

从标准输入读取一行到 `buf` 中，最多读入 `max-1` 个字符，并补上 `\0`。

#### 例子

```c
char buf[100];
gets(buf, sizeof(buf));
```

#### 注意

现代 C 编程里通常不推荐 `gets` 风格接口，因为容易出问题；但 xv6 为了教学简化，保留了这个形式。

---

### strlen

```c
uint strlen(const char *s);
```

#### 作用

返回字符串长度，不包括末尾 `\0`。

#### 例子

```c
int n = strlen("abc");   // n = 3
```

---

### memset

```c
void* memset(void *dst, int c, uint n);
```

#### 作用

把 `dst` 开始的 `n` 个字节都设置为字节值 `c`。

返回 `dst`。

#### 例子

```c
char buf[100];
memset(buf, 0, sizeof(buf));
```

#### 常见用途

- 清零缓冲区
- 初始化结构体

---

### malloc

```c
void* malloc(uint n);
```

#### 作用

在堆上申请 `n` 字节内存，成功返回指针，失败返回 `0`。

#### 原理

xv6 用户态的 `malloc` 一般是一个简化的内存分配器。它内部维护空闲链表，当现有空闲块不够时，会调用 `sbrk()` 向内核申请更多地址空间。

也就是说：

- `malloc/free` 是用户态分配策略
- `sbrk` 是向内核要页的底层接口

#### 例子

```c
char *p = malloc(100);
if(p == 0){
  printf("malloc failed\n");
  exit(1);
}
```

---

### free

```c
void free(void *p);
```

#### 作用

释放由 `malloc` 返回的内存块。

#### 原理

在 xv6 的简化分配器里，`free` 一般是把该块放回空闲链表，供后续再次分配，不一定立刻把内存还给内核。

#### 易错点

- 只能释放 `malloc` 得到的指针
- 不要重复 `free`
- `free` 后不要继续使用这块内存

---

### atoi

```c
int atoi(const char *s);
```

#### 作用

把数字字符串转换成整数。

#### 例子

```c
int n = atoi("123");
```

#### 注意

它一般功能很简化，不像标准库那样处理特别复杂的输入和错误情况。

---

### memcmp

```c
int memcmp(const void *a, const void *b, uint n);
```

#### 作用

比较两块内存的前 `n` 个字节。

返回值风格与 `strcmp` 类似：

- `0`：相等
- `< 0`：前者较小
- `> 0`：前者较大

#### 适用场景

比较原始字节数据，而不是字符串。

---

### memcpy

```c
void *memcpy(void *dst, const void *src, uint n);
```

#### 作用

复制 `n` 个字节，通常要求源和目标区域不重叠。

返回 `dst`。

#### 和 memmove 的关系

如果你不确定会不会重叠，优先使用 `memmove` 更安全。

---

## 三、把这些接口串起来理解

单独记函数容易忘，真正要理解的是它们在 xv6 用户程序中的配合方式。

### 1. 启动一个新程序

典型流程：

```c
int pid = fork();
if(pid == 0){
  char *argv[] = { "echo", "hello", 0 };
  exec("echo", argv);
  exit(1);
}
wait(0);
```

这里体现了：

- `fork`：先复制出子进程
- `exec`：子进程装载新程序
- `wait`：父进程等待回收

### 2. 读写文件

典型流程：

```c
int fd = open("a.txt", O_RDONLY);
char buf[64];
int n = read(fd, buf, sizeof(buf));
close(fd);
```

这里体现了：

- `open`：获得 fd
- `read`：通过 fd 读数据
- `close`：释放 fd

### 3. 进程间通信

典型流程：

```c
int p[2];
pipe(p);
```

然后：

- 子进程持有写端
- 父进程持有读端
- 双方关闭不用的一端

### 4. 动态内存

典型理解链条：

```text
malloc/free
    ↓
用户态内存分配器
    ↓
sbrk
    ↓
内核扩展进程地址空间
```

---

## 四、学习这些接口时最应该抓住的几个“本质”

### 1. “系统调用”和“库函数”不是一回事

例如：

- `read` 是系统调用
- `printf` 是库函数，但内部会调用 `write`
- `stat` 是库函数，但内部可由 `open + fstat + close` 组合实现

### 2. 进程和程序不是一回事

- `fork`：产生新进程
- `exec`：把当前进程换成新程序

这是 xv6 里最核心的一组概念。

### 3. 文件、管道、设备都尽量统一成“fd”

这体现了 Unix 的一个经典思想：

“尽量把不同资源抽象成统一的读写接口。”

所以很多对象都能通过：

- `read`
- `write`
- `close`

来操作。

### 4. 内存分配有分层

- `malloc/free`：给程序员用
- `sbrk`：给分配器向内核要空间

---

## 五、容易混淆的接口对比

### `stat` 和 `fstat`

- `stat(path, &st)`：根据路径查看文件信息
- `fstat(fd, &st)`：根据已打开 fd 查看文件信息

### `memcpy` 和 `memmove`

- `memcpy`：一般假定不重叠
- `memmove`：即使重叠也安全

### `fork` 和 `exec`

- `fork`：复制当前进程
- `exec`：替换当前进程的程序映像

### `close` 和 `unlink`

- `close(fd)`：关闭一个文件描述符
- `unlink(path)`：删除一个路径名

---

## 六、建议你怎么记这份文件

可以按下面四组记：

### 进程管理

- `fork`
- `exit`
- `wait`
- `kill`
- `getpid`
- `exec`

### 文件与目录

- `open`
- `read`
- `write`
- `close`
- `fstat`
- `stat`
- `link`
- `unlink`
- `mkdir`
- `chdir`
- `mknod`
- `dup`

### 通信与时间

- `pipe`
- `sleep`
- `uptime`

### 内存与字符串工具

- `sbrk`
- `malloc`
- `free`
- `strcpy`
- `strcmp`
- `strlen`
- `memset`
- `memmove`
- `memcpy`
- `memcmp`
- `strchr`
- `atoi`
- `printf`
- `fprintf`
- `gets`

---

## 七、最后给你一个学习建议

学习 `user.h` 最好的方法不是只背函数原型，而是把它们放回 xv6 自带用户程序里观察。

建议你重点去看这些程序：

- `sh.c`：重点看 `fork/exec/wait/dup/open/close/pipe`
- `cat.c`：重点看 `read/write`
- `ls.c`：重点看 `open/fstat/stat/read`
- `mkdir.c`：重点看 `mkdir`
- `rm.c`：重点看 `unlink`
- `echo.c`：重点看 `printf`

这样你会很快发现：`user.h` 其实就是 xv6 用户世界的“工具箱总入口”。

---

## 八、一句话总结

`user.h` 的意义不是“列出了一堆函数”，而是定义了 xv6 用户程序能向内核请求哪些服务，以及用户态已经有哪些基础工具可用。

如果你把这份文件彻底吃透，后面做 xv6-labs 时，看到大多数用户程序，你都会知道它到底在做什么。
