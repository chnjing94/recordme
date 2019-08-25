# 1. 进程

进程是程序的运行实例。当一段程序被同时运行多次时，就会创建多个进程。进程在Linux内核里也叫做task。

## 1.1 进程描述符

进程描述符(process descriptor)，结构类型是task_struct。用来记录进程的状态，存储线程的数据。

![](./pic/1-task_struct.png)

###1.1.1 任务ID

- pid，新建一个进程时，会在前一个进程的ID上加1，赋予新的进程，用来唯一标识这个进程。pid上限是32767，当到达上限时，会重用已经回收的进程的PID。
- tgid，标识该进程所属的线程组，如果某个task的pid等于tgid，则表明该task是线程组的主线程，由这个主线程创建的子线程，其tgid等于主线程的tgid也等于主线程的pid，每个线程都有自己独立的pid，也就是说线程数量不可能超过32767。
- group leader，指向线程组的主线程。

###1.1.2 亲缘关系

- *parent，指向其父进程，父进程是创建它的进程。
- \*real_parent，通常与\*parent一致，在debug的时候，会出现不一致。
- children，子进程链表的头结点，顺着头结点可以访问每一个子进程。
- sibling，该进程的兄弟进程。

###1.1.3 任务状态

- state，表示进程当前状态。
- exit_state，表示当前进程处于哪种退出状态。
- flags，是一个bitmap，表示线程的状态。从下面可以看出如何设置这个bitmap。

```c
/* Used in tsk->state: */
#define TASK_RUNNING                    0
#define TASK_INTERRUPTIBLE              1
#define TASK_UNINTERRUPTIBLE            2
#define __TASK_STOPPED                  4
#define __TASK_TRACED                   8
/* Used in tsk->exit_state: */
#define EXIT_DEAD                       16
#define EXIT_ZOMBIE                     32
#define EXIT_TRACE                      (EXIT_ZOMBIE | EXIT_DEAD)
/* Used in tsk->state again: */
#define TASK_DEAD                       64
#define TASK_WAKEKILL                   128
#define TASK_WAKING                     256
#define TASK_PARKED                     512
#define TASK_NOLOAD                     1024
#define TASK_NEW                        2048
#define TASK_STATE_MAX                  4096
```

![](./pic/1-线程状态.jpeg)

TASK_RUNNING表示进程处于可运行状态，不一定获得了CPU时间片，也就是不一定正在执行。

Linux中，有两种睡眠（挂起）状态，

1. TASK_INTERRUPTIBLE，可中断睡眠，当一个信号到来，进程可以被唤醒，醒来之后处理这个信号。处理完之后，继续睡眠或者退出睡眠。只有当真正等待的条件满足时，才醒来继续往下执行。
2. TASK_UNINTERRUPTIBLE，不可中断的睡眠。不可以被信号唤醒，只能等待满足条件之后才醒来。kill信号都不行，kill本身也是一个信号，也就是说，除了重启电脑，没有别的方法。

TASK_KILLABLE是一种可以终止的新睡眠状态，运行原理和TASK_UNINTERRUPTIBLE类似，只不过可以响应致命信号。

TASK_STOPPED是进程在收到SIGSTOP，SIGTSTP或者SIGTTOU信号之后进入的状态。

一个进程刚结束时，进入EXIT_ZOMBIE状态，此时仅保持进程的描述符，其余的都释放掉。只有等待父进程调用wait()类方法后，才彻底删除这个进程。之后进程的状态为EXIT_DEAD。

还有一些状态，我们称为标志。放在flags字段中，这些字段被定义为宏，以PF开头。例如

```c
#define PF_EXITING			0x00000004
#define PF_VCPU					0x00000010
#define PF_FORKNOEXEC		0x00000040
```

PF_EXITING表示正在退出。PF_VCPU表示运行在虚拟CPU上，PF_FORKNOEXEC表示fork完了，还没有exec。

### 1.1.4 权限

\*real_cred表示谁能操作当前进程，\*cred表示当前进程能操作谁。

cred结构定义如下，cap_xxx表示拥有的能力。xxid都是用户和用户组的信息。

