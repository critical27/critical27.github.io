---
layout: single
title: explore x86_64 main(), part 1
date: 2025-05-05 00:00:00 +0800
categories: 学习
tags: Linux
---

看了之后“The Bits Between the Bits: How We Get to main()“，本来想按图索骥再学习一遍，没想到在我的环境下已经完全没法复现，于是才有了这篇文章。

## 相关环境

经过若干探索，最后我发现`main`之前的执行流程和很多因素有关，这也是我的环境观察到的很多现象和CppCon演讲中不一致的原因。ld和glibc均为2.39，gcc版本为13.3.0。

```bash
$ /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2  --version
ld.so (Ubuntu GLIBC 2.39-0ubuntu8.4) stable release version 2.39.

$ /lib/x86_64-linux-gnu/libc.so.6 --version
GNU C Library (Ubuntu GLIBC 2.39-0ubuntu8.4) stable release version 2.39

$ gcc --version
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
```

glibc是动态链接的

```nasm
$ file /lib/x86_64-linux-gnu/libc.so.6
/lib/x86_64-linux-gnu/libc.so.6: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=42c84c92e6f98126b3e2230ebfdead22c235b667, for GNU/Linux 3.2.0, stripped
```

这边就先用一个最简单的`main`函数为例（文件名为`main.cpp`）：

```cpp
int main() { }
```

采用以下的参数进行编译，可以看到是一个动态链接的可执行文件，接下来会在下文，按照我观察到的顺序，依次阐述`main`函数之前发生了什么事情：

```bash
$ g++ -O0 -ggdb -o main main.cpp
$ file main
main: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=99c9e60fc3108009c19d1c738da7545d6514301a, for GNU/Linux 3.2.0, with debug_info, not stripped
```

> 再次强调，本文介绍是动态链接的执行文件，静态链接的情况会完全不同，可能会单独整理。
>

glibc的源码可以通过如下形式获取

```bash
sudo apt install glibc-source
cd /usr/src/glibc
sudo tar xvf glibc-2.39.tar.xz
```

然后在gdb中加载

```bash
 dir /usr/src/glibc/glibc-2.39
```

## 内核阶段

每当执行一个可执行文件，内核会调用系统调用`execve()`。这个系统调用会完成以下工作：

1. 首先检查文件格式，读取ELF Header中的入口地址，即`readelf -h`中的Entry point address，也就是后文所说的用户可执行文件的`_start`

```bash
$ g++ -O0 -ggdb -o main main.cpp
$ readelf -h main
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1040
  Start of program headers:          64 (bytes into file)
  Start of section headers:          14456 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         35
  Section header string table index: 34
```

1. 然后加载ELF Header中的`LOAD`（代码段、数据段等）和`INTERP`（链接器）部分，可以使用`readelf -l`来查看

```bash
$ readelf -l main

Elf file type is DYN (Position-Independent Executable file)
Entry point 0x1040
There are 13 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000002d8 0x00000000000002d8  R      0x8
  INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x00000000000005f0 0x00000000000005f0  R      0x1000
  LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 0x0000000000000145 0x0000000000000145  R E    0x1000
  LOAD           0x0000000000002000 0x0000000000002000 0x0000000000002000
                 0x00000000000000c4 0x00000000000000c4  R      0x1000
  LOAD           0x0000000000002df0 0x0000000000003df0 0x0000000000003df0
                 0x0000000000000220 0x0000000000000228  RW     0x1000
  DYNAMIC        0x0000000000002e00 0x0000000000003e00 0x0000000000003e00
                 0x00000000000001c0 0x00000000000001c0  RW     0x8
  NOTE           0x0000000000000338 0x0000000000000338 0x0000000000000338
                 0x0000000000000030 0x0000000000000030  R      0x8
  NOTE           0x0000000000000368 0x0000000000000368 0x0000000000000368
                 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_PROPERTY   0x0000000000000338 0x0000000000000338 0x0000000000000338
                 0x0000000000000030 0x0000000000000030  R      0x8
  GNU_EH_FRAME   0x0000000000002004 0x0000000000002004 0x0000000000002004
                 0x000000000000002c 0x000000000000002c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000002df0 0x0000000000003df0 0x0000000000003df0
                 0x0000000000000210 0x0000000000000210  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn
   03     .init .plt .plt.got .text .fini
   04     .rodata .eh_frame_hdr .eh_frame
   05     .init_array .fini_array .dynamic .got .data .bss
   06     .dynamic
   07     .note.gnu.property
   08     .note.gnu.build-id .note.ABI-tag
   09     .note.gnu.property
   10     .eh_frame_hdr
   11
   12     .init_array .fini_array .dynamic .got
```

可以看到`LOAD`有四部分，分别对应02-05部分：

- `02`对应动态链接部分（只读）
- `03`对应代码段`.text`，用于动态链接解析的`.plt`和`.plt.got`，可读可执行
- `05`对应只读数据段：`.rodata`（如字符串常亮和全局常量等）
- `05`对应其他数据段：`.data`（初始化的全局变量和静态变量等）和`.bss`（未初始化的全局变量和静态变量等），可读可写

而`INTERP`会指向动态链接器`ld.so`，后续有一部分工作会由`ld.so`来完成。

1. 设置用户态栈，将`argc`、`argv`、`envp`和辅助信息`auxv`(ELF auxiliary vector)压栈，辅助信息下文会描述怎么查看

## ld.so

之后，动态链接器`ld.so`会被加载，它在跳转到用户可执行文件入口地址`_start`之前执行一些操作：

- 动态链接器bootstrap自举
- 预初始化：初始化可执行文件的相关参数，包括`argc`、`argv`和`envp`等
- 最终跳转到用户可执行文件的`_start`

### bootstrap

`ld.so`本身也是一个ELF程序，也需要自己的`_start`来初始化。所以自举阶段的核心任务是让`ld.so`在自身尚未完全加载的情况下，逐步构建出执行用户程序的环境，包括：

