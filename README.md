# ucore_os_labs_luolinbin
ucore_os_labs
# Lab2 Report
#### 计43 沈天成 2014010646
---

### 练习1：实现 first-fit 连续物理内存分配算法

Ucore使用结构体Page描述页的属性。Page的定义如下：
```c
struct Page {
    int ref;                        // page frame's reference counter
    uint32_t flags;                 // array of flags that describe the status of the page frame
    unsigned int property;          // the num of free block, used in first fit pm manager
    list_entry_t page_link;         // free list link
};
```
其中，当property不为0时，该页为一段连续空闲页面的首页；当property为0时，该页不空闲或不是一段连续空闲页面的首页。free list仅链接连续空闲页面的首页，而不是所有空闲页面。

为实现first_fit内存分配算法，修改default_pmm.c中的default_alloc_pages函数和default_free_pages函数如下。

```c
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if (p->property >= n) {
            page = p;
            break;
        }
    }
    if (page != NULL) {
    	struct Page *p = page;
        for(; p != page + n; p++) {
            ClearPageProperty(p);
            SetPageReserved(p);
        }
        if (page->property > n) {
            p = page + n;
            p->property = page->property - n;
            page->property = n;
            SetPageProperty(p);
            list_add(&page->page_link, &(p->page_link));
        }
        list_del(&(page->page_link));
        nr_free -= n;
    }
    return page;
}
```
default_alloc_pages函数完成了内存的分配，具体实现为：
1. 判断总的空闲地址空间是否大于所需空间。
2. 从前向后遍历free_list，直至找到第一片不小于所需空间大小的连续空闲页面。
3. 对这连续的n页，修改PageProperty位、PageReserved位。
4. 若该空闲内存块大小大于所需，则将剩余的小内存块插入链表，并设置相应属性。
5. 从链表中删除此内存块，修改空闲地址空间nr_free的值。

```c
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        p->flags = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);

    list_entry_t *le = list_next(&free_list);
    while (le != &free_list) {
        p = le2page(le, page_link);
        if(p > base) break;
        le = list_next(le);
    }
    list_add_before(le, &(base->page_link));

    le = list_next(&(base->page_link));
    if(le != &free_list) {
       	p = le2page(le, page_link);
       	if(base + base->property == p) {
   			base->property += p->property;
   			ClearPageProperty(p);
   			list_del(&(p->page_link));
       	}
    }

    le = list_prev(&(base->page_link));
    if(le != &free_list) {
    	p = le2page(le, page_link);
    	if(p + p->property == base) {
    		p->property += base->property;
    		ClearPageProperty(base);
    		list_del(&(base->page_link));
    	}
    }
    nr_free += n;
}
```
default_alloc_pages函数完成了内存的释放，具体实现为：
1. 对于要释放的n页，修改其标志位。
2. 从前向后遍历free_list，查找在 base 之后的第一个空闲块，将 base 插入到该块之前（实际为该块的首页之前）。
3. 在free_list中，分别查询后一个、前一个内存块是否空闲，若空闲，则进行合并、将后一个块从链表中删除、修改前一个块的大小。
4. 修改空闲地址空间nr_free的值。

> 你的first fit算法是否有进一步的改进空间

与答案相比，我的实现仅将连续空闲页的首页存入free_list，而不是将所有空闲页面存入free_list，因而效率较高。
在释放内存空间时，我的算法仍有改进空间：在我的实现中，需要从前向后依次遍历free_list来找到待释放内存块的位置，效率较低；如果使用平衡树等数据结构来维护空闲地址空间，则可以加快这一查找过程，改进算法。

---

### 练习2：实现寻找虚拟地址对应的页表项

为实现寻找虚拟地址对应的页表项，修改pmm.c中的get_pte函数如下。

```c
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
    pde_t *pdep = &pgdir[PDX(la)];          // (1) find page directory entry
    if (!(*pdep & PTE_P)) {                 // (2) check if entry is not present
        if (!create) return NULL;           // (3) check if creating is needed, then alloc page for page table
        struct Page *page = alloc_page();
        if (page == NULL) return NULL;      // CAUTION: this page is used for page table, not for common data page
        set_page_ref(page, 1);              // (4) set page reference
        uintptr_t pa = page2pa(page);       // (5) get linear address of page
        memset(KADDR(pa), 0, PGSIZE);       // (6) clear page content using memset
        *pdep = pa | PTE_P | PTE_W | PTE_U; // (7) set page directory entry's permission
    }
    return  &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)]; // (8) return page table entry
}
```