```c
struct cred {
......
        kuid_t          uid;            /* real UID of the task */
        kgid_t          gid;            /* real GID of the task */
        kuid_t          suid;           /* saved UID of the task */
        kgid_t          sgid;           /* saved GID of the task */
        kuid_t          euid;           /* effective UID of the task */
        kgid_t          egid;           /* effective GID of the task */
        kuid_t          fsuid;          /* UID for VFS ops */
        kgid_t          fsgid;          /* GID for VFS ops */
......
        kernel_cap_t    cap_inheritable; /* caps our children can inherit */
        kernel_cap_t    cap_permitted;  /* caps we're permitted */
        kernel_cap_t    cap_effective;  /* caps we can actually use */
        kernel_cap_t    cap_bset;       /* capability bounding set */
        kernel_cap_t    cap_ambient;    /* Ambient capability set */
......
} __randomize_layout;

```

chmod u+s 命令，如果一个文件被设置这样的权限的话，任何进程在执行这个文件时，会获得该文件拥有者的权限，就可以像文件拥有者一样去执行这个文件。

需要了解setuid(uid)函数的原理。

![](./pic/1-setuid.png)

### 1.1.5 运行统计信息

这些字段用来记录该进程运行的时间信息

```c
u64								utime;// 用户态消耗的 CPU 时间
u64								stime;// 内核态消耗的 CPU 时间
unsigned long			nvcsw;// 自愿 (voluntary) 上下文切换计数
unsigned long			nivcsw;// 非自愿 (involuntary) 上下文切换计数
u64								start_time;// 进程启动时间，不包含睡眠时间
u64								real_start_time;// 进程启动时间，包含睡眠时间
```

### 1.1.6 进程调度

详解见进程调度章节

### 1.1.7 信号处理

task_struct里面的信号处理字段如下，

```c
/* Signal handlers: */
struct signal_struct		*signal;
struct sighand_struct		*sighand;
sigset_t								blocked;
sigset_t								real_blocked;
sigset_t								saved_sigmask;
struct sigpending				pending;
unsigned long						sas_ss_sp;
size_t									sas_ss_size;
unsigned int						sas_ss_flags;
```

blocked这个集合表示哪些信号暂不处理，pending表示哪些信号暂待处理，sighand表示正在被处理的信号。详解见信号处理章节。

### 1.1.8 内存管理

每个进程都有自己独立的虚拟内存空间，用mm_struct来表示。详解见内存管理章节。

```c
struct mm_struct                *mm;
struct mm_struct                *active_mm;
```

详见内存管理章节。

### 1.1.9 文件与文件系统

每个进程都有一个文件系统的数据结构，还有一个数据结构用来记录已打开文件。详解见文件系统章节。

```c
/* Filesystem information: */
struct fs_struct                *fs;
/* Open file information: */
struct files_struct             *files;
```

### 1.1.10 内核栈

进程一旦涉及到用户态和内核态切换，就会用到以下两个变量

```c
struct thread_info		thread_info;
void  *stack;
```

stack表示内核栈，进程在内核运行时，需要切换到内核栈。thread_info用来存储CPU上下文切换的信息。详见进程切换章节。

## 1.2 创建进程

###1.2.1 fork

在linux中，怎么创建进程呢？是使用fork()系统调用。内部的调用过程为fork -> sys_fork -> _do_fork，sys_fork的定义如下：

```c
SYSCALL_DEFINE0(fork)
{
......
	return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
}
```

sys_fork又会去调用_do_fork，

```c
long _do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr,
	      unsigned long tls)
{
	struct task_struct *p;
	int trace = 0;
	long nr;


......
	p = copy_process(clone_flags, stack_start, stack_size,
			 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
......
	if (!IS_ERR(p)) {
		struct pid *pid;
		pid = get_task_pid(p, PIDTYPE_PID);
		nr = pid_vnr(pid);


		if (clone_flags & CLONE_PARENT_SETTID)
			put_user(nr, parent_tidptr);


......
		wake_up_new_task(p);
......
		put_pid(pid);
	} 
......
```

这里主要做了两件事，`copy_process`复制结构和`wake_up_new_task`唤醒线程。

### 1.2.2 copy_process

复制父进程就是复制它的进程描述符task_struct，按照下图逐一解释复制过程。

![](./pic/1-task_struct.png)

- 整体

  - 先调用alloc_task_struct_node，分配一个task_struct结构。
  - 调用memcpy将父进程的task_struct进程复制。
  - 此时所有的成员变量都指向父进程对应的值，也就是共用父进程的数据，之后一步步地创建自己的数据，与父进程彻底隔绝。

