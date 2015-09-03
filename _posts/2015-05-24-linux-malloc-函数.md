title: linux malloc 函数
date: 2015-05-24 13:58:53
categories:
- linux
tags: 
- memory
---

## 进程堆

想找个时间，看下malloc是如何实现的。有一种做法是，把进程的内存管理交给操作系统内核去做，既然内核管理者进程的地址空间，那么如何它提供一个系统调用，可以让程序使用这个系统调用申请内存，不久可以么？但是实际上这样做的性能比较差，因为每次程序申请或者释放内存都要进行系统调用。我们知道系统调用的性能开销是很大的，当程序中对堆的操作比较繁琐时，这样做的结果是会严重影响程序的性能。比较好的做法是程序向操作系统申请一块适当大小的堆空间，然后由程序自己管理这块空间，而具体来讲，管理者堆空间分配的往往是程序的运行库。

<!--more-->

运行库相当于是向操作系统“批发”了一块较大的堆空间，然后“零售”给程序员用。当全部“售完”或者程序有大量的内存需求时，再根据实际需求想操作系统“进货”。当然运行库在程序零售堆空间时，必须管理它批发来的堆空间，不能把同一块地址出售两次，导致地址冲突。于是运行库需要一个算法来管理堆空间，这个算法就堆的分配算法。


## Linux 进程堆空间

Linux 下的进程堆管理提供了两种堆空间的分配方式，即两个系统调用：

- 一个是brk()系统调用
- 一个是mmap()系统调用。

brk的声明如下

``` c
  int brk(void *addr);
```
``` c
SYSCALL_DEFINE1(brk, unsigned long, brk)
{
	unsigned long rlim, retval;
	unsigned long newbrk, oldbrk;
	struct mm_struct *mm = current->mm;
	unsigned long min_brk;

	down_write(&mm->mmap_sem);

#ifdef CONFIG_COMPAT_BRK
	min_brk = mm->end_code;
#else
	min_brk = mm->start_brk;
#endif
	if (brk < min_brk)
		goto out;

	/*
	 * Check against rlimit here. If this check is done later after the test
	 * of oldbrk with newbrk then it can escape the test and let the data
	 * segment grow beyond its set limit the in case where the limit is
	 * not page aligned -Ram Gupta
	 */
	rlim = current->signal->rlim[RLIMIT_DATA].rlim_cur;
	if (rlim < RLIM_INFINITY && (brk - mm->start_brk) +
			(mm->end_data - mm->start_data) > rlim)
		goto out;

	newbrk = PAGE_ALIGN(brk);
	oldbrk = PAGE_ALIGN(mm->brk);
	if (oldbrk == newbrk)
		goto set_brk;

	/* Always allow shrinking brk. */
	if (brk <= mm->brk) {
		if (!do_munmap(mm, newbrk, oldbrk-newbrk))
			goto set_brk;
		goto out;
	}

	/* Check against existing mmap mappings. */
	if (find_vma_intersection(mm, oldbrk, newbrk+PAGE_SIZE))
		goto out;

	/* Ok, looks good - let it rip. */
	if (do_brk(oldbrk, newbrk-oldbrk) != oldbrk)
		goto out;
set_brk:
	mm->brk = brk;
out:
	retval = mm->brk;
	up_write(&mm->mmap_sem);
	return retval;
}
```
brk 的实际作用就是设置进程数据段的结束地址，即它可以扩大或者缩小数据段。如果我们可将数据段的结束地址向高地址移动，那么扩大的那部分空间就可以被我们使用，把这块空间拿来作为堆空间是最常见的做法之一。

brk()在内核中对应的系统调用服务例程为SYSCALL_DEFINE1(brk, unsigned long, brk)，参数brk用来指定heap段新的结束地址，也就是重新指定mm_struct结构中的brk字段。

brk系统调用服务例程首先会确定heap段的起始地址min_brk，然后再检查资源的限制问题。接着，将新老heap地址分别按照页大小对齐，对齐后的地址分别存储与newbrk和okdbrk中。