1. 解析内核传递的原始参数，如`argc`/`argv`/`envp`/`auxv`等，之后再传给相应的可执行文件。
2. 定位并加载自身的动态段`PT_DYNAMIC`，以支持后续符号解析。
3. 初始化关键数据结构，如`link_map`、符号表、TLS。
4. 加载用户可执行文件及其依赖库
5. 最终跳转到用户可执行文件入口

`ld.so`核心逻辑调用关系如下：

```bash
_start
  -> _dl_start
    -> _dl_start_final (返回用户可执行文件的入口)
      -> _dl_sysdep_start (执行下面的一系列操作 最终返回用户可执行文件的入口给ld)
        -> _dl_sysdep_parse_arguments (解析参数)
        -> dl_main
          -> process_envvars (处理环境变量)
          -> _dl_new_object (初始化一个空的link_map 作为用户可执行文件的主link_map)
          -> _dl_add_to_namespace_list (将用户可执行文件的main link_map加入到命名空间链表中)
          -> rtld_setup_main_map (根据ProgramHeaderTable 填充用户可执行文件的主link_map)
          -> _dl_map_object_deps (递归加载动态链接库)
          -> init_tls (初始化tls)
				  -> _dl_relocate_object (重定位动态链接库)
  -> _dl_init
  -> 跳转到用户可执行文件入口
```

注意，这里的`_start`不是用户可执行文件的入口地址`_start`，而是`ld.so`的入口。`ld.so`的`_start`不会直接调用`main`，而是跳转到用户可执行文件的入口地址，即用户程序的`_start`。

- `ld.so`的完整`_start`汇编代码如下

    ```nasm
    /* Initial entry point code for the dynamic linker.
       The C function `_dl_start' is the real entry point;
       its return value is the user program's entry point.  */
    #define RTLD_START asm ("\n\
    .text\n\
    	.align 16\n\
    .globl _start\n\
    .globl _dl_start_user\n\
    _start:\n\
    	movq %rsp, %rdi\n\
    	call _dl_start\n\
    _dl_start_user:\n\
    	# Save the user entry point address in %r12.\n\
    	movq %rax, %r12\n\
    	# Save %rsp value in %r13.\n\
    	movq %rsp, %r13\n\
    "\
    	RTLD_START_ENABLE_X86_FEATURES \
    "\
    	# Read the original argument count.\n\
    	movq (%rsp), %rdx\n\
    	# Call _dl_init (struct link_map *main_map, int argc, char **argv, char **env)\n\
    	# argc -> rsi\n\
    	movq %rdx, %rsi\n\
    	# And align stack for the _dl_init call. \n\
    	andq $-16, %rsp\n\
    	# _dl_loaded -> rdi\n\
    	movq _rtld_local(%rip), %rdi\n\
    	# env -> rcx\n\
    	leaq 16(%r13,%rdx,8), %rcx\n\
    	# argv -> rdx\n\
    	leaq 8(%r13), %rdx\n\
    	# Clear %rbp to mark outermost frame obviously even for constructors.\n\
    	xorl %ebp, %ebp\n\
    	# Call the function to run the initializers.\n\
    	call _dl_init\n\
    	# Pass our finalizer function to the user in %rdx, as per ELF ABI.\n\
    	leaq _dl_fini(%rip), %rdx\n\
    	# And make sure %rsp points to argc stored on the stack.\n\
    	movq %r13, %rsp\n\
    	# Jump to the user's entry point.\n\
    	jmp *%r12\n\
    .previous\n\
    ");
    ```


接下来我们分别看下三个主要步骤`_dl_start`、`_dl_init`和跳转到用户可执行文件入口是怎么完成的。

### _dl_start

`_dl_start`中一些关键函数：

- `_dl_sysdep_parse_arguments`从内核设置的栈中读取`argc`、`argv`、`envp`和`auxv`，以及Program Header表的地址`AT_PHDR`，Program Header的个数`AT_PHNUM`，入口地址`AT_ENTRY`等：

    ```cpp
    static void _dl_sysdep_parse_arguments(void **start_argptr, struct dl_main_arguments *args) {
        _dl_argc = (intptr_t)*start_argptr;
        _dl_argv = (char **)(start_argptr + 1); /* Necessary aliasing violation.  */
        _environ = _dl_argv + _dl_argc + 1;
        for (char **tmp = _environ;; ++tmp)
            if (*tmp == NULL) {
                /* Another necessary aliasing violation.  */
                GLRO(dl_auxv) = (ElfW(auxv_t) *)(tmp + 1);
                break;
            }

        dl_parse_auxv_t auxv_values = {0, };
        _dl_parse_auxv(GLRO(dl_auxv), auxv_values);

        args->phdr = (const ElfW(Phdr) *)auxv_values[AT_PHDR];
        args->phnum = auxv_values[AT_PHNUM];
        args->user_entry = auxv_values[AT_ENTRY];
    }

    ```

- 初始化可执行文件的主`link_map`，并设置ProgramHeaderTable的地址，以及可执行文件的入口地址`user_entry`

    ```cpp
          /* Create a link_map for the executable itself.
    	 This will be what dlopen on "" returns.  */
          main_map = _dl_new_object ((char *) "", "", lt_executable, NULL,
    				 __RTLD_OPENEXEC, LM_ID_BASE);
          assert (main_map != NULL);
          main_map->l_phdr = phdr;
          main_map->l_phnum = phnum;
          main_map->l_entry = *user_entry;

    ```

- `rtld_setup_main_map`会根据ProgramHeaderTable，填充可执行文件的主`link_map`
- `_dl_map_object_deps`会加载动态链接库，这部分不过多阐述，会在下一篇详细介绍

### _dl_init

`_dl_init`会将之前`ld.so`解析的`argc`、`argv`和`envp`再保存到几个全局变量中。这一步是在`_init_first`函数中完成的，调用栈如下：

```cpp
(gdb) bt
#0  _init_first (argc=1, argv=0x7fffffffe1c8, envp=0x7fffffffe1d8) at ./csu/init-first.c:46
#1  0x00007ffff7fca71f in call_init (l=<optimized out>, argc=argc@entry=1, argv=argv@entry=0x7fffffffe1c8, env=env@entry=0x7fffffffe1d8) at ./elf/dl-init.c:74
#2  0x00007ffff7fca824 in call_init (env=<optimized out>, argv=<optimized out>, argc=<optimized out>, l=<optimized out>) at ./elf/dl-init.c:120
#3  _dl_init (main_map=0x7ffff7ffe2e0, argc=1, argv=0x7fffffffe1c8, env=0x7fffffffe1d8) at ./elf/dl-init.c:121
#4  0x00007ffff7fe45a0 in _dl_start_user () from /lib64/ld-linux-x86-64.so.2
#5  0x0000000000000001 in ?? ()
#6  0x00007fffffffe472 in ?? ()
#7  0x0000000000000000 in ?? ()
```

相关的代码和汇编代码如下，可以看到在这保存了`argc`、`argv`和`envp`：

```cpp
static void __attribute__((constructor))
_init_first(int argc, char **argv, char **envp) {
    if (__libc_initial) {
        /* Set the FPU control word to the proper default value if the
       kernel would use a different value.  */
        if (__fpu_control != GLRO(dl_fpu_control))
            __setfpucw(__fpu_control);
    }

    /* Save the command-line arguments.  */
    __libc_argc = argc;
    __libc_argv = argv;
    __environ = envp;

    // ...

    __init_misc(argc, argv, envp);
}
```

```cpp
0x00007ffff7c2a060 <+0>:  endbr64
0x00007ffff7c2a064 <+4>:  push   %rbp
0x00007ffff7c2a065 <+5>:  mov    %rsp,%rbp
0x00007ffff7c2a068 <+8>:  push   %rbx                     ; 保存被调用者保存寄存器 %rbx
0x00007ffff7c2a069 <+9>:  mov    %edi,%ebx                ; 将第一个参数 argc 保存到 %ebx
0x00007ffff7c2a06b <+11>: sub    $0x18,%rsp