- 内核栈

  - 调用alloc_thread_stack_node，分配内存空间并创建一个栈赋值给task_struct的 void *stack成员变量。
  - 调用setup_thread_stack设置thread_info。

- 权限

  - 调用prepare_creds，在内存中分配一个新的struct cred结构，然后调用memcpy复制一份父进程的cred。
  - 将新进程的read_cred和cred都指向这个新创建的cred。

- 运行统计

  初始化变量值。

  ```c
  p->utime = p->stime = p->gtime = 0;
  p->start_time = ktime_get_ns();
  p->real_start_time = ktime_get_boot_ns();
  ```

- 调度相关

  - 调用__sched_fork，将on_rq设为0，初始化shed_entity，shed_entity的定义如下

    ```c
    struct sched_entity {
    	struct load_weight		load;
    	struct rb_node			run_node;
    	struct list_head		group_node;
    	unsigned int			on_rq;
    	u64				exec_start;
    	u64				sum_exec_runtime;
    	u64				vruntime;
    	u64				prev_sum_exec_runtime;
    	u64				nr_migrations;
    	struct sched_statistics		statistics;
    ......
    };
    ```

    初始化shed_entity就是把exec_start，sum_exec_runtime等涉及进程运行时间置零。

  - 设置进程的状态p->state设置为TASK_NEW。

  - 初始化优先级

  - 设置调度类，如果是普通进程，则设置sched_class = &fair_sched_class，公平调度。

  - 调用调度类的task_fork函数，将子进程和父进程的vruntime设为一样，这样就能防止父进程通过不断新建子进程来占用cpu。如果设置了让子进程先行，调用resched_curr，标记当前运行的进程TIF_NEED_RESCHED，这样下次调度的时候，父进程会被子进程抢占。

- 文件与文件系统

  ```c
  retval = copy_files(clone_flags, p);
  retval = copy_fs(clone_flags, p);
  ```

  copy_files是复制父进程打开的文件信息，这些信息存在一个结构files_struct中，每个打开的文件都有一个文件描述符。这里创建一个新的files_struct，然后将所有的文件描述符数组fdtable拷贝一份。

  copy_fs用于复制一个进程的目录信息，这些目录信息包括进程自己的根目录，根文件系统root，也有当前目录pwd，和当前目录的文件系统。

- 信号相关

  ```c
  init_sigpending(&p->pending);
  retval = copy_sighand(clone_flags, p);
  retval = copy_signal(clone_flags, p);
  ```

  copy_sighand会分配一个新的sighand_struct，主要是维护信号处理函数。然后调用memcpy将信号处理函数从父进程复制到子进程。

  copy_singal用于复制和维护发给这个进程的信号的数据结构。

- 内存空间

  ```c
  retval = copy_mm(clone_flags, p);
  ```

  进程都有自己的内存空间，用mm_struct结构来表示。copy_mm函数中调用dup_mm分配一个新的mm_struct结构。

- 任务ID

  - 分配pid，设置tid，group_leader。
  - 建立进程直接的亲缘关系。

至此，copy_process就结束了。

### 1.2.3 wake_up_new_task

新创建的进程，需要做些工作，才可能去抢占CPU。wake_up_new_task定义如下，

```c
void wake_up_new_task(struct task_struct *p)
{
	struct rq_flags rf;
	struct rq *rq;
......
	p->state = TASK_RUNNING;
......
	activate_task(rq, p, ENQUEUE_NOCLOCK);
	p->on_rq = TASK_ON_RQ_QUEUED;
	trace_sched_wakeup_new(p);
	check_preempt_curr(rq, p, WF_FORK);
......
}
```

1. 将state设置为TASK_RUNNING。
2. activate_task函数会调用enqueue_task，将进程放到运行队列里面，每个CPU有自己的多个运行队列，一个进程只能同时在一个队列里排队。如果是CFS的调度类，也就是公平调度，就会将进程放到cfs_rq，然后调用date_curr更新运行的统计量，将进程的sched_entity加入到红黑树（cfs_rq的实现是红黑树）里，设置se->on_rq = 1。
3. 调用check_preempt_curr，看能否抢占当前进程。如果前面设置了让子进程先运行，则直接返回。如果没有设置，就让父子进程PK一下，看是否要抢占，如果是，则标记父进程为TIF_NEED_RESCHED。抢占的时机为，fork系统调用返回的时候，因为进程抢占可以发生在系统调用返回的时候。

##1.3 创建线程

线程和进程的创建过程有很多相似的地方，因此接着讲线程的创建。

