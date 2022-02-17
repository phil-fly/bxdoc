这次调式的异常不算多难，但是里面还是包含了几个小知识点。
话不多说，边调式边了解下涉及到的知识点。

> 加载vmcore

```
crash /usr/lib/debug/lib/modules/3.10.0-957.el7.x86_64/vmlinux vmcore
```
```
KERNEL: /usr/lib/debug/lib/modules/3.10.0-957.el7.x86_64/vmlinux 
    DUMPFILE: vmcore  [PARTIAL DUMP]
        CPUS: 4
        DATE: Thu Jan 13 11:54:08 2022
      UPTIME: 23:13:09
LOAD AVERAGE: 6.35, 6.04, 4.32
       TASKS: 1678
    NODENAME: localhost.localdomain
     RELEASE: 3.10.0-957.el7.x86_64
     VERSION: #1 SMP Thu Nov 8 23:39:32 UTC 2018
     MACHINE: x86_64  (1699 Mhz)
      MEMORY: 8 GB
       PANIC: "BUG: unable to handle kernel NULL pointer dereference at 0000000000000018"
         PID: 16985
     COMMAND: "redis-server"
        TASK: ffff8ff3ea9230c0  [THREAD_INFO: ffff8ff468ffc000]
         CPU: 1
       STATE: TASK_RUNNING (PANIC)
```
从PANIC来看是个空指针错误，有意思的是空指针的地址是18.

> 查看错误堆栈

```
crash> bt
PID: 16985  TASK: ffff8ff3ea9230c0  CPU: 1   COMMAND: "redis-server"
 #0 [ffff8ff468fffb70] machine_kexec at ffffffff88263674
 #1 [ffff8ff468fffbd0] __crash_kexec at ffffffff8831ce12
 #2 [ffff8ff468fffca0] crash_kexec at ffffffff8831cf00
 #3 [ffff8ff468fffcb8] oops_end at ffffffff8896c758
 #4 [ffff8ff468fffce0] no_context at ffffffff8895aa7e
 #5 [ffff8ff468fffd30] __bad_area_nosemaphore at ffffffff8895ab15
 #6 [ffff8ff468fffd80] bad_area_nosemaphore at ffffffff8895ac86
 #7 [ffff8ff468fffd90] __do_page_fault at ffffffff8896f6b0
 #8 [ffff8ff468fffe00] do_page_fault at ffffffff8896f915
 #9 [ffff8ff468fffe30] page_fault at ffffffff8896b758
    [exception RIP: do_sys_open+326]
    RIP: ffffffff8843ff36  RSP: ffff8ff468fffee8  RFLAGS: 00010207
    RAX: 0000000000000000  RBX: 0000000000000008  RCX: 0000000000000038
    RDX: ffff8ff468fffef4  RSI: ffff8ff3ee2ae000  RDI: ffffffffc06534a0
    RBP: ffff8ff468ffff38   R8: ffff8ff467b507d0   R9: 0000000000000040
    R10: 8080808080808080  R11: 0000000000000000  R12: 0000000000000000
    R13: ffff8ff3ee2ae000  R14: 0000000000000000  R15: 0000000000000000
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#10 [ffff8ff468ffff40] sys_open at ffffffff8844004e
#11 [ffff8ff468ffff50] tracesys at ffffffff8897505b (via system_call)
    RIP: 00007f54214d7352  RSP: 00007ffee69a2dd8  RFLAGS: 00000246
    RAX: ffffffffffffffda  RBX: 00007f542117f890  RCX: ffffffffffffffff
    RDX: 0000000000000000  RSI: 0000000000008000  RDI: 00007ffee69a2e90
    RBP: 00007ffee69a2e90   R8: 0000000000000000   R9: 0000000000000000
    R10: 0000000000000000  R11: 0000000000000246  R12: 0000000000000002
    R13: 0000000000000001  R14: 0000000000000000  R15: 000000137653a88d
    ORIG_RAX: 0000000000000002  CS: 0033  SS: 002b
```

 `[exception RIP: do_sys_open+326]` 这一行意思是异常指令在do_sys_open+326触发的，
而云影的file模块在do_filp_open处做了hook.

> 导致NULL的崩溃点分析

```
crash> dis -l do_sys_open+326
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/include/linux/fsnotify.h: 216
0xffffffff8843ff36 <do_sys_open+326>:   mov    0x18(%rax),%rsi
```

根据给的提示是(%rax)的内存偏移0x18导致了NULL的dump

```
PANIC: "BUG: unable to handle kernel NULL pointer dereference at 0000000000000018"
```
减去18则表示(%rax)的内存地址是0，也就是NULL空指针了