0x00007ffff7c2a06f <+15>: cmpb   $0x0,0x1e7d18(%rip)      ; 检查全局变量 __libc_initial 是否为0
0x00007ffff7c2a076 <+22>: je     0x7ffff7c2a08f           ; 如果为0，跳转到 <+47>

0x00007ffff7c2a078 <+24>: mov    0x1d8de1(%rip),%rax      ; FPU 检查
0x00007ffff7c2a07f <+31>: movzwl (%rax),%edi
0x00007ffff7c2a082 <+34>: mov    0x1d8e27(%rip),%rax
0x00007ffff7c2a089 <+41>: cmp    %di,0x58(%rax)
0x00007ffff7c2a08d <+45>: jne    0x7ffff7c2a0b3

0x00007ffff7c2a08f <+47>: mov    0x1d8f0a(%rip),%rax      ; 加载 __libc_argv 的地址到 %rax
0x00007ffff7c2a096 <+54>: mov    %ebx,0x1da64c(%rip)      ; 将 argc 存储到全局变量 __libc_argc
0x00007ffff7c2a09c <+60>: mov    %ebx,%edi
0x00007ffff7c2a09e <+62>: mov    %rsi,0x1da63b(%rip)      ; 将 argv 存储到全局变量 __libc_argv
0x00007ffff7c2a0a5 <+69>: mov    %rdx,(%rax)              ; 将 envp 存储到全局变量 __environ
0x00007ffff7c2a0a8 <+72>: add    $0x18,%rsp
0x00007ffff7c2a0ac <+76>: pop    %rbx
0x00007ffff7c2a0ad <+77>: pop    %rbp
0x00007ffff7c2a0ae <+78>: jmp    0x7ffff7298f0           ;  __init_misc 函数

0x00007ffff7c2a0b3 <+83>: mov    %rdx,-0x20(%rbp)
0x00007ffff7c2a0b7 <+87>: mov    %rsi,-0x18(%rbp)
0x00007ffff7c2a0bb <+91>: call   0x7ffff7c44e80
0x00007ffff7c2a0c0 <+96>: mov    -0x20(%rbp),%rdx
0x00007ffff7c2a0c4 <+100>: mov    -0x18(%rbp),%rsi
0x00007ffff7c2a0c8 <+104>: jmp    0x7ffff7c2a08f
```

### `ld.so`怎么跳转到用户可执行文件入口

当`ld.so`完成上述准备工作后，就可以准备跳转到用户可执行文件的入口的了。相关代码如下，省略掉无关部分，首先用户可执行文件入口地址是在`_dl_start_final`中返回的：

```cpp
static inline ElfW(Addr) __attribute__((always_inline)) _dl_start_final(void* arg) {
    ElfW(Addr) start_addr;

    /* Do not use an initializer for these members because it would
       interfere with __rtld_static_init.  */
    GLRO(dl_find_object) = &_dl_find_object;

    /* If it hasn't happen yet record the startup time.  */
    rtld_timer_start(&start_time);

    /* Transfer data about ourselves to the permanent link_map structure.  */
    _dl_setup_hash(&GL(dl_rtld_map));
    GL(dl_rtld_map).l_real = &GL(dl_rtld_map);
    GL(dl_rtld_map).l_map_start = (ElfW(Addr))&__ehdr_start;
    GL(dl_rtld_map).l_map_end = (ElfW(Addr))_end;

    /* Initialize the stack end variable.  */
    __libc_stack_end = __builtin_frame_address(0);

    /* Call the OS-dependent function to set up life so we can do things like
       file access.  It will call `dl_main' (below) to do all the real work
       of the dynamic linker, and then unwind our frame and run the user
       entry point on the same stack we entered on.  */
    start_addr = _dl_sysdep_start(arg, &dl_main);

    if (__glibc_unlikely (GLRO(dl_debug_mask) & DL_DEBUG_STATISTICS)) {
        RTLD_TIMING_VAR (rtld_total_time);
        rtld_timer_stop (&rtld_total_time, start_time);
        print_statistics (RTLD_TIMING_REF(rtld_total_time));
    }

    return ELF_MACHINE_START_ADDRESS(GL(dl_ns)[LM_ID_BASE]._ns_loaded, start_addr);
}
```

`_dl_sysdep_start`回之前的汇编代码如下：

```nasm
   # ...
   0x00007ffff7fe576e <+1438>:	add    $0x88,%rsp
   0x00007ffff7fe5775 <+1445>:	mov    %rbx,%rax
   0x00007ffff7fe5778 <+1448>:	pop    %rbx
   0x00007ffff7fe5779 <+1449>:	pop    %r12
   0x00007ffff7fe577b <+1451>:	pop    %r13
   0x00007ffff7fe577d <+1453>:	pop    %r14
   0x00007ffff7fe577f <+1455>:	pop    %r15
   0x00007ffff7fe5781 <+1457>:	pop    %rbp