![](./pic/1-创建线程.jpeg)

如上图所示，左边是创建进程，右边是创建线程，它们最大的区别有2点，首先是线程的创建有一部分工作是在用户态完成的，也就是`pthread_create`函数，这个函数来自Glibc，不是系统调用。第二点不同是在下面复制数据的时候，子线程是直接引用了parent的数据，也就是说，这部分数据被所有的子线程共享。

### 1.3.1 用户态工作pthread_create

首先说说pthread是什么，我们知道在无论是线程还是进程内核里都用task_struct来表示，而线程在用户态是用pthread这个结构来维护。因此首先就是创建这么一个pthread结构，怎么创建呢？

```c
struct pthread *pd = NULL;
int err = ALLOCATE_STACK (iattr, &pd);
```

看起来有点奇怪，创建一个pthread是调用ALLOCATE_STACK去创建一个stack，但是经过这么一个调用后确实创建了一个新的pthread，接下来来看具体里面做了什么。ALLOCATE_STACK的定义如下

```c
1   # define ALLOCATE_STACK(attr, pd) allocate_stack (attr, pd, &stackaddr)
2   
3   static int
4   allocate_stack (const struct pthread_attr *attr, struct pthread **pdp,
5                   ALLOCATE_STACK_PARMS)
6   {
7     struct pthread *pd;
8     size_t size;
9     size_t pagesize_m1 = __getpagesize () - 1;
10  ......
11    size = attr->stacksize;
12  ......
13    /* Allocate some anonymous memory.  If possible use the cache.  */
14    size_t guardsize;
15    void *mem;
16    const int prot = (PROT_READ | PROT_WRITE
17                     | ((GL(dl_stack_flags) & PF_X) ? PROT_EXEC : 0));
18    /* Adjust the stack size for alignment.  */
19    size &= ~__static_tls_align_m1;
20    /* Make sure the size of the stack is enough for the guard and
21    eventually the thread descriptor.  */
22    guardsize = (attr->guardsize + pagesize_m1) & ~pagesize_m1;
23    size += guardsize;
24    pd = get_cached_stack (&size, &mem);
25    if (pd == NULL)
26    {
27      /* If a guard page is required, avoid committing memory by first
28      allocate with PROT_NONE and then reserve with required permission
29      excluding the guard page.  */
30  	mem = __mmap (NULL, size, (guardsize == 0) ? prot : PROT_NONE,
31  			MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);
32      /* Place the thread descriptor at the end of the stack.  */
33  #if TLS_TCB_AT_TP
34      pd = (struct pthread *) ((char *) mem + size) - 1;
35  #elif TLS_DTV_AT_TP
36      pd = (struct pthread *) ((((uintptr_t) mem + size - __static_tls_size) & ~__static_tls_align_m1) - TLS_PRE_TCB_SIZE);
37  #endif
38      /* Now mprotect the required region excluding the guard area. */
39      char *guard = guard_position (mem, size, guardsize, pd, pagesize_m1);
40      setup_stack_prot (mem, size, guard, guardsize, prot);
41      pd->stackblock = mem;
42      pd->stackblock_size = size;
43      pd->guardsize = guardsize;
44      pd->specific[0] = pd->specific_1stblock;
45      /* And add to the list of stacks in use.  */
46      stack_list_add (&pd->list, &stack_used);
47    }
48    
49    *pdp = pd;
50    void *stacktop;
51  # if TLS_TCB_AT_TP
52    /* The stack begins before the TCB and the static TLS block.  */
53    stacktop = ((char *) (pd + 1) - __static_tls_size);
54  # elif TLS_DTV_AT_TP
55    stacktop = (char *) (pd - 1);
56  # endif
57    *stack = stacktop;
58  ...... 
59  }
```

`allocate_stack`具体做了一下工作：

1. 11行，如果你设置了stack的size，就把这个值取出来，之后要用来设置stack的大小。
2. 22行，为了防止栈的越界访问，在栈的尾部会有一块儿空间guardsize，一旦访问这里就错误了。
3. 24行，接下来就是真正创建pthread的地方了，会调用`get_cached_stack`去缓存中获取一个栈，因为所有线程的栈都是在进程的堆里面创建的，创建删除十分频繁，所以需要把这些分配的内存缓存起来。
4. 如果从缓存中获取栈失败，则进入26-47行
   1. 30行，调用`__mmap`分配一块新的栈内存。
   2. 接下来将栈底这块空间分配给pthread，也就是说pthread是放在栈里面的。至此，我们得到了pthread。
   3. 39行，计算出guard内存的位置，40行调用`setup_stack_prot`来设置这一块内存是受保护的。
   4. 41-44行，填充pthread这个结构里的stackblock（刚才分配的栈内存块），stackblock_size（栈内存大小），guardsize（内存保护区），specific（存放的是线程的全局变量）。
   5. 将新建的这个线程栈放入到stack_used链表中，之后用完了会缓存到stack_cache中。