brk()系统调用本身既可以缩小堆大小，又可以扩大堆大小。缩小堆这个功能是通过调用do_munmap()完成的。如果要扩大堆的大小，那么必须先通过find_vma_intersection()检查扩大以后的堆是否与已经存在的某个虚拟内存重合，如何重合则直接退出。否则，调用do_brk()进行接下来扩大堆的各种工作。

用户进程调用malloc()会使得内核调用brk系统调用服务例程，因为malloc总是动态的分配内存空间，因此该服务例程此时会进入第二条执行路径中，即扩大堆。do_brk()主要完成以下工作：

1.通过get_unmapped_area()在当前进程的地址空间中查找一个符合len大小的线性区间，并且该线性区间的必须在addr地址之后。如果找到了这个空闲的线性区间，则返回该区间的起始地址，否则返回错误代码-ENOMEM；

2.通过find_vma_prepare()在当前进程所有线性区组成的红黑树中依次遍历每个vma，以确定上一步找到的新区间之前的线性区对象的位置。如果addr位于某个现存的vma中，则调用do_munmap()删除这个线性区。如果删除成功则继续查找，否则返回错误代码。

3.目前已经找到了一个合适大小的空闲线性区，接下来通过vma_merge()去试着将当前的线性区与临近的线性区进行合并。如果合并成功，那么该函数将返回prev这个线性区的vm_area_struct结构指针，同时结束do_brk()。否则，继续分配新的线性区。

4.接下来通过kmem_cache_zalloc()在特定的slab高速缓存vm_area_cachep中为这个线性区分配vm_area_struct结构的描述符。

5.初始化vma结构中的各个字段。

6.更新mm_struct结构中的vm_total字段，它用来同级当前进程所拥有的vma数量。

7.如果当前vma设置了VM_LOCKED字段，那么通过mlock_vma_pages_range()立即为这个线性区分配物理页框。否则，do_brk()结束。

可以看到，do_brk()主要是为当前进程分配一个新的线性区，在没有设置VM_LOCKED标志的情况下，它不会立刻为该线性区分配物理页框，而是通过vma一直将分配物理内存的工作进行延迟，直至发生缺页异常。

---

mmap()的作用是向操作系统申请一段虚拟地址空间，当然这块虚拟地址空间可以映射到某个文件（这也是这个系统调用的最初作用），当它不将地址空间映射到某个文件时，我们称为匿名空间，匿名空间就可以拿来作为堆分配。

关于mmap函数的详细介绍可以看后文的参考。

``` c
 #include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
int munmap(void *addr, size_t length);

prot：期望的内存保护标志，不能与文件的打开模式冲突。是以下的某个值，可以通过or运算合理地组合在一起
PROT_EXEC
Pages may be executed.
PROT_READ
Pages may be read.
PROT_WRITE
Pages may be written.
PROT_NONE
Pages may not be accessed.
```
flags：指定映射对象的类型，映射选项和映射页是否可以共享。它的值可以是一个或者多个以下位的组合体

MAP_FIXED //使用指定的映射起始地址，如果由start和len参数指定的内存区重叠于现存的映射空间，重叠部分将会被丢弃。如果指定的起始地址不可用，操作将会失败。并且起始地址必须落在页的边界上。

MAP_SHARED //与其它所有映射这个对象的进程共享映射空间。对共享区的写入，相当于输出到文件。直到msync()或者munmap()被调用，文件实际上不会被更新。

MAP_PRIVATE //建立一个写入时拷贝的私有映射。内存区域的写入不会影响到原文件。这个标志和以上标志是互斥的，只能使用其中一个。

MAP_DENYWRITE //这个标志被忽略。

MAP_EXECUTABLE //同上

MAP_NORESERVE //不要为这个映射保留交换空间。当交换空间被保留，对映射区修改的可能会得到保证。当交换空间不被保留，同时内存不足，对映射区的修改会引起段违例信号。

MAP_LOCKED //锁定映射区的页面，从而防止页面被交换出内存。

MAP_GROWSDOWN //用于堆栈，告诉内核VM系统，映射区可以向下扩展。