=> 0x00007ffff7fe5782 <+1458>:	ret
```

在函数返回`ret`执行之前，`rip`和`rsp`的值如下：

```nasm
(gdb) p $rip
$6 = (void (*)()) 0x7ffff7fe5782 <_dl_start+1458>
(gdb) p $rsp
$7 = (void *) 0x7fffffffe1b8
(gdb) x $rsp
0x7fffffffe1b8:	0x00007ffff7fe4548
```

当前栈顶指向`0x7ffff7fe4548`，此时`rax`为用户可执行文件入口地址。执行`ret`后下一条指令就会跳转到`0x7ffff7fe4548`：

```nasm
(gdb) ni
0x00007ffff7fe4548 in _dl_start_user () from /lib64/ld-linux-x86-64.so.2
1: x/i $pc
=> 0x7ffff7fe4548 <_dl_start_user>:	mov    %rax,%r12
```

跳转后的汇编代码如下，`ld.so`先保存了用户可执行文件入口地址到`r12`中，执行了若干操作，比如上面提到的`_dl_init`，最终就无条件跳转到用户可执行文件入口地址：

```nasm
   # 将用户可执行文件入口地址保存到%r12中
=> 0x00007ffff7fe4548 <+0>:	mov    %rax,%r12
   0x00007ffff7fe454b <+3>:	mov    %rsp,%r13
   0x00007ffff7fe454e <+6>:	mov    0x19b14(%rip),%edx        # 0x7ffff7ffe068 <_rtld_global+4200>
   0x00007ffff7fe4554 <+12>:	test   $0x2,%edx
   0x00007ffff7fe455a <+18>:	je     0x7ffff7fe456d <_dl_start_user+37>
   0x00007ffff7fe455c <+20>:	mov    $0x1,%esi
   0x00007ffff7fe4561 <+25>:	mov    $0x5001,%edi
   0x00007ffff7fe4566 <+30>:	mov    $0x9e,%eax
   0x00007ffff7fe456b <+35>:	syscall

   0x00007ffff7fe456d <+37>:	mov    %edx,%edi
   0x00007ffff7fe456f <+39>:	and    $0xfffffffffffffff0,%rsp
   0x00007ffff7fe4573 <+43>:	call   0x7ffff7fdcb40 <_dl_cet_setup_features>
   0x00007ffff7fe4578 <+48>:	mov    %r12,%rax
   0x00007ffff7fe457b <+51>:	mov    %r13,%rsp
   0x00007ffff7fe457e <+54>:	mov    (%rsp),%rdx
   0x00007ffff7fe4582 <+58>:	mov    %rdx,%rsi
   0x00007ffff7fe4585 <+61>:	and    $0xfffffffffffffff0,%rsp
   0x00007ffff7fe4589 <+65>:	mov    0x18a70(%rip),%rdi        # 0x7ffff7ffd000 <_rtld_global>
   0x00007ffff7fe4590 <+72>:	lea    0x10(%r13,%rdx,8),%rcx
   0x00007ffff7fe4595 <+77>:	lea    0x8(%r13),%rdx
   0x00007ffff7fe4599 <+81>:	xor    %ebp,%ebp
   0x00007ffff7fe459b <+83>:	call   0x7ffff7fca780 <_dl_init>
   0x00007ffff7fe45a0 <+88>:	lea    -0x1a227(%rip),%rdx        # 0x7ffff7fca380 <_dl_fini>
   0x00007ffff7fe45a7 <+95>:	mov    %r13,%rsp

   # 注意在这跳转到用户可执行文件入口地址
   0x00007ffff7fe45aa <+98>:	jmp    *%r12
   0x00007ffff7fe45ad <+101>:	nopl   (%rax)
```

执行这个无条件跳转之后，就可以看到开始执行用户可执行文件的入口`_start`了

```nasm
(gdb) bt
#0  __libc_start_call_main (main=main@entry=0x401106 <main()>, argc=argc@entry=1, argv=argv@entry=0x7fffffffe1c8)
    at ../sysdeps/nptl/libc_start_call_main.h:29
#1  0x00007ffff7c2a28b in __libc_start_main_impl (main=0x401106 <main()>, argc=1, argv=0x7fffffffe1c8, init=<optimized out>, fini=<optimized out>,
    rtld_fini=<optimized out>, stack_end=0x7fffffffe1b8) at ../csu/libc-start.c:360