5. 49行，将新建的pthread赋值给目标。

到这里，pthread就创建好了，现在我们就能理解了为什么用allocate_stack来创建pthread，因为要先创建线程栈，再指定一块栈内存用来创建pthread。现在我们知道了，在用户态中，创建线程需要做的工作就是创建了新线程所需的栈。接下来要把这个栈交给内核态去完成后续的创建工作。

在交给create_thread去进行内核态工作之前，我们还需要为新建的pthread设置start_routine，也就是线程需要执行的函数，以及arg，传给这个函数的参数，以及调度策略。

```c
pd->start_routine = start_routine;
pd->arg = arg;
pd->schedpolicy = self->schedpolicy;
pd->schedparam = self->schedparam;
/* Pass the descriptor to the caller.  */
*newthread = (pthread_t) pd;
atomic_increment (&__nptl_nthreads);
retval = create_thread (pd, iattr, &stopped_start, STACK_VARIABLES_ARGS, &thread_ran);
```

###1.3.2 内核态工作create_thread

create_thread是内核态工作的入口，还没有正式进入内核态，因为我们知道要进行系统调用才算进入内核态。其定义如下：

```c
static int
create_thread (struct pthread *pd, const struct pthread_attr *attr,
bool *stopped_start, STACK_VARIABLES_PARMS, bool *thread_ran)
{
  const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM | CLONE_SIGHAND | CLONE_THREAD | CLONE_SETTLS | CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID | 0);
  ARCH_CLONE (&start_thread, STACK_VARIABLES_ARGS, clone_flags, pd, &pd->tid, tp, &pd->tid)；
  /* It's started now, so if we fail below, we'll have to cancel it
and let it clean itself up.  */
  *thread_ran = true;
}
```

clone_flags定义了一些标志，指明了要复制的数据是哪些。需要特别关注。

ARCH_CLONE宏，其实就是调用__clone，定义如下，这里正式进入系统调用了。

```c
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
{
	return _do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}
```

看到了熟悉的_do_fork，也就是和进程的复制步骤一样了。

_do_fork前面讲过了，现在重点关心一下创建进程和创建线程在这个地方的区别。

1. 之前我们设置了clone_flags，其影响了以下几方面。

   首先是copy_files，创建进程时，是复制一个files_struct，而这里因为有CLONE_FILES标志，只是将files_struct引用计数加一。

   ```c
   static int copy_files(unsigned long clone_flags, struct task_struct *tsk)
   {
   	struct files_struct *oldf, *newf;
   	oldf = current->files;
   	if (clone_flags & CLONE_FILES) {
   		atomic_inc(&oldf->count);
   		goto out;
   	}
   	newf = dup_fd(oldf, &error);
   	tsk->files = newf;
   out:
   	return error;
   }
   ```

   对于copy_fs，原来是复制一个fs_struct，因为CLONE_FS标志的存在，将原来的fs_struct用户数加一。

   ```c
   static int copy_fs(unsigned long clone_flags, struct task_struct *tsk)
   {
   	struct fs_struct *fs = current->fs;
   	if (clone_flags & CLONE_FS) {
   		fs->users++;
   		return 0;
   	}
   	tsk->fs = copy_fs_struct(fs);
   	return 0;
   }
   ```

   同理，对于copy_sighand，copy_signal，copy_mm都从以前的复制一个新的，到现在的共享一个老的。

2. 对于亲缘关系的影响，因为在clone_flags里设置了CLONE_THREAD，就表明现在是在创建一个线程。

   - 如果是创建新进程，设置新进程的group_leader是他自己，tgid是它自己的pid。如果是创建新线程，group_leader是当前进程的group_leader，tgid是当前进程的tgid，也就是说，这个新线程是当前进程的线程组的一员。
   - 如果是创建新进程，那么新进程的real_parent就是当前进程，进程树上又会多一层。如果是创建新线程，线程的real_parent是当前进程的real_parent，就是一辈的。

