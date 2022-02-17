> 环境准备

1、首先根据内核版本安装对应的kernel debuginfo

debuginfo的网址链接:http://debuginfo.centos.org/7/x86_64/

我的环境是Linux localhost.localdomain 3.10.0-1160.49.1.el7.x86_64 #1 SMP Tue Nov 30 15:51:32 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux

2、安装配置kdump网上搜下就ok了

3、加载云影，如果dump了在/var/crash目录下会产生一个根据ip+日期命名的目录，在该目录下会有两个文件。

vmcore这个是在系统dump的时候保存的崩溃时的内存+寄存器信息的core文件。

vmcore-dmesg.txt是以文本形式保存的崩溃时候的堆栈信息以及崩溃原因。


> 分析过程
- 查看文件vmcore-dmesg.txt，拉到末尾

```bash
[ 5227.336959] akfs: loading out-of-tree module taints kernel.
[ 5227.337415] akfs: module verification failed: signature and/or required key missing - tainting kernel
[ 5234.427590] BUG: unable to handle kernel NULL pointer dereference at           (null)
[ 5234.427706] IP: [<          (null)>]           (null)
[ 5234.427899] PGD 8000000003850067 PUD 1ef04067 PMD 0
[ 5234.428096] Oops: 0010 [#1] SMP

[ 5234.429703] CPU: 1 PID: 2227 Comm: bash Kdump: loaded Tainted: G           OE  ------------   3.10.0-1160.49.1.el7.x86_64 #1
[ 5234.429944] Hardware name: Red Hat KVM, BIOS 0.5.1 01/01/2011
[ 5234.430136] task: ffff97c7df2ea100 ti: ffff97c7df298000 task.ti: ffff97c7df298000
[ 5234.430368] RIP: 0010:[<0000000000000000>]  [<          (null)>]           (null)
[ 5234.430599] RSP: 0018:ffff97c7df29ae28  EFLAGS: 00010292
[ 5234.430738] RAX: 0000000000000000 RBX: ffff97c7df29ae30 RCX: 0000000000000000
[ 5234.430934] RDX: 0000000000000040 RSI: ffff97c7df29aea0 RDI: ffff97c7d29b5e00
[ 5234.431137] RBP: ffff97c7df29be50 R08: 000000000000ffff R09: 000000000000ffff
[ 5234.431388] R10: ffff97c7df0121b0 R11: 0000000000000007 R12: ffff97c7d29b5c00
[ 5234.431593] R13: ffff97c7df2ea100 R14: 00000000000008b3 R15: ffff97c7df2ea100
[ 5234.431798] FS:  00007f43b3713740(0000) GS:ffff97c7dfc80000(0000) knlGS:0000000000000000
[ 5234.432011] CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
[ 5234.432201] CR2: 0000000000000000 CR3: 0000000003836000 CR4: 00000000000006e0
[ 5234.432425] Call Trace:
[ 5234.432615]  [<ffffffffc04e83d8>] ? process_bprm_put+0xf8/0x130 [process]
[ 5234.432800]  [<ffffffff98b08a85>] ? security_bprm_check+0x5/0x30
[ 5234.432999]  [<ffffffffc04e8756>] __akfs_trace_security_bprm_check+0x36/0x60 [process]
[ 5234.433234]  [<ffffffff98a5587a>] search_binary_handler+0x2a/0x1c0
[ 5234.433453]  [<ffffffff98a56dd6>] do_execve_common.isra.23+0x616/0x880
[ 5234.433637]  [<ffffffff98a572e9>] SyS_execve+0x29/0x30
[ 5234.433816]  [<ffffffff98f96538>] stub_execve+0x48/0x80
[ 5234.433985] Code:  Bad RIP value.
[ 5234.434159] RIP  [<          (null)>]           (null)
[ 5234.434364]  RSP <ffff97c7df29ae28>
[ 5234.434516] CR2: 0000000000000000
```
从这个日志上直观看，空指针错误导致了dump。

- 接下来启用crash分析一波。

crash vmlinux路径 vmcore路径