#2  0x0000000000401045 in _start ()
```

之后就开始执行用户的可执行文件了。

我们还可以在ld.so查看跳转到用户可执行文件的对应汇编代码的实际位置，跳转的代码在`0x00007ffff7fe45aa`，而ld.so的其实地址是`0x00007ffff7fc6000`，所以偏移量为`0x1e5aa`。

```nasm
(gdb) info sharedlibrary
From                To                  Syms Read   Shared Object Library
0x00007ffff7fc6000  0x00007ffff7ff0195  Yes         /lib64/ld-linux-x86-64.so.2
0x00007ffff7c28800  0x00007ffff7dafcb9  Yes         /lib/x86_64-linux-gnu/libc.so.6
(gdb) p/x 0x00007ffff7fe45aa - 0x00007ffff7fc6000
$2 = 0x1e5aa
```

readelf查看代码段所在位置，可以看到起始位置在`0x1000`：

```nasm
$ readelf -S /lib64/ld-linux-x86-64.so.2
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [11] .text             PROGBITS         0000000000001000  00001000
       000000000002a195  0000000000000000  AX       0     0     32
```

所以实际汇编代码是在代码段起始地址 `0x1000`+ 偏移量`0x1e5aa` = `0x1f5aa`处，我们通过objdump可以看到相应的代码是和上面gdb看到的汇编代码是一致的：

```nasm
$ objdump -dC /lib64/ld-linux-x86-64.so.2 --start-address=0x1f548 --stop-address=0x1f5b0

/lib64/ld-linux-x86-64.so.2:     file format elf64-x86-64

Disassembly of section .text:

000000000001f548 <_dl_mcount@@GLIBC_2.2.5+0x1598>:
   1f548:	49 89 c4             	mov    %rax,%r12
   1f54b:	49 89 e5             	mov    %rsp,%r13
   1f54e:	8b 15 14 9b 01 00    	mov    0x19b14(%rip),%edx        # 39068 <_rtld_global@@GLIBC_PRIVATE+0x1068>
   1f554:	f7 c2 02 00 00 00    	test   $0x2,%edx
   1f55a:	74 11                	je     1f56d <_dl_mcount@@GLIBC_2.2.5+0x15bd>
   1f55c:	be 01 00 00 00       	mov    $0x1,%esi
   1f561:	bf 01 50 00 00       	mov    $0x5001,%edi
   1f566:	b8 9e 00 00 00       	mov    $0x9e,%eax
   1f56b:	0f 05                	syscall
   1f56d:	89 d7                	mov    %edx,%edi
   1f56f:	48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
   1f573:	e8 c8 85 ff ff       	call   17b40 <__tls_get_addr@@GLIBC_2.3+0x320>
   1f578:	4c 89 e0             	mov    %r12,%rax
   1f57b:	4c 89 ec             	mov    %r13,%rsp
   1f57e:	48 8b 14 24          	mov    (%rsp),%rdx
   1f582:	48 89 d6             	mov    %rdx,%rsi
   1f585:	48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
   1f589:	48 8b 3d 70 8a 01 00 	mov    0x18a70(%rip),%rdi        # 38000 <_rtld_global@@GLIBC_PRIVATE>
   1f590:	49 8d 4c d5 10       	lea    0x10(%r13,%rdx,8),%rcx
   1f595:	49 8d 55 08          	lea    0x8(%r13),%rdx
   1f599:	31 ed                	xor    %ebp,%ebp
   1f59b:	e8 e0 61 fe ff       	call   5780 <__nptl_change_stack_perm@@GLIBC_PRIVATE+0x1490>
   1f5a0:	48 8d 15 d9 5d fe ff 	lea    -0x1a227(%rip),%rdx        # 5380 <__nptl_change_stack_perm@@GLIBC_PRIVATE+0x1090>
   1f5a7:	4c 89 ec             	mov    %r13,%rsp
   1f5aa:	41 ff e4             	jmp    *%r12
   1f5ad:	0f 1f 00             	nopl   (%rax)
```

## 可执行文件入口`_start`

可执行文件入口`_start`一段由汇编写的代码，在`sysdeps/x86_64/start.S`中，它主要完成的工作包括：

- 从相关寄存器获取相关信息，包括`argc`，`argv`，`init`，`fini`，`rtld_fini`等，为调用`__libc_start_main`做准备
- 将`rsp`按16字节对齐
- 调用`__libc_start_main`

```nasm

#include <sysdep.h>

ENTRY (_start)
	/* Clearing frame pointer is insufficient, use CFI.  */
	cfi_undefined (rip)
	/* Clear the frame pointer.  The ABI suggests this be done, to mark
	   the outermost frame obviously.  */
	xorl %ebp, %ebp

	/* Extract the arguments as encoded on the stack and set up
	   the arguments for __libc_start_main (int (*main) (int, char **, char **),
		   int argc, char *argv,
		   void (*init) (void), void (*fini) (void),
		   void (*rtld_fini) (void), void *stack_end).
	   The arguments are passed via registers and on the stack:
	main:		%rdi
	argc:		%rsi
	argv:		%rdx
	init:		%rcx
	fini:		%r8
	rtld_fini:	%r9
	stack_end:	stack.	*/

	mov %RDX_LP, %R9_LP	/* Address of the shared library termination
				   function.  */
#ifdef __ILP32__
	mov (%rsp), %esi	/* Simulate popping 4-byte argument count.  */
	add $4, %esp
#else
	popq %rsi		/* Pop the argument count.  */
#endif
	/* argv starts just at the current stack top.  */
	mov %RSP_LP, %RDX_LP
	/* Align the stack to a 16 byte boundary to follow the ABI.  */
	and  $~15, %RSP_LP

	/* Push garbage because we push 8 more bytes.  */
	pushq %rax

	/* Provide the highest stack address to the user code (for stacks
	   which grow downwards).  */
	pushq %rsp

	/* These used to be the addresses of .fini and .init.  */
	xorl %r8d, %r8d
	xorl %ecx, %ecx

#ifdef PIC
	mov main@GOTPCREL(%rip), %RDI_LP
#else
	mov $main, %RDI_LP
#endif

	/* Call the user's main function, and exit with its value.
	   But let the libc call main.  Since __libc_start_main in
	   libc.so is called very early, lazy binding isn't relevant
	   here.  Use indirect branch via GOT to avoid extra branch
	   to PLT slot.  In case of static executable, ld in binutils
	   2.26 or above can convert indirect branch into direct
	   branch.  */
	call *__libc_start_main@GOTPCREL(%rip)

	hlt			/* Crash if somehow `exit' does return.	 */