3. 对信号处理的不同在于，对于新建进程，会在copy_signal的时候，初始化signal_struct里面的shard_pending，也就是说不同的进程有自己的信号处理。而新建线程的时候，共享了父进程的signal_struct，也就是线程组中的所有线程共享一个share_pending，这是一个信号列表。因此发给父进程的信号所有的子线程都能收到。kill掉一个进程，也就会Kill掉所有子线程。

至此，clone在内核的调用完毕，要退出系统调用，返回用户态。

### 1.3.3 用户态执行线程

回到用户态后，根据__clone的第一个参数，不是马上执行我们指定的那个函数，而是执行一个通用的start\_thread，这是所有线程再用户态的统一入口。

```c
1   #define START_THREAD_DEFN \
2     static int __attribute__ ((noreturn)) start_thread (void *arg)
3   
4   
5   START_THREAD_DEFN
6   {
7       struct pthread *pd = START_THREAD_SELF;
8       /* Run the code the user provided.  */
9       THREAD_SETMEM (pd, result, pd->start_routine (pd->arg));
10      /* Call destructors for the thread_local TLS variables.  */
11      /* Run the destructor for the thread-local data.  */
12      __nptl_deallocate_tsd ();
13      if (__glibc_unlikely (atomic_decrement_and_test (&__nptl_nthreads)))
14          /* This was the last thread.  */
15          exit (0);
16      __free_tcb (pd);
17      __exit_thread ();
18  }
```

- 在第9行，真正执行调用用户提供的函数
- 12-17行是在用户的函数执行完毕之后，释放这个线程相关的数据。
  - 13行，线程数目减一，如果是进程中最后一个线程了，就退出进程。
  - 16行，__free\_tcb用于释放pthread。\_\_free\_tcb会调用\_\_deallocate_stack来释放整个栈，把这个栈从stack\_used中拿下来，放到stack\_cache中。

总结一下，创建进程的系统调用是fork，在copy_process里面讲五大结构都复制一遍，从此父子进程各用个的。而创建线程的话，先创建线程的栈，然后调用系统调用clone，在copy_process的时候，仅仅将五大结构的引用计数加一，也就是共享进程的数据结构。

##1.4 退出进程

进程结束了自己的任务之后，就应该做一些善后的工作。这些工作包括，通知父进程自己已经结束，以及释放占用的资源。

进程的退出分为正常退出和非正常退出：

- 正常退出：在main中return，调用exit()，调用_exit()。
- 异常退出：调用abort函数，收到终止信号，或者产生了一个不可恢复的CPU异常。

Linux中有两个终止用户态应用的系统调用，exit_group()和exit()。

###1.4.1 exit_group()和exit()

exit_group()用来终止整个线程组，即整个基于多线程的应用。具体调用的内核实现函数式do_group_exit()。do_exit_group()接受进程终止代号作为参数，这个代号可以是正常结束，也可以是内核提供的一个错误代号。该函数做以下操作：

1. 检查已退出线程的SIGNAL_GROUP_EXIT标志是否为0，如果不为0，说明内核已经开始执行退出线程组。则调到第4步。
2. 否则。设置进程的SIGNAL_GROUP_EXIT标志，并把终止代号存放到current->signal_group_exit_code字段。
3. 调用zap_other_threads()杀死当前线程组里的其他线程（除了自己），也就是找到tgid对应的线程组，对线程组的每个不同于当前进程的线程发送SIGKILL信号。结果就是每个线程都会执行do_exit()，从而被杀死。
4. 调用do_exit()，把终止代号传给它，结束自己。



exit()系统调用用来终止某一个进程（或线程）。do_exit()是实现这个系统调用的主要内核函数。所有进程的终止都是用do_exit()来处理，这个函数从终止进程的内核数据结构(task_struct)中删除大部分引用。它接受一个终止代码作为参数并执行以下操作：