MAP_ANONYMOUS //匿名映射，映射区不与任何文件关联。

MAP_ANON //MAP_ANONYMOUS的别称，不再被使用。

MAP_FILE //兼容标志，被忽略。  

MAP_32BIT //将映射区放在进程地址空间的低2GB，MAP_FIXED指定时会被忽略。当前这个标志只在x86-64平台上得到支持。

MAP_POPULATE //为文件映射通过预读的方式准备好页表。随后对映射区的访问不会被页违例阻塞。

MAP_NONBLOCK //仅和MAP_POPULATE一起使用时才有意义。不执行预读，只为已存在于内存中的页面建立页表入口。

fd：有效的文件描述词。如果MAP_ANONYMOUS被设定，为了兼容问题，其值应为-1。

offset：被映射对象内容的起点。
下面的do_mmap 函数是mmap函数的实现。


``` c
static inline unsigned long do_mmap(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot,
	unsigned long flag, unsigned long offset)
{
	unsigned long ret = -EINVAL;
	if ((offset + PAGE_ALIGN(len)) < offset)
		goto out;
	if (!(offset & ~PAGE_MASK))
		ret = do_mmap_pgoff(file, addr, len, prot, flag, offset >> PAGE_SHIFT);
out:
	return ret;
}
```
内存的虚拟页和物理页面的数据结构
``` c
/*
 * This struct defines a memory VMM memory area. There is one of these
 * per VM-area/task.  A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
	struct mm_struct * vm_mm;	/* The address space we belong to. */
	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;

	pgprot_t vm_page_prot;		/* Access permissions of this VMA. */
	unsigned long vm_flags;		/* Flags, see mm.h. */

	struct rb_node vm_rb;

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap prio tree, or
	 * linkage to the list of like vmas hanging off its node, or
	 * linkage of vma in the address_space->i_mmap_nonlinear list.
	 */
	union {
		struct {
			struct list_head list;
			void *parent;	/* aligns with prio_tree_node parent */
			struct vm_area_struct *head;
		} vm_set;

		struct raw_prio_tree_node prio_tree_node;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	struct list_head anon_vma_node;	/* Serialized by anon_vma->lock */
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */

	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops;

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units, *not* PAGE_CACHE_SIZE */
	struct file * vm_file;		/* File we map to (can be NULL). */
	void * vm_private_data;		/* was vm_pte (shared mem) */
	unsigned long vm_truncate_count;/* truncate_count or restart_addr */

#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
};

struct mm_struct {
	struct vm_area_struct * mmap;		/* list of VMAs */
	struct rb_root mm_rb;
	struct vm_area_struct * mmap_cache;	/* last find_vma result */
	unsigned long (*get_unmapped_area) (struct file *filp,
				unsigned long addr, unsigned long len,
				unsigned long pgoff, unsigned long flags);
	void (*unmap_area) (struct mm_struct *mm, unsigned long addr);
	unsigned long mmap_base;		/* base of mmap area */
	unsigned long task_size;		/* size of task vm space */
	unsigned long cached_hole_size; 	/* if non-zero, the largest hole below free_area_cache */
	unsigned long free_area_cache;		/* first hole of size cached_hole_size or larger */
	pgd_t * pgd;
	atomic_t mm_users;			/* How many users with user space? */
	atomic_t mm_count;			/* How many references to "struct mm_struct" (users count as 1) */
	int map_count;				/* number of VMAs */
	struct rw_semaphore mmap_sem;
	spinlock_t page_table_lock;		/* Protects page tables and some counters */

	struct list_head mmlist;		/* List of maybe swapped mm's.	These are globally strung
						 * together off init_mm.mmlist, and are protected
						 * by mmlist_lock
						 */

	/* Special counters, in some configurations protected by the
	 * page_table_lock, in other configurations by being atomic.
	 */
	mm_counter_t _file_rss;
	mm_counter_t _anon_rss;

	unsigned long hiwater_rss;	/* High-watermark of RSS usage */
	unsigned long hiwater_vm;	/* High-water virtual memory usage */

	unsigned long total_vm, locked_vm, shared_vm, exec_vm;
	unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
	unsigned long start_code, end_code, start_data, end_data;
	unsigned long start_brk, brk, start_stack;
	unsigned long arg_start, arg_end, env_start, env_end;

	unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

	struct linux_binfmt *binfmt;

	cpumask_t cpu_vm_mask;

	/* Architecture-specific MM context */
	mm_context_t context;

	/* Swap token stuff */
	/*
	 * Last value of global fault stamp as seen by this process.
	 * In other words, this value gives an indication of how long
	 * it has been since this task got the token.
	 * Look at mm/thrash.c
	 */
	unsigned int faultstamp;
	unsigned int token_priority;
	unsigned int last_interval;

	unsigned long flags; /* Must use atomic bitops to access the bits */

	struct core_state *core_state; /* coredumping support */
#ifdef CONFIG_AIO
	spinlock_t		ioctx_lock;
	struct hlist_head	ioctx_list;
#endif
#ifdef CONFIG_MM_OWNER
	/*
	 * "owner" points to a task that is regarded as the canonical
	 * user/owner of this mm. All of the following must be true in
	 * order for it to be changed:
	 *
	 * current == mm->owner
	 * current->mm != mm
	 * new_owner->mm == mm
	 * new_owner->alloc_lock is held
	 */
	struct task_struct *owner;
#endif

#ifdef CONFIG_PROC_FS
	/* store ref to file /proc/<pid>/exe symlink points to */
	struct file *exe_file;
	unsigned long num_exe_file_vmas;
#endif
#ifdef CONFIG_MMU_NOTIFIER
	struct mmu_notifier_mm *mmu_notifier_mm;
#endif
};
 
```
## 堆分配算法