END (_start)
```

### 可执行文件入口地址

我们可以再从用户可执行文件的视角看看从`ld.so`跳转到可执行文件入口地址的这个过程。通过`objdump`能看到可执行文件入口地址在`0x1040`：

```bash
$ g++ -O0 -ggdb -o main main.cpp
$ objdump -dC main

main:     file format elf64-x86-64

Disassembly of section .init:

0000000000001000 <_init>:
    1000:	f3 0f 1e fa          	endbr64
    1004:	48 83 ec 08          	sub    $0x8,%rsp
    1008:	48 8b 05 d9 2f 00 00 	mov    0x2fd9(%rip),%rax        # 3fe8 <__gmon_start__@Base>
    100f:	48 85 c0             	test   %rax,%rax
    1012:	74 02                	je     1016 <_init+0x16>
    1014:	ff d0                	call   *%rax
    1016:	48 83 c4 08          	add    $0x8,%rsp
    101a:	c3                   	ret

Disassembly of section .plt:

0000000000001020 <.plt>:
    1020:	ff 35 a2 2f 00 00    	push   0x2fa2(%rip)        # 3fc8 <_GLOBAL_OFFSET_TABLE_+0x8>
    1026:	ff 25 a4 2f 00 00    	jmp    *0x2fa4(%rip)        # 3fd0 <_GLOBAL_OFFSET_TABLE_+0x10>
    102c:	0f 1f 40 00          	nopl   0x0(%rax)

Disassembly of section .plt.got:

0000000000001030 <__cxa_finalize@plt>:
    1030:	f3 0f 1e fa          	endbr64
    1034:	ff 25 be 2f 00 00    	jmp    *0x2fbe(%rip)        # 3ff8 <__cxa_finalize@GLIBC_2.2.5>
    103a:	66 0f 1f 44 00 00    	nopw   0x0(%rax,%rax,1)

Disassembly of section .text:

0000000000001040 <_start>:
    1040:	f3 0f 1e fa          	endbr64
    1044:	31 ed                	xor    %ebp,%ebp
    1046:	49 89 d1             	mov    %rdx,%r9
    1049:	5e                   	pop    %rsi
    104a:	48 89 e2             	mov    %rsp,%rdx
    104d:	48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
    1051:	50                   	push   %rax
    1052:	54                   	push   %rsp
    1053:	45 31 c0             	xor    %r8d,%r8d
    1056:	31 c9                	xor    %ecx,%ecx
    1058:	48 8d 3d ca 00 00 00 	lea    0xca(%rip),%rdi        # 1129 <main>
    105f:	ff 15 73 2f 00 00    	call   *0x2f73(%rip)        # 3fd8 <__libc_start_main@GLIBC_2.34>
    1065:	f4                   	hlt
    1066:	66 2e 0f 1f 84 00 00 	cs nopw 0x0(%rax,%rax,1)
    106d:	00 00 00

// ...
```

我们刚才提到内核会传一个辅助信息`auxv` (ELF auxiliary vector)，内核会传一些信息给可执行文件。我们可以通过设置`LD_SHOW_AUXV=1`来查看这些辅助信息，注意到`AT_ENTRY`的地址`0x5ec96d924040`，也就是入口地址，这和刚才通过`objdump`看到的`<_start>`地址`0000000000001040`不同，这是为什么？

```bash
$ LD_SHOW_AUXV=1 ./main

AT_SYSINFO_EHDR:      0x7ffe6ddb9000
AT_MINSIGSTKSZ:       3632
AT_HWCAP:             bfebfbff
AT_PAGESZ:            4096
AT_CLKTCK:            100
AT_PHDR:              0x5ec96d923040
AT_PHENT:             56
AT_PHNUM:             13
AT_BASE:              0x7a53c142c000
AT_FLAGS:             0x0
AT_ENTRY:             0x5ec96d924040
AT_UID:               1004
AT_EUID:              1004
AT_GID:               1004
AT_EGID:              1004
AT_SECURE:            0
AT_RANDOM:            0x7ffe6ddaa589
AT_HWCAP2:            0x2
AT_EXECFN:            ./main
AT_PLATFORM:          x86_64
AT_??? (0x1b): 0x1c
AT_??? (0x1c): 0x20
```

这是因为Linux默认启用 ASLR（Address Space Layout Randomization），程序每次加载时，其代码段`.text`、堆、栈、动态链接库等会被随机分配到不同的虚拟内存地址，以提高安全性。

- 通过`LD_SHOW_AUXV=1`看到的`AT_ENTRY`是程序运行时实际加载的入口地址（已加上 ASLR 随机偏移）。
- 通过`objdump -d`看到的是静态编译时的相对地址（未考虑 ASLR 偏移）。

内核使用ASLR的前提是编译器将可执行文件编译为PIE `(Position-Independent Executable)`。注意下面`file`结果中的`ELF 64-bit LSB pie executable`：

```bash
$ file main
main: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=99c9e60fc3108009c19d1c738da7545d6514301a, for GNU/Linux 3.2.0, with debug_info, not stripped
```

我们可以试下关掉pie来看下入口地址是否匹配

```bash
$ g++ -no-pie -O0 -ggdb -o main main.cpp
$ file main
main: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=0635e0888701e687eff83e277ec1070e324cde3d, for GNU/Linux 3.2.0, with debug_info, not stripped
```

可以看到在非PIE可执行文件中，默认加载地址都是固定加载地址`0x400000`。x86-64选择`0x400000`作为非PIE可执行文件的默认加载地址，低地址`0x000000 - 0x400000`通常用于保护空指针访问或保留给内核。

```bash
$ objdump -dC main

main:     file format elf64-x86-64

Disassembly of section .init:

0000000000401000 <_init>:
  401000:	f3 0f 1e fa          	endbr64
  401004:	48 83 ec 08          	sub    $0x8,%rsp
  401008:	48 8b 05 d1 2f 00 00 	mov    0x2fd1(%rip),%rax        # 403fe0 <__gmon_start__@Base>
  40100f:	48 85 c0             	test   %rax,%rax
  401012:	74 02                	je     401016 <_init+0x16>
  401014:	ff d0                	call   *%rax
  401016:	48 83 c4 08          	add    $0x8,%rsp
  40101a:	c3                   	ret

Disassembly of section .text:

0000000000401020 <_start>:
  401020:	f3 0f 1e fa          	endbr64
  401024:	31 ed                	xor    %ebp,%ebp
  401026:	49 89 d1             	mov    %rdx,%r9
  401029:	5e                   	pop    %rsi
  40102a:	48 89 e2             	mov    %rsp,%rdx
  40102d:	48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
  401031:	50                   	push   %rax
  401032:	54                   	push   %rsp
  401033:	45 31 c0             	xor    %r8d,%r8d
  401036:	31 c9                	xor    %ecx,%ecx
  401038:	48 c7 c7 06 11 40 00 	mov    $0x401106,%rdi
  40103f:	ff 15 93 2f 00 00    	call   *0x2f93(%rip)        # 403fd8 <__libc_start_main@GLIBC_2.34>
  401045:	f4                   	hlt
  401046:	66 2e 0f 1f 84 00 00 	cs nopw 0x0(%rax,%rax,1)
  40104d:	00 00 00

0000000000401050 <_dl_relocate_static_pie>:
  401050:	f3 0f 1e fa          	endbr64
  401054:	c3                   	ret
  401055:	66 2e 0f 1f 84 00 00 	cs nopw 0x0(%rax,%rax,1)
  40105c:	00 00 00
  40105f:	90                   	nop

0000000000401060 <deregister_tm_clones>:
  401060:	b8 10 40 40 00       	mov    $0x404010,%eax
  401065:	48 3d 10 40 40 00    	cmp    $0x404010,%rax
  40106b:	74 13                	je     401080 <deregister_tm_clones+0x20>
  40106d:	b8 00 00 00 00       	mov    $0x0,%eax
  401072:	48 85 c0             	test   %rax,%rax
  401075:	74 09                	je     401080 <deregister_tm_clones+0x20>
  401077:	bf 10 40 40 00       	mov    $0x404010,%edi
  40107c:	ff e0                	jmp    *%rax
  40107e:	66 90                	xchg   %ax,%ax
  401080:	c3                   	ret
  401081:	66 66 2e 0f 1f 84 00 	data16 cs nopw 0x0(%rax,%rax,1)
  401088:	00 00 00 00
  40108c:	0f 1f 40 00          	nopl   0x0(%rax)

0000000000401090 <register_tm_clones>:
  401090:	be 10 40 40 00       	mov    $0x404010,%esi
  401095:	48 81 ee 10 40 40 00 	sub    $0x404010,%rsi
  40109c:	48 89 f0             	mov    %rsi,%rax
  40109f:	48 c1 ee 3f          	shr    $0x3f,%rsi
  4010a3:	48 c1 f8 03          	sar    $0x3,%rax
  4010a7:	48 01 c6             	add    %rax,%rsi
  4010aa:	48 d1 fe             	sar    $1,%rsi
  4010ad:	74 11                	je     4010c0 <register_tm_clones+0x30>
  4010af:	b8 00 00 00 00       	mov    $0x0,%eax
  4010b4:	48 85 c0             	test   %rax,%rax
  4010b7:	74 07                	je     4010c0 <register_tm_clones+0x30>
  4010b9:	bf 10 40 40 00       	mov    $0x404010,%edi
  4010be:	ff e0                	jmp    *%rax
  4010c0:	c3                   	ret
  4010c1:	66 66 2e 0f 1f 84 00 	data16 cs nopw 0x0(%rax,%rax,1)
  4010c8:	00 00 00 00
  4010cc:	0f 1f 40 00          	nopl   0x0(%rax)

00000000004010d0 <__do_global_dtors_aux>:
  4010d0:	f3 0f 1e fa          	endbr64
  4010d4:	80 3d 35 2f 00 00 00 	cmpb   $0x0,0x2f35(%rip)        # 404010 <__TMC_END__>
  4010db:	75 13                	jne    4010f0 <__do_global_dtors_aux+0x20>
  4010dd:	55                   	push   %rbp
  4010de:	48 89 e5             	mov    %rsp,%rbp
  4010e1:	e8 7a ff ff ff       	call   401060 <deregister_tm_clones>
  4010e6:	c6 05 23 2f 00 00 01 	movb   $0x1,0x2f23(%rip)        # 404010 <__TMC_END__>
  4010ed:	5d                   	pop    %rbp
  4010ee:	c3                   	ret
  4010ef:	90                   	nop
  4010f0:	c3                   	ret
  4010f1:	66 66 2e 0f 1f 84 00 	data16 cs nopw 0x0(%rax,%rax,1)
  4010f8:	00 00 00 00
  4010fc:	0f 1f 40 00          	nopl   0x0(%rax)

0000000000401100 <frame_dummy>:
  401100:	f3 0f 1e fa          	endbr64
  401104:	eb 8a                	jmp    401090 <register_tm_clones>

0000000000401106 <main>:
  401106:	f3 0f 1e fa          	endbr64
  40110a:	55                   	push   %rbp
  40110b:	48 89 e5             	mov    %rsp,%rbp
  40110e:	b8 00 00 00 00       	mov    $0x0,%eax
  401113:	5d                   	pop    %rbp
  401114:	c3                   	ret

Disassembly of section .fini:

0000000000401118 <_fini>:
  401118:	f3 0f 1e fa          	endbr64
  40111c:	48 83 ec 08          	sub    $0x8,%rsp
  401120:	48 83 c4 08          	add    $0x8,%rsp
  401124:	c3                   	ret