函数的实现过程为：
1. 根据页目录表基址和虚地址的前十位找到PDE。
2. 判断PDE是否在内存中、是否需要创建二级页表。
3. 对于需要创建二级页表的情形，分配一个页，清空内容并设置标志位。
4. 根据虚地址的中间十位，找到对应的页表项并返回。

> 请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。

```
#define PTE_P           0x001 // 是否存在
#define PTE_W           0x002 // 是否可写
#define PTE_U           0x004 // 用户是否可获取
#define PTE_PWT         0x008 // 写直达缓存
#define PTE_PCD         0x010 // 禁用缓存
#define PTE_A           0x020 // 是否被访问过
#define PTE_D           0x040 // 是否为脏页
#define PTE_PS          0x080 // 页大小
#define PTE_MBZ         0x180 // 必须为0
#define PTE_AVAIL       0xE00 // 软件使用的位
```

> 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

硬件首先保存现场、将相关信息存入相应寄存器，接着执行中断服务例程处理异常。最后恢复现场，回到出现异常的语句重新执行。

---

### 练习3：释放某虚地址所在的页并取消对应二级页表项的映射

修改pmm.c中的page_remove_pte函数如下。

```c
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
    if (*ptep & PTE_P) {                          //(1) check if this page table entry is present
        struct Page *page =  pte2page(*ptep);     //(2) find corresponding page to pte
        page_ref_dec(page);                       //(3) decrease page reference
        if (page_ref(page) == 0) free_page(page); //(4) and free this page when page reference reachs 0
        *ptep = 0;                                //(5) clear second page table entry
        tlb_invalidate(pgdir, la);                //(6) flush tlb
    }
}
```

函数的实现过程为：
1. 判断页表项是否在内存中。
2. 根据页表项找到相应的页，将其被引用次数减一。若被引用次数减至0，则释放该页。
3. 清除页表项，更新TLB。

> 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

Page的全局变量pages对应每一页的虚拟地址，而页表项和页目录项保存了对应的物理地址。可以通过pde2page、pte2page函数完成pde、pte中的页到pages中的页的映射。

> 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？

```
 * Virtual memory map:                                          Permissions
 *                                                              kernel/user
 *
 *     4G ------------------> +---------------------------------+
 *                            |                                 |
 *                            |         Empty Memory (*)        |
 *                            |                                 |
 *                            +---------------------------------+ 0xFB000000
 *                            |   Cur. Page Table (Kern, RW)    | RW/-- PTSIZE
 *     VPT -----------------> +---------------------------------+ 0xFAC00000
 *                            |        Invalid Memory (*)       | --/--
 *     KERNTOP -------------> +---------------------------------+ 0xF8000000
 *                            |                                 |
 *                            |    Remapped Physical Memory     | RW/-- KMEMSIZE
 *                            |                                 |
 *     KERNBASE ------------> +---------------------------------+ 0xC0000000
 *                            |                                 |
 *                            |                                 |
 *                            |                                 |
 *                            ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

如图，将memlayout.h中KERNBASE的值改为0即可使得虚拟地址和物理地址相等。
```
#define KERNBASE     0x00000000
```

其他地方也需要相应地进行修改，例如kernel.ld：
```
/* Load the kernel at this address: "." means the current address */
. = 0x00100000;
```

---

### 与参考答案的差异

在练习一中，参考答案将所有空闲页面存入free_list，而我的实现仅将连续空闲页的首页存入free_list，在分配和释放时也仅需查找连续空闲页的首页，效率更高。

### 实验涉及的知识点
1. 连续物理内存分配的first-fit算法
2. 虚拟地址与物理地址的转换
3. 多级页表的建立和索引
4. 页表的释放

实验中的知识点与原理相互对应。

### OS原理中很重要，但在实验中没有对应上的知识点
1. 其他连续物理内存分配算法，如best fit, worst fit, buddy system等
2. 反置页表