### 1.空闲链表

空闲链表的方法实际上就是把堆中各个空闲的块按照链表的方式连接起来，当用户请求一块空间时，可以遍历整个列表，直到找个合适大小的块将它拆分；当用户释放时将释放空间进行合并。

空闲链表是这样的一种结构，在堆里的每个空闲空间的开头有一个头（header）,头结构里记录了上一个（prev）和下一个（next）空闲块的地址，也就是说所有的空闲块行程了一个链表。

### 2.位图
其核心思想就是将整个堆划分为大量的块（block）,每个块的大小相同。当用户请求内存的时候，总是分配整数个块的空间给用户，第一个块成为已分配区域的头(head)，其余的成为已分配区域的主体(body)。我们可以使用一个整数数组来记录块的使用情况，由于每个块只有头/主体/空闲三张状态，因此仅仅需要两位即可表示一个块，因此成为位图。

![]( /img/blog/bitmap.png)

这个堆分配了3片内存，分别有2/4/1个块，用虚线框表示。其对应的位图将是:

	11 00 00 10 10 10 11 00 00 00 00 00 00 00 00 10 11

其中11表示H(head), 10 表示主体B(Body), 00 表示空闲F(Free)

> 实现方式的优点：

- 速度快：由于整个堆的空闲信息存储在一个数组内，因此访问该数组时，cache容易命中。
- 稳定性好：为了避免用户越界读写而破坏了数据，我们只需简单地备份一下位图即可。而且即使部分数据被破坏，也不会导致整个堆无法工作。
- 块不需要格外信息，易于管理。

> 缺点

- 分配内存的时候容易产生碎片
- 如果堆很大，或者设定的一个块很小，那么位图将很大，可能失去cache命中率高的优势，而且会浪费一定的空间。针对这种情况，可以使用多级位图。  



## glibc malloc 实现

dlmalloc是目前一个十分流行的内存分配器，其由Doug Lea 从1987年开始编写，到目前为止，最新版本为2.8.3 ，由于其高效率等特点被广泛的使用和研究（很多linux系统等用的就是dlmalloc或其变形，比如ptmalloc ）。

> dlmalloc采用所谓的边界标记法将内存划分成很多块，从而对内存的分配与回收进行管理。