1. 把进程描述符的flag字段设置为PF_EXITING标志，以表示进程正在被删除。
2. 分别调用exit_mm()，exit_sem()，\_\_exit_files()，\_\_exit\_fs()，exit\_namespace()和exit\_thread()函数从进程描述符中分离出与分页，信号量，文件系统，打开文件描述符，命名空间以及I/O权限位图相关的数据结构。如果没有其他进程共享这些数据结构，就会被删除。
3. 将进程描述符的exit_code字段设置成进程的终止代号。
4. 调用exit_notify()函数执行下面的操作：
   1. 更新父进程与子进程的亲属关系，如果被终止的进程所在的线程组中，还有别的线程，则把终止进程创建的所有子进程都过继给自己的兄弟，也就是线程组中的某一个线程，因为线程组中的线程都是终止进程的兄弟。否则，也就是当线程组里没有别的线程了，就把终止进程创建的子进程过继给init进程。
   2. 如果终止进程是线程组里最后一个成员，则向父进程发送SIGCHLD信号，通知父进程子进程死亡。如果线程组还有其他进程，且终止进程正在被追踪，就向调试进程发送一个SIGCHLD信号，通知自己已经死亡。
   3. 如果进程没有被跟踪，就把进程描述符的exit_state字段设置为EXIT_DEAD，然后调用release_task()回收进程的其他数据结构占用的内存，但不会释放进程描述符本身。如果进程被追踪，就把exit_state字段设置为EXIT_ZOMBIE。
   4. 把进程描述符的flag字段设置为PF_DEAD。
5. 调用schedule()选择一个新进程运行，选择的时候会忽略那些EXIT_ZOMBIE状态的进程。

### 1.4.2 进程删除

想象一下这个场景，父进程在fork出子进程之后就接着往下执行了，如果父进程想知道子进程的执行结果，怎么办呢？当然是在某个地方等着了。wait()类系统调用就是提供这样的功能，wait()可以设置无限等待子线程，直到返回一个终止代号，告诉父进程是执行成功还是失败。也可以设置timeout，等多久，要是等不到就不等了，父进程接着往下执行。

对于子进程来说，事情做完了也要保留进程描述符，因为父进程可能会用到。但是如果父进程根本没有等子进程，也就是没有调用wait()，或者父进程比子进程先结束，子进程的进程描述符就一直得不到释放，而子进程也就变成了僵尸进程。如果这些僵尸进程得不到处理，就会白白浪费资源，因此僵尸进程都会过继给init进程，init进程会调用wait()来处理这些进程。

release_task()用来回收进程描述符所占用的内存空间。执行步骤如下：

1. 递减终止进程拥有者的进程个数。
2. 如果进程正在被跟踪，把它从调试程序的ptrace_children链表中删除，并让该进程重新属于初始的父进程。
3. 删除所有挂起的信号并释放singal_struct描述符。
4. 删除信号处理函数。
5. 从进程描述符链表中删除该进程描述符。将nr_threads减一。
6. 递减进程描述符的使用计数器，如果为0，就释放进程描述符以及thread_info描述符和内核栈所占用的空间。

## 1.5 进程调度

一颗4核CPU，加上超线程技术也就8线程，也就是说，任意一时刻可以执行8个进程，而操作系统里同时运行着成千上万的进程，因此需要一个进程调度机制，让进程们有条不紊地使用CPU，这是一个非常复杂需要平衡的事情。先来看看跟调度有关的数据结构。

### 1.5.1 数据结构

#### 1.5.1.1 调度策略与调度类

在Linux中，进程可以被分为两类

1. 实时进程，它们有很强的调度需求，需要很短的响应时间，不能被低优先级进程阻塞，例如音视频程序。
2. 普通进程，大部分进程都是普通进程。

不同的进程，我们需要不同的调度策略，这个策略设置在task_struct中，这个成员变量叫调度策略

```c
unsigned int policy;
```

它有以下几种定义，

```c
#define SCHED_NORMAL		0
#define SCHED_FIFO		1
#define SCHED_RR		2
#define SCHED_BATCH		3
#define SCHED_IDLE		5
#define SCHED_DEADLINE		6
```

- 实时调度策略：针对实时进程的调度策略，有以下三种
  - SCHED_FIFO：对于相同的优先级，先来先服务，高优先级可以抢占低优先级。
  - SCHED_RR：对于相同优先级，采用时间片轮流调度法，用完时间片的进程就放到队列后面。高优先级可以抢占低优先级。
  - SCHED_DEADLINE：按照任务的deadline进行调度，总是先选择离deadline最近的那个任务执行。

配合调度策略的，还有优先级，也在task_struct中

```c
int prio, static_prio, normal_prio;
unsigned int rt_priority;
```

优先级是一个数值，对于实时进程，优先级范围是0~99；对于普通进程，优先级范围是100~139。数值越小优先级越高。