看下do_sys_open+326上面的指令内容,以下内容只截取了326附近的指令
```
crash> dis -l do_sys_open
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/fs/open.c: 1050
0xffffffff8843ff18 <do_sys_open+296>:   lea    -0x44(%rbp),%rdx
0xffffffff8843ff1c <do_sys_open+300>:   mov    %r12d,%edi
0xffffffff8843ff1f <do_sys_open+303>:   mov    %r13,%rsi
0xffffffff8843ff22 <do_sys_open+306>:   callq  0xffffffff88453db0 <do_filp_open>
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/fs/open.c: 1051
0xffffffff8843ff27 <do_sys_open+311>:   cmp    $0xfffffffffffff000,%rax
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/fs/open.c: 1050
0xffffffff8843ff2d <do_sys_open+317>:   mov    %rax,%r12
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/fs/open.c: 1051
0xffffffff8843ff30 <do_sys_open+320>:   ja     0xffffffff88440020 <do_sys_open+560>
/usr/src/debug/kernel-3.10.0-957.el7/linux-3.10.0-957.el7.x86_64/include/linux/fsnotify.h: 216
0xffffffff8843ff36 <do_sys_open+326>:   mov    0x18(%rax),%rsi
```

这段指令就是调用do_filp_open，然后根据返回值来确定是调转到560还是执行326.
而根据崩溃的信息是cmp    $0xfffffffffffff000,%rax不相等走到了326，引发了NULL

根据提示看下内核代码

```c
struct file *f = do_filp_open(dfd, tmp, &op);
        if (IS_ERR(f)) {
            put_unused_fd(fd);
            fd = PTR_ERR(f);
        } else {
            fsnotify_open(f);
            fd_install(fd, f);
        }
```

云影的file模块相关代码如下：
```c
akfs_trace_declare(do_filp_open);
static struct file *akfs_trace_function(do_filp_open)(int dfd, struct filename *pathname,const struct open_flags *op)
{
    struct file *(*run)(int ,struct filename * ,const struct open_flags *) = (void *)akfs_trace_get_original(do_filp_open);
    struct file *file = NULL;

    assert_error(try_module_get(THIS_MODULE) ,NULL);

    file = run(dfd ,pathname ,op);
    assert_goto(!IS_ERR_OR_NULL(file) ,out ,);

    file_file_put(current ,file ,FILE_TYPE_CREATE);

out:
    module_put(THIS_MODULE);

    return file;
}
```

崩溃的代码如下:
```c
static inline void fsnotify_open(struct file *file)
{
    const struct path *path = &file->f_path;
    struct inode *inode = path->dentry->d_inode;  //崩溃点
    ......
}
```

这里面涉及到三个结构体:
```c
struct file {
    /*
     * fu_list becomes invalid after file_free is called and queued via
     * fu_rcuhead for RCU freeing
     */
    union {
        struct list_head    fu_list;
        struct rcu_head     fu_rcuhead;
    } f_u;
    struct path     f_path;
    ........
}

struct path {
    struct vfsmount *mnt;
    struct dentry *dentry;
};

struct dentry {
    /* RCU lookup touched fields */
    unsigned int d_flags;       /* protected by d_lock */
    seqcount_t d_seq;       /* per dentry seqlock */
    struct hlist_bl_node d_hash;    /* lookup hash list */
    struct dentry *d_parent;    /* parent directory */
    struct qstr d_name;
    struct inode *d_inode;      /* Where the name belongs to - NULL is
                     * negative */
	.......
}
```
通过三个结构体和崩溃点可以确定fsnotify_open的file参数为NULL.
而云影在`assert_error(try_module_get(THIS_MODULE) ,NULL);`返回了NULL.
内核使用IS_ERR(NULL)来判断返回之struct file *是不是正常的。
根据崩溃来看IS_ERR(NULL)判断为false，也就是说不是个错误，所以接下来的才会对NULL进行了指针偏移赋值操作。

此处就涉及到另一个知识点:内核的指针类型
- 正常指针
- 错误指针
- NULL


```c
#define MAX_ERRNO   4095
        
#ifndef __ASSEMBLY__
            
#define IS_ERR_VALUE(x) unlikely((x) >= (unsigned long)-MAX_ERRNO)
            
static inline void * __must_check ERR_PTR(long error)
{       
    return (void *) error;
}   
    
static inline long __must_check PTR_ERR(const void *ptr)
{   
    return (long) ptr;
}   
    
static inline long __must_check IS_ERR(const void *ptr)
{
    return IS_ERR_VALUE((unsigned long)ptr);
}
```

简单来说就是判断ptr的值是不是大于等于(unsigned long)-4095.

```
0xffffffff8843ff27 <do_sys_open+311>:   cmp    $0xfffffffffffff000,%rax
```

这个地址可看出 `0xfffffffffffff000-0xfffffffffffffffff`是寻指范围的最后一个PAGE.
而之所以在把最后一个PAGE作为错误码是因为当指针位于最后一个PAGE的时候，不足一个PAGE无法对齐了，所以保留最后一个PAGE.而这么做的好处还在于当错误返回的时候，不仅仅代表了错误，还能获取到错误码，一个返回值代表两层含义.
而云影的file模块返回NULL(0)也就自然不再这个范围了，所以没有当作错误来处理，造成了对NULL的偏移赋值.
找到问题之后修复就简单了，不返回NULL，使用ERR_PTR(错误码)即可.