``` c

struct malloc_chunk {

  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};
 
```

我们先来看看只考虑使用结构体malloc_chunk管理内存的情况，如下图所示：

![]( /img/blog/malloc_chunk.png)

可以有多个连续的被使用中chunk块，但是不会有多个连续的空闲chunk块，因为连续的多个空闲chunk块可以合并成一个大的空闲chunk块。
按照边界标记法，结构体malloc_chunk通过字段head和prev_foot将内存分割成很多块，从图1中①所示。字段head记录与本块相关的信息，这包括本chunk块大小，本块是否在使用中，前一chunk块是否在使用中。head一个字段就能存储这么多信息是因为dlmalloc在分割内存的时候总是以地址对齐（默认是8字节，可以自由设置，但是8字节是最小值并且设置的值必须是2为底的幂函数值，即是alignment = 2^n，n为整数且n>=3）的方式来进行的，所以用head来存储本chunk块大小字节数的话，其末3bit位总是0，因此这三位可以用来存储其它信息，比如：
以第0位作为标志位，标记前一chunk块是否在使用中，为1表示使用，为0表示空闲。

我们来看看它们的各自相关判断代码：

	#define SIZE_T_ONE          ((size_t)1)
	#define SIZE_T_TWO          ((size_t)2)
	#define PINUSE_BIT          (SIZE_T_ONE)
	#define CINUSE_BIT          (SIZE_T_TWO)
	#define cinuse(p)           ((p)->head & CINUSE_BIT)
	#define pinuse(p)           ((p)->head & PINUSE_BIT)

prev_foot字段虽然在当前chunk块结构体内，记录的却是前一个邻接chunk块的信息（有特例存在，马上会讲到），这样做的好处就是我们通过本块chunk结构体就可以直接获取到前一chunk块的信息，从而方便做进一步的处理操作。相对的，当前chunk块的foot信息就存在于下一个邻接chunk块的结构体内。
``` c

malloc_chunk details:

(The following includes lightly edited explanations by Colin Plumb.)

Chunks of memory are maintained using a `boundary tag' method as
described in e.g., Knuth or Standish.  (See the paper by Paul
Wilson ftp://ftp.cs.utexas.edu/pub/garbage/allocsrv.ps for a
survey of such techniques.)  Sizes of free chunks are stored both
in the front of each chunk and at the end.  This makes
consolidating fragmented chunks into bigger chunks very fast.  The
size fields also hold bits representing whether chunks are free or
in use.

An allocated chunk looks like this:


chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Size of previous chunk, if allocated            | |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Size of chunk, in bytes                       |M|P|
  mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             User data starts here...                          .
    .                                                               .
    .             (malloc_usable_size() bytes)                      .
    .                                                               |
nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Size of chunk                                     |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+


Where "chunk" is the front of the chunk for the purpose of most of
the malloc code, but "mem" is the pointer that is returned to the
user.  "Nextchunk" is the beginning of the next contiguous chunk.

Chunks always begin on even word boundaries, so the mem portion
(which is returned to the user) is also on an even word boundary, and
thus at least double-word aligned.

Free chunks are stored in circular doubly-linked lists, and look like this:

chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Size of previous chunk                            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
`head:' |             Size of chunk, in bytes                         |P|
  mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Forward pointer to next chunk in list             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Back pointer to previous chunk in list            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |             Unused space (may be 0 bytes long)                .
    .                                                               .
    .                                                               |
nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
`foot:' |             Size of chunk, in bytes                           |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

```
在深入理解操作系统那本书里有关于这个分配器算法的介绍，再次就不再详细说了。（其实我也不太懂。。囧。。）

## 参考

- [brk() 函数](http://blog.csdn.net/jltxgcy/article/details/44150429)
- [brk() 函数解释](http://edsionte.com/techblog/archives/4174)
- [mmap() 函数详解](http://blog.chinaunix.net/uid-26669729-id-3077015.html)
- [dlmalloc() 函数详解](http://blog.csdn.net/larryliuqing/article/details/7224672)