```
 KERNEL: /usr/local/src/vmlinux            
    DUMPFILE: /var/crash/127.0.0.1-2021-12-06-13:47:24/vmcore  [PARTIAL DUMP]
        CPUS: 4
        DATE: Mon Dec  6 21:46:56 2021
      UPTIME: 01:27:13
LOAD AVERAGE: 0.24, 0.08, 0.06
       TASKS: 136
    NODENAME: localhost.localdomain
     RELEASE: 3.10.0-1160.49.1.el7.x86_64
     VERSION: #1 SMP Tue Nov 30 15:51:32 UTC 2021
     MACHINE: x86_64  (1496 Mhz)
      MEMORY: 511.6 MB
       PANIC: "BUG: unable to handle kernel NULL pointer dereference at           (null)"
         PID: 2227
     COMMAND: "bash"
        TASK: ffff97c7df2ea100  [THREAD_INFO: ffff97c7df298000]
         CPU: 1
       STATE: TASK_RUNNING (PANIC)
```
这是加载crash之后显示的信息，从这可以看出在运行 COMMAND: "bash" pid=2227的时候内核崩溃了，
崩溃原因是 PANIC: "BUG: unable to handle kernel NULL pointer dereference at           (null)"


- bt 2227 打印出崩溃时的堆栈信息

```
crash> bt 2227
PID: 2227   TASK: ffff97c7df2ea100  CPU: 1   COMMAND: "bash"
 #0 [ffff97c7df29aa68] machine_kexec at ffffffff988662c4
 #1 [ffff97c7df29aac8] __crash_kexec at ffffffff98922a32
 #2 [ffff97c7df29ab98] crash_kexec at ffffffff98922b20
 #3 [ffff97c7df29abb0] oops_end at ffffffff98f8d798
 #4 [ffff97c7df29abd8] no_context at ffffffff98875d14
 #5 [ffff97c7df29ac28] __bad_area_nosemaphore at ffffffff98875fe2
 #6 [ffff97c7df29ac78] bad_area at ffffffff98f7ca01
 #7 [ffff97c7df29aca0] __do_page_fault at ffffffff98f908b7
 #8 [ffff97c7df29ad10] trace_do_page_fault at ffffffff98f90a26
 #9 [ffff97c7df29ad50] do_async_page_fault at ffffffff98f8ffa2
#10 [ffff97c7df29ad70] async_page_fault at ffffffff98f8c7a8
#11 [ffff97c7df29ae28] process_bprm_put at ffffffffc04e83d8 [process]
#12 [ffff97c7df29be58] __akfs_trace_security_bprm_check at ffffffffc04e8756 [process]
#13 [ffff97c7df29be78] search_binary_handler at ffffffff98a5587a
#14 [ffff97c7df29bea8] do_execve_common at ffffffff98a56dd6
#15 [ffff97c7df29bf30] compat_sys_execve at ffffffff98a572e9
#16 [ffff97c7df29bf50] stub_execve at ffffffff98f96538
    RIP: 00007f43b2dc6dd7  RSP: 00007ffe56d1d588  RFLAGS: 00000206
    RAX: 000000000000003b  RBX: 0000000001a6f390  RCX: ffffffffffffffff
    RDX: 0000000001a345a0  RSI: 0000000001a5f720  RDI: 0000000001a6f390
    RBP: 0000000001a34520   R8: 0000000000000000   R9: 0000000000000010
    R10: 00007ffe56d1d150  R11: 0000000000000206  R12: 0000000000000001
    R13: 0000000001a5f720  R14: 0000000001a345a0  R15: 0000000000000000
    ORIG_RAX: 000000000000003b  CS: 0033  SS: 002b
```
注:Linux的参数使用情况是:
RDI,RSI,RDX,RCX,R8,R9
超过之后就是开始压栈大概是
```
RBP+24
RBP+16
RBP+8
```
根据参数的类型压入栈内，最下面是返回值。

从Call Trace来看跟云影相关的函数最后是在process_bprm_put at ffffffffc04e83d8
这是云影process.ko模块的一个函数。

