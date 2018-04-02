`struct Page`的定义如下：

```c
/* *
 * struct Page - Page descriptor structures. Each Page describes one
 * physical page. In kern/mm/pmm.h, you can find lots of useful functions
 * that convert Page to other data types, such as phyical address.
 * */
struct Page {
    int ref;                        // page frame's reference counter
    uint32_t flags;                 // array of flags that describe the status of the page frame
    unsigned int property;          // the num of free block, used in first fit pm manager
    list_entry_t page_link;         // free list link
};

/* Flags describing the status of a page frame */
#define PG_reserved                 0      
#define PG_property                 1 

#define SetPageReserved(page)       set_bit(PG_reserved, &((page)->flags))
#define ClearPageReserved(page)     clear_bit(PG_reserved, &((page)->flags))
#define PageReserved(page)          test_bit(PG_reserved, &((page)->flags))
#define SetPageProperty(page)       set_bit(PG_property, &((page)->flags))
#define ClearPageProperty(page)     clear_bit(PG_property, &((page)->flags))
#define PageProperty(page)          test_bit(PG_property, &((page)->flags))
```

- `ref`表示页被引用的计数器，当有一个虚拟页映射到了这个结构描述的物理页，计数器加1，反之减1，当变为0的时候，说明没有虚拟页与该物理页对应，需要设置其标记位表示该页是未分配的
- `flags`表示页的状态标记，有2个标记，0位表示此页是否被保留，若被保留则设置为1，1位表示此页是否是空闲的，若是空闲则设置为1，可以通过`SetPageReserved`等宏进行标记的设置
- `property`表示某连续内存空闲块的大小，即地址连续空闲页的个数，需要设置此变量的页是头页，即连续内存空闲块地址最小的一页，该页为`0`时说明该页不是空闲块的首页
- `page_link`表示把多个连续内存空闲块链接在一起的双向链表指针，用于构建双向链表，下面的`struct free_area_t`就是使用了这一参数将各页穿在一起
- 后面的宏定义很容易看懂，就是对`flags`的设置和检测，这里不再赘述

管理空闲块的双向链表结构`free_area_t`：

```c
/* free_area_t - maintains a doubly linked list to record free (unused) pages */
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```

`free_list`为这个链表的表头，

`nr_free`为这个链表中空闲页的数目。

实验中实现`First Fit`算法时对该结构进行了操作，主要是在函数`default_init`，`default_init_memmap`，`default_alloc_pages`， `default_free_pages`。

首先，`default_init`函数，对链表初始化，然后设置页数为`0`

```c
static void
default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
```

`default_init_memmap`函数用于初始化空闲页链表，对每一个空闲页进行初始化，并更新空闲页总数。

```c
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));   //确认该页是否为保留页
        p->flags = 0;              //清空该页的标志位
        SetPageProperty(p);        //使能该页的空闲标志位
        p->property = 0;           //设置property为0，后面通过设置base页的property为n，来实现空闲块的第一页property为空闲块页数，其他页的property为0
        set_page_ref(p, 0);        //清空引用
        list_add_before(&free_list, &(p->page_link)); //插入到空闲块链表中
    }
    base->property = n;  //实现空闲块的第一页property为空闲块页数，其他页的property为0
    nr_free += n;        //更新空闲页总数
}
```

然后是`default_alloc_pages`函数，该函数是实现`FFMA`算法的重要一步——分配内存页，从开头按照地址顺序搜索，在发现一块不小于需要的大小的空闲块时，将该块的需要大小部分分配出去，并返回起始地址。

```c
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    // 如果需要的页数大于总的空闲页数，则直接返回无法分配
    if (n > nr_free) {
        return NULL;
    }
    list_entry_t *le = &free_list;
    list_entry_t *len;
    while ((le = list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        //顺次寻找第一个大小足够的空闲块
        if (p->property >= n) {
            int i;
            //找到后将开头的n个页面设置为已用，并将其从空闲块链表中删除
            for (i = 0; i < n; ++i) {
                len = list_next(le);
                struct Page *p_temp = le2page(le, page_link);
                SetPageReserved(p_temp);
                ClearPageProperty(p_temp);
                list_del(le);
                le = len;
            }
            //如果空闲块大小大于n，则剩下的部分构成一个新的空闲块，将起始页的property设置为新的空闲块的大小
            if (p->property > n) {
                struct Page* p_temp = le2page(le, page_link);
                p_temp->property = p->property - n;
            }
            ClearPageProperty(p);
            SetPageReserved(p);
            //更新空闲页总数
            nr_free -= n;
            //返回分配到的块的起始页地址
            return p;
        }
    }
    return NULL;
}
```

然后是释放内存页的`default_free_pages`函数。释放后，需要首先从空闲块链表中找到该被释放的块应当放置的位置，然后将所有的页面释放，重置各个参数，然后检查该块后面和前面是否有空闲块，如果有的话，就需要将相邻的空闲块合并。

```c
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    assert(PageReserved(base));
    struct Page *pp, *p = base;

    list_entry_t *le = &free_list;
    //寻找第一个大于地址base的空闲块地址
    while ((le = list_next(le)) != &free_list) {
        pp = le2page(le, page_link);
        if (pp > p) {
            break;
        }
    }

    //将需要释放的块的各个页面插入到该位置
    for (;p != base + n; ++p) {
        list_add_before(le, &(p->page_link));
    }

    //如果base所在的块不能和前后合并，那么base的各个参数的设置
    base->flags = 0;
    set_page_ref(base, 0);
    SetPageProperty(base);
    base->property = n;

    //如果base+n==p，说明base所在的空闲块和后面p所在的空闲块在地址上是相邻的，可以合并
    p = le2page(le, page_link);
    if (base+n == p) {
        base->property += p->property;
        p->property = 0;
    }

    le = list_prev(&(base->page_link));
    p = le2page(le, page_link);


    //如果p==base-1，说明base前面的页面也是空闲的，那么就只需要向前找到前面这个空闲块的起始页面，然后将这两个块合并即可。
    if (le != &free_list && p == base - 1) {
        while (le != &free_list) {
            if (p->property != 0) {
                p->property += base->property;
                base->property = 0;
                break;
            }
            le = list_prev(le);
            p = le2page(le, page_link);
        }
    }
    //更新空闲页总数
    nr_free += n;
    return;
}
```

