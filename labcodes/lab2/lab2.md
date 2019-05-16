## Lab2实验报告  

### 知识点  
#### 这是与本次lab有关的知识点：
* first-fit 匹配算法  
* x86-32分页机制  
* 二级页表机制  
#### 下面知识点在本次实验中没有对照：  
* 最佳匹配  
* 最差匹配  
* 磁盘碎片的整理  
* 反置页表  
* 页寄存器  

### 练习一  

1. 对相关数据结构的分析：

   ```c
   struct Page {
       int ref;                        // page frame's reference counter
       uint32_t flags;                 // array of flags that describe the status of the page frame
       unsigned int property;          // the num of free block, used in first fit pm manager
       list_entry_t page_link;         // free list link
   };
   ```

   在default_pmm.c中，将空闲内存块按照起始地址从小到大组织成一个双向链表。每一个链表的节点是一个Page中的page_link，可以通过链表节点得到对应的Page指针。

2. 分配、释放操作的实现：

   1. 分配操作：

      ```c
      struct Page *page = NULL;
          list_entry_t *le = &free_list;
          while ((le = list_next(le)) != &free_list) {
              struct Page *p = le2page(le, page_link);
              if (p->property > n) {
      			struct Page *np = p + n;	//分配完之后，新的Page所在位置
                  np->property = p->property - n;	//设置新的Page的大小
                  list_add_after(&(p->page_link),&(np->page_link));	//先加入新的Page
                  list_del(&(p->page_link));		//再从链表中删除被分配了的Page
                  page = p;	//旧的Page被作为分配内存，清除标记之后返回首地址。
                  ClearPageProperty(page);
                  nr_free -= n;	//总空闲内存的页数减少n
                  break;
              }else if(p->property == n){	//有一块刚好可以分配出去。
      			page = p;
      			list_del(&(page->page_link));	//直接将其从链表中删除即可。
                  ClearPageProperty(page);	//清楚标记后返回首地址
                  nr_free -= n;	//总空闲内存的页数减少n
      			break;
      		}
          }
      ```

      分配一块大小为n个页大小的内存，first-fit采取的策略是：按照地址从小到大，遍历所有空闲内存块，当找到第一个大小大于等于n的内存块p。之后，按照p的大小为n、p的大小等于n两种情况分别进行处理。

      ​	p的大小为n时，直接从链表中删除p。

      ​	p的大小大于n时，在p+n处重新创建一个Page指针，并将其加入到链表中的p后面，然后再从链表中删除p。

   2. 释放操作：

      ```c
          p = le2page(le, page_link);
          while (le != &free_list) {
              p = le2page(le, page_link);
              le = list_next(le);
              if (base + base->property == p) {	//可以接在当前空闲块前面
                  base->property += p->property;
                  ClearPageProperty(p);
                  list_add_before(&(p->page_link),&(base->page_link));
                  list_del(&(p->page_link));
                  break;
              }else if (base + base->property < p){	//位于当前空闲块前面
                  list_add_before(&(p->page_link),&(base->page_link));
                  break;
              }else if (p + p->property == base) {	//可以接在当前空闲块后面
                  if(le == &free_list){
                      p->property += base->property;
                      ClearPageProperty(base);
                  }else{
                      p->property += base->property;
                      ClearPageProperty(base);
                      np = le2page(le, page_link);
                      if(base + base->property == np){
                          p->property += np->property;
                          ClearPageProperty(np);
                          list_del(&(np->page_link));
                      }
                  }
                  break;
              }else if (le == &free_list){
                  list_add_after(&(p->page_link),&(base->page_link));
                  break;
              }
          }
          nr_free += n;
      ```

      在进行释放操作之前，空闲内存块链表维持了有序。释放操作的目标是维护这种有序性并尽可能合并空闲内存块。所以可以通过遍历所有的内存空闲块，比较对应的地址：

      ​	如果被释放内存块base恰好是当前空闲块p的前面部分，即base+n==p，则说明被释放内存块可以合并在当前空闲块的前面。

      ​	如果被释放内存块base恰好是当前空闲块的后面部分，即p+p->property==base，说明被释放内存块可以合并在当前空闲块后面。这时再继续检查后一个空闲块是否也可以合并进来，即可将三个空闲块合并的情况囊括进来。

      还需要考虑的一种特殊情况是，当前没有空闲内存块，这时链表为空，将释放内存块的标记处理完直接加入即可。

      ---