- 那我们把process.ko的符号信息加载进来
```
mod -d process
mod -s process process.ko路径
MODULE       NAME                            SIZE  OBJECT FILE
ffffffffc04ea360  process                        13459  /home/bxsec/akps/process.ko 
```

- dis -l ffffffffc04e83d8 反汇编看下该函数干了点哈:

```
crash> dis -l ffffffffc04e83d8
/home/bxsec/akps/./src/capacity.c: 117
0xffffffffc04e83d8 <process_bprm_put+248>:      mov    0x2239(%rip),%rax        # 0xffffffffc04ea618 <__akfs_ops>
```
提示的是__akfs_ops做的操作，那我们反汇编看下__akfs_ops做的了啥:
```
crash> dis -u ffffffffc04ea618
 0xffffffffc04ea618 <__akfs_ops>:     dis: invalid user virtual address: ffffffffc04ea618  type: "gdb_readmem_callback"
Cannot access memory at address 0xffffffffc04ea618
```
这表示__akfs_ops内存不可用了。
打印下 __akfs_ops的地址
```
crash> p __akfs_ops
__akfs_ops = $1 = (akfs_operation_t *) 0xffffffffc0481060
```

查看下__akfs_ops的内容:
```
crash> struct akfs_operation_t ffffffffc0481060
struct akfs_operation_t {
  init = 0xffffffffc047f550, 
  reg = 0xffffffffc047f620, 
  unreg = 0xffffffffc047f740, 
  alloc = 0xffffffffc047f340, 
  free = 0xffffffffc047f360, 
  put = 0xffffffffc047f5b0, 
  get_fpath = 0xffffffffc047e780, 
  get_dpath = 0xffffffffc047e850, 
  get_ipath = 0xffffffffc047e8f0, 
  get_ppath = 0xffffffffc047e6c0, 
  get_tpath = 0xffffffffc047e930, 
  get_args = 0xffffffffc047ebd0, 
  get_cmdline = 0xffffffffc047ea30, 
  get_thash = 0xffffffffc047edb0, 
  get_fhash = 0x0, 
  get_timestamp = 0xffffffffc047ee20, 
  get_unixts = 0xffffffffc047eee0, 
  get_meta = 0xffffffffc047ef20, 
  get_stop_status = 0xffffffffc047f820
}
```
get_fhash值为0

```
crash> dis -l ffffffffc04e83d8
/home/bxsec/akps/./src/capacity.c: 117
```

查看下上面反汇编下来的源码:

```c
int process_bprm_put(struct task_struct *task ,struct linux_binprm *bprm)
{
    unsigned char buffer[PAGE_SIZE] = {0};

    assert_error(__akfs_ops->get_stop_status() ,AKFS_RULE_ALLOW);

    assert_error(!__pad_exec(task ,bprm ,buffer) ,
            AKFS_RULE_ALLOW);

    __akfs_ops->put(&__process.c_args ,buffer ,PAGE_SIZE);

    return AKFS_RULE_ALLOW;
}
static inline int __pad_exec(struct task_struct *task ,
        struct linux_binprm *bprm ,void *buffer)
{

    assert_error(!(__akfs_ops->get_args(bprm , ((akfs_process_t *)buffer)->argv,
                    sizeof(((akfs_process_t *)buffer)->argv))) ,-EACCES);

    __pad_process(task ,PROCESS_TYPE_EXEC ,buffer);

    __akfs_ops->get_fpath(bprm->file ,((akfs_process_t *)buffer)->exec_file ,
            sizeof(((akfs_process_t *)buffer)->exec_file));
    
    __akfs_ops->get_fhash(bprm->file ,((akfs_process_t *)buffer)->exec_hash ,
            sizeof(((akfs_process_t *)buffer)->exec_hash));

    return 0;
}
```
从这可以看出在调用__akfs_ops->get_fhash,所以崩溃了。

> 总结
使用crash调试的时候dis命令反汇编查看对应内存地址的指令，p命令打印变量地址，
struct命令显示对应地址结构体的内容。
这些命令组合起来就是为了定位到那一块代码出了问题，然后审查源码就能很快的发现问题。