这篇笔记我们来了解一下`user.h`这个文件中，系统为我们写好的功能接口。
## 源文件
首先来看一下`user.h`文件：
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
接下来，我们将慢慢了解其中的每一个接口。

---
## 系统调用

### fork
fork是开一个新的进程