```

这时候`AT_ENTRY`的地址和`<_start>`地址就相同了，都是`0x401020`：

```bash
$ LD_SHOW_AUXV=1 ./main
AT_SYSINFO_EHDR:      0x7ffe2d4e2000
AT_MINSIGSTKSZ:       3632
AT_HWCAP:             bfebfbff
AT_PAGESZ:            4096
AT_CLKTCK:            100
AT_PHDR:              0x400040
AT_PHENT:             56
AT_PHNUM:             13
AT_BASE:              0x7dcb89532000
AT_FLAGS:             0x0
AT_ENTRY:             0x401020
AT_UID:               1004
AT_EUID:              1004
AT_GID:               1004
AT_EGID:              1004
AT_SECURE:            0
AT_RANDOM:            0x7ffe2d480da9
AT_HWCAP2:            0x2
AT_EXECFN:            ./main
AT_PLATFORM:          x86_64
AT_??? (0x1b): 0x1c
AT_??? (0x1c): 0x20
```

其中有一些信息值得注意：

- `AT_ENTRY`是程序的入口地址，即 `_start` 函数的地址
- `AT_PHDR`是Program Header Table的虚拟地址，即`Elf64_Phdr` 数组的起始地址，Program Headers描述可执行文件或共享库各个内存段信息。
- `AT_PHENT`是单个Program Header条目的大小。

另外注意到：

- no-pie的情况下，objdump结果中没有`.plt`和`.plt.got`
- 而pie的情况下，objdump结果中有`.plt`和`.plt.got`

这两个section跟动态链接解析相关的。简单来说，`.plt`保存有跳转到`got`的stub代码，`.plt.got`会保存动态解析后的函数地址。如果指定了no-pie，由于我们只有一个源文件，且没有使用动态链接库，因此在no-pie可执行文件中目前是不会出现这两个section。而pie可执行文件要求所有代码必须是位置无关的，因此外部函数调用不能硬编码地址（比如ASLR会使库加载地址随机化）。并且如果存在动态链接库，必须通过`PLT(Procedure Linkage Table)`进行延迟绑定。

> PIE和我们在编译动态链接库时指定的PIC（Position-Independent Code）都是位置无关代码的实现方式，却别在于PIE只是用于可执行文件，即主程序的，而PIC用于动态链接库(.so)。动态库必须用`-fPIC`，因为多个进程可能加载同一库到不同地址。而PIE是PIC之后才出现的，原因是早期可执行文件默认用固定地址作为加载地址，后来为了增强安全性，引入PIE使主程序也能随机化。
>

### __libc_start_main

```cpp
$ ldd main
	linux-vdso.so.1 (0x00007ffddfbc8000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x000072e592c00000)
	/lib64/ld-linux-x86-64.so.2 (0x000072e592f97000)

$ file /lib/x86_64-linux-gnu/libc.so.6
/lib/x86_64-linux-gnu/libc.so.6: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=42c84c92e6f98126b3e2230ebfdead22c235b667, for GNU/Linux 3.2.0, stripped
```

在glibc2.39和动态链接环境下，`_start`仍然是用户程序的入口。但由于`ld.so`已经进行了很多工作，可执行文件的`__libc_start_main`负责的工作比老版本的glibc少了很多：

- 注册动态链接器的退出函数`rtld_fini`
- 通过`call_init`调用全局构造函数
- 通过`__libc_start_call_main`，进而调用`main`函数

对应的`__libc_start_main`代码如下（省略掉静态链接的相关代码）：

```cpp
/* Note: The init and fini parameters are no longer used.  fini is
   completely unused, init is still called if not NULL, but the
   current startup code always passes NULL.  (In the future, it would
   be possible to use fini to pass a version code if init is NULL, to
   indicate the link-time glibc without introducing a hard
   incompatibility for new programs with older glibc versions.)

   For dynamically linked executables, the dynamic segment is used to
   locate constructors and destructors.  For statically linked
   executables, the relevant symbols are access directly.  */
STATIC int LIBC_START_MAIN(int (*main)(int, char**, char** MAIN_AUXVEC_DECL),
                           int argc,
                           char** argv,
#ifdef LIBC_START_MAIN_AUXVEC_ARG
                           ElfW(auxv_t) * auxvec,
#endif
                           __typeof(main) init,
                           void (*fini)(void),
                           void (*rtld_fini)(void),
                           void* stack_end) {
    if (__glibc_likely(rtld_fini != NULL))
        __cxa_atexit((void (*)(void*))rtld_fini, NULL, NULL);

    if (__builtin_expect(GLRO(dl_debug_mask) & DL_DEBUG_IMPCALLS, 0))
        GLRO(dl_debug_printf)("\ninitialize program: %s\n\n", argv[0]);

    if (init != NULL)
        /* This is a legacy program which supplied its own init
           routine.  */
        (*init)(argc, argv, __environ MAIN_AUXVEC_PARAM);
    else
        /* This is a current program.  Use the dynamic segment to find
           constructors.  */
        call_init(argc, argv, __environ);

    /* Auditing checkpoint: we have a new object.  */
    _dl_audit_preinit(GL(dl_ns)[LM_ID_BASE]._ns_loaded);

    if (__glibc_unlikely(GLRO(dl_debug_mask) & DL_DEBUG_IMPCALLS))
        GLRO(dl_debug_printf)("\ntransferring control: %s\n\n", argv[0]);

    __libc_start_call_main(main, argc, argv MAIN_AUXVEC_PARAM);
}
```

这篇文章就到这结束，否则篇幅太长了。好在我们离main已经很近了，下一篇我们将从`__libc_start_main`开始，继续我们的`main`函数之旅。

## Reference

[CppCon 2018: Matt Godbolt “The Bits Between the Bits: How We Get to main()”](https://www.youtube.com/watch?v=dOfucXtyEsU)

[Linux x86 Program Start Up](http://dbp-consulting.com/tutorials/debugging/linuxProgramStartup.html)