### 练习二  

1. 实现get_pte需要完成：

    ```c
	pde_t *pdep = &pgdir[PDX(la)];
	pte_t *ptep = 0;
	if(!(*pdep & PTE_P)){
		if(!create){
			return NULL;
		}
		struct Page *p = alloc_page();
		ptep = page2kva(p);
		*pdep = page2pa(p) | PTE_W | PTE_P | PTE_U;
		memset(page2kva(p), 0, PGSIZE);
		set_page_ref(p,1);
	}else{
		ptep = KADDR(PDE_ADDR(*pdep));
	}
	return &ptep[PTX(la)];
    ```

  - 根据给定的线性地址和页目录表，查看相应的页表是否存在，如果不存在，则根据给定的参数create决定是否创建。

  - 在对应的页表中，根据线性地址la找到对应的表项，返回其内核虚拟地址。
2. 页目录项以及页表项的各部分的含义和潜在用处:

   页目录项：

   ```c
          31                                  12 11    8 7 6 5 4  3  2 1 0
         +--------------------------------------+-------+---+-+-+---+-+-+-+
         |                                      |       | |i| |P|   |U|R| |
         |      address of page table 31..12    |ignored|0|g|A|C|PWT|/|/|P|
         |                                      |       | |n| |D|   |S|W| |
         +--------------------------------------+-------+---+-+-+---+-+-+-+
   ```

   各部分的含义：

   * [31:12] : 页表的物理地址的高20位。

   * A : 如果当前页面已经被处理过，则置为1

   * PCD ：禁用缓存(Cache Disable)

   * PWT ： 写直达(Write Through)

   * U/S ：用户态可访问还是内核态可访问

   * R/W ：可读还是可写

   * P ：present位，表示当前页目录项是否有效  

   页表项：
   ```c
          31                                  12 11  9 8 7 6 5 4  3  2 1 0
         +--------------------------------------+-------+---+-+-+---+-+-+-+
         |                                      |     | |P| | |P|   |U|R| |
         |      address of page       31..12    |igno |G|A|D|A|C|PWT|/|/|P|
         |                                      |     | |T| | |D|   |S|W| |
         +--------------------------------------+-------+---+-+-+---+-+-+-+
   ```

   各部分的含义：

   * [31:12] : 物理页的物理地址的高20位
   * G ： Global。标志该页是否是全局的
   * PAT ：
   * D ： 在上次清零之后，该页是否被写过，用于页替换算法。 


3. 如果在访问内存时发生页访问异常，则硬件需要做的事情包括：

   - 保存发生异常的访存地址到cr2寄存器
   - 保存：eflags、cs寄存器、eip寄存器、error信息到栈中。
   - 进入异常处理例程。
---

### 练习三  

1. 实现remove_pte需要完成：

```c
	struct Page *p = pte2page(*ptep);
	if(*ptep & PTE_P){
		page_ref_dec(p);
		*ptep &= ~PTE_P;
		if(p->ref == 0){
			free_page(p);
		}
		tlb_invalidate(pgdir, la);
	}
```

- 如果对应的页表项ptep确实存在（present位为真），则

  - 清除present位

  - 将对应页的被引用数减一，如果被引用数为0则释放该页

  - 刷新tlb的相关内容
2. 页目录项/页表项的高20位存放的是页表/页的物理地址的高20位，以此作为索引，在pages数组中，可以访问到与页对应的数据结构Page，其中包含引用数量等信息。

3. 如果希望虚拟地址与物理地址相等，则需要：

   1. 修改tools/kernel.ld中的起始地址：

      ```shell
      SECTIONS {
          /* Load the kernel at this address: "." means the current address */
          . = 0x00100000;
          ...
      }
      ```

   2. 修改kern/mm/memlayout.h中的KERNBASE

      ```c
      #define KERNBASE            0x00000000
      ```

   3. 注释entry.S中unmap 0~4M地址空间的页表的部分代码

   4. 不调用boot_map_segament函数

   5. 修改某些测试代码，避开对地址0x00000000～0x38000000的访存。我直接将测试函数注释掉之后，可以运行make qemu，时间有限，没有再做进一步修改。

   详细的修改请看<https://github.com/PanQL/ucore_os_lab/commit/5e1f1a3fc3a84d3475e205cdd980339b48d1326d>

