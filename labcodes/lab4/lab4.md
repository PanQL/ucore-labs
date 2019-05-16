## Lab4 Report  

### 知识点  
#### 本实验包含的知识点：  

* 内核线程的创建
* 内核线程的切换
* 进程状态模式

#### 还有一些很重要但是本次实验没有包含的知识点：  

* 用户进程

### 练习1  

---

alloc_proc函数的具体实现：

```c
static struct proc_struct *
alloc_proc(void) {
    struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
    if (proc != NULL) {
		proc->state = PROC_UNINIT;
		proc->pid = -1;
		proc->runs = 0;
		proc->kstack = NULL;
		proc->need_resched = 0;
		proc->parent = NULL;
		proc->mm = 0;
		memset(&(proc->context), 0, sizeof(struct context));
		proc->tf = 0;
		proc->cr3 = boot_cr3;
		proc->flags = 0;
		memset(proc->name, 0, sizeof(char) * PROC_NAME_LEN );
    }
    return proc;
}
```

根据实验文档的介绍可知，alloc_proc函数的功能主要是为一个新的process控制块分配内存空间，并进行必要的初始化。

- 根据proc_state定义处的相关注释，可以知道在alloc_proc中初始化的proc_struct的state都设置为uninit。
- 实验文档中指出，pid默认初始化值为-1,通过阅读后续源码发现，进行alloc_proc之后，还会重新设置pid，所以此处设置为-1即可。
- kstack同样会在后续的代码中通过setup_stack()设置，此处给出简单的初始化即可。
- need_resched代表是否需要被调度，同样，在后续代码中会进行设置。同理，mm后续代码中会通过mm_copy()设置，
  - 如果alloc的是内核线程，则使用内核的内存空间，mm不需要设置，此时使用cr3来作为页目录表的基地址，
  - 否则如果alloc的是用户进程，则调用alloc_proc()之后可以通过mm_copy()进行设置。
- tf也会在后续代码中设置，在此处初始化为全0即可。
- 关于cr3,由于是内核线程，初始化为内核的页目录表基地址即可。
- 需要对context和name成员进行清零处理。

**关于struct context context 和 struct trapframe *tf的含义与作用：**

* context代表进程/线程相关的寄存器的值，包括eip,esp等寄存器，代表的是某一个线程在内核管理下的上下文信息。
* trapframe可以认为是当前任务发生中断进入内核时，保存的信息，包括通用寄存器等信息，可以认为是线程在执行过程中的上下文。

### 练习2  

---

最终实现的do_fork函数如下

```c
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    int ret = -E_NO_FREE_PROC;
    struct proc_struct *proc;
    if (nr_process >= MAX_PROCESS) {
        goto fork_out;
    }
    ret = -E_NO_MEM;
	
	proc = alloc_proc();
	proc->parent = current;
	if (setup_kstack(proc) != 0) {
        goto bad_fork_cleanup_proc;
    }
    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }
	
	copy_thread(proc, stack, tf);
	//insert proc_struct into hash_list && proc_list
	bool intr_flag;
    local_intr_save(intr_flag);
    {
		proc->pid = get_pid();
		list_add(&proc_list, &(proc->list_link));
		hash_proc(proc);
		nr_process ++;
	}
	local_intr_restore(intr_flag);
	wakeup_proc(proc);
	ret = proc->pid;
fork_out:
    return ret;

bad_fork_cleanup_kstack:
    put_kstack(proc);
bad_fork_cleanup_proc:
    kfree(proc);
    goto fork_out;
}
```

根据上述代码不难得知，分配一个线程之后，还需要调用get_pid()来获取一个线程号，而get_pid()可以保证返回的线程号是当前没有在使用的。所以能够保证每个进程/线程的id是独一无二的。下面对get_pid（）函数进行简要的分析：

​	get_pid()中主要维护了两个static的全局变量，`last_pid`维护了最新分配出去的`pid`，而`next_safe`维护了`last_pid`所在空闲`pid`区间的上界加一。

​	整个算法可以看作是进行了两重循环，

​	外层循环每次开始时，last_pid都被设置为MAX_PID。每经过一次外层循环，都会实现last_pid的步进值为1的若干次增长，代表对已经分配并且尚未回收的pid的确认。

​	内层循环以遍历线程列表为主体，忽略pid小于当前last_pid的线程，根据大于或等于last_pid的pid，选择步进last_pid或减小next_safe,期望能够压缩得到一个以last_pid为下确界，next_safe为上界的空闲pid区间。

​	循环返回的条件是找到一个last_pid，该pid当前确实没有被分配。

​	这样就实现了get_pid()分配的是全新的pid，并且是当前未被分配的pid中最小的，有效控制了pid的大小范围。

### 练习3  

---

* 当内核初始化线程管理机制并且使能中断之后，每次时钟中断到来时，当前线程的任务信息将会保存在trapframe中。而内核可能会调用schedule函数进行线程调度。线程调度过程中会调用proc_run（）启动一个新的线程。proc_run（）函数中主要完成的事情有：
  * 将当前的进程控制结构修改为目标进程的控制结构。
  * 设置线程运行的堆栈
  * 通过load_cr3函数来设置页表
  * 切换上下文。
* 在本次实验中，创建并执行了两个线程。
* local_intr_save(intr_flag);....local_intr_restore(intr_flag);语句的作用为：
  * 在特定的语句块内禁止中断，保证当前操作为原子操作。
  * 理由：loacal_intr_save(),local_intr_restore()的底层实现是用内联汇编实现的cli、sti语句。是用来屏蔽/使能中断的。