# 深入理解 Linux 物理内存分配全链路实现

### 前文回顾

在上篇文章 [《深入理解 Linux 物理内存管理》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486879&idx=1&sn=0bcc59a306d59e5199a11d1ca5313743&chksm=ce77cbd8f90042ce06f5086b1c976d1d2daa57bc5b768bac15f10ee3dc85874bbeddcd649d88#rd)中，笔者详细的为大家介绍了 Linux 内核如何对物理内存进行管理以及相关的一些内核数据结构。

在介绍物理内存管理之前，笔者先从 CPU 的角度开始，介绍了三种 Linux 物理内存模型：FLATMEM 平坦内存模型，DISCONTIGMEM 非连续内存模型，SPARSEMEM 稀疏内存模型。

![image](image/ad95e678bf9193e47340196080dc5e33.png)

![image](image/19a7c1edfd87b64f29e6877c80293c24.png)

![image](image/1d33fec43c33320effd170679565e41d.png)

随后笔者又带大家站在一个新的视角上，把物理内存看做成一个整体，从 CPU 访问物理内存以及 CPU 与物理内存的相对位置变化的角度介绍了两种物理内存架构：一致性内存访问 UMA 架构，非一致性内存访问 [NUMA](https://so.csdn.net/so/search?q=NUMA&spm=1001.2101.3001.7020) 架构。

![image](image/867bb12a5e75d48cf3f58571c66c2463.png)

![image](image/3f61d76fdec80777f4bf368e1a789464.png)

> 在 NUMA 架构下，只有 DISCONTIGMEM 非连续内存模型和 SPARSEMEM 稀疏内存模型是可用的。而 UMA 架构下，前面介绍的三种内存模型可以配置使用。

无论是 NUMA 架构还是 UMA 架构在内核中都是使用相同的数据结构来组织管理的，在内核的内存管理模块中会把 UMA 架构当做只有一个 NUMA 节点的伪 NUMA 架构。

![image](image/8a5a548da361ace924a08dc52b68d561.png)

这样一来这两种架构模式就在内核中被统一管理起来，我们基于这个事实，深入剖析了内核针对 NUMA 架构下用于物理内存管理的相关数据结构：struct pglist\_data （NUMA 节点），struct [zone](https://so.csdn.net/so/search?q=zone&spm=1001.2101.3001.7020)（物理内存区域），struct page（物理页）。

![image](image/6d9819486947ce314d26ea83d90655d2.png)

上图展示的是在 NUMA 架构下，NUMA 节点与物理内存区域 zone 以及物理内存页 page 之间的层次关系。

物理内存被划分成了一个一个的内存节点（NUMA 节点），在每个 NUMA 节点内部又将其所管理的物理内存按照功能不同划分成了不同的内存区域 zone ，每个内存区域 zone 管理一片用于具体功能的物理内存页 page，而内核会为每一个内存区域分配一个伙伴系统用于管理该内存区域下物理内存页 page 的分配和释放。

> 物理内存在内核中管理的层级关系为：`None -> Zone -> page`

在上篇文章的最后，笔者又花了大量的篇幅来为大家介绍了 struct page 结构，我们了解了内核如何通过 struct page 结构来描述物理内存页，这个结构是内核中最为复杂的一个结构体，因为它是物理内存管理的最小单位，被频繁应用在内核中的各种复杂机制下。

通过以上内容的介绍，笔者觉得大家已经在架构层面上对 Linux 物理内存管理有了一个较为深刻的认识，现在物理内存管理的架构我们已经建立起来了，那么内核如何根据这个架构层次来分配物理内存呢？

为了给大家梳理清楚内核分配物理内存的过程及其涉及到的各个重要模块，于是就有了本文的内容~~

![image](image/99146f94cafa8ad3a67320411444543c.png)

### 1\. 内核物理内存分配接口

![image](image/101ec098fa40c1cb2c2859b3cce0b1d9.png)

在为大家介绍物理内存分配之前，笔者先来介绍下内核中用于物理内存分配的几个核心接口，这几个物理内存分配接口全部是基于伙伴系统的，==伙伴系统有一个特点就是它所分配的物理内存页全部都是物理上连续的，并且只能分配 2 的整数幂个页，这里的整数幂在内核中称之为分配阶。==

==下面要介绍的这些物理内存分配接口均需要指定这个分配阶，意思就是从伙伴系统申请多少个物理内存页，假设我们指定分配阶为 order，那么就会从伙伴系统中申请 2 的 order 次幂个物理内存页。==

内核中提供了一个 alloc\_pages 函数用于分配 2 的 order 次幂个物理内存页，参数中的 unsigned int order 表示向底层伙伴系统指定的分配阶，参数 gfp\_t gfp 是内核中定义的一个用于规范物理内存分配行为的修饰符，这里我们先不展开，后面的小节中笔者会详细为大家介绍。

    struct page *alloc_pages(gfp_t gfp, unsigned int order);

==alloc\_pages 函数用于向底层伙伴系统申请 2 的 order 次幂个物理内存页组成的内存块，该函数返回值是一个 struct page 类型的指针用于指向申请的内存块中第一个物理内存页。==

==alloc\_pages 函数用于分配**多个**连续的物理内存页，在内核的某些内存分配场景中有时候并不需要分配这么多的连续内存页，而是只需要分配一个物理内存页即可，于是内核又提供了 alloc\_page 宏，用于这种**单内存页**分配的场景，我们可以看到其底层还是依赖了 alloc\_pages 函数，只不过 order 指定为 0。==

    #define alloc_page(gfp_mask) alloc_pages(gfp_mask, 0)


当系统中空闲的物理内存无法满足内存分配时，就会导致内存分配失败，alloc\_pages，alloc\_page 就会返回空指针 NULL 。

> vmalloc 分配机制底层就是用的 alloc\_page

在物理内存分配成功的情况下， alloc\_pages，alloc\_page 函数返回的都是指向其申请的物理内存块第一个物理内存页 struct page 指针。

==大家可以直接理解成返回的是一块物理内存，而 CPU 可以直接访问的却是虚拟内存，所以内核又提供了一个函数 \_\_get\_free\_pages ，该函数直接返回物理内存页的虚拟内存地址。用户可以直接使用。==

    unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);

==\_\_get\_free\_pages 函数在使用方式上和 alloc\_pages 是一样的，函数参数的含义也是一样，只不过一个是返回物理内存页的虚拟内存地址，一个是直接返回物理内存页。==

==事实上 \_\_get\_free\_pages 函数的底层也是基于 alloc\_pages 实现的，只不过多了一层虚拟地址转换的工作。==

    unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order)
    {
    	struct page *page;
        // 不能在高端内存中分配物理页，因为无法直接映射获取虚拟内存地址
    	page = alloc_pages(gfp_mask & ~__GFP_HIGHMEM, order);
    	if (!page)
    		return 0;
        // 将直接映射区中的物理内存页转换为虚拟内存地址
    	return (unsigned long) page_address(page);
    }

==page\_address 函数用于将给定的物理内存页 page 转换为它的虚拟内存地址，**不过这里只适用于内核虚拟内存空间中的直接映射区**，因为在直接映射区中虚拟内存地址到物理内存地址是直接映射的，虚拟内存地址减去一个固定的偏移就可以直接得到物理内存地址。==

如果物理内存页处于高端内存中，则不能这样直接进行转换，在通过 alloc\_pages 函数获取物理内存页 page 之后，需要调用 kmap 映射将 page 映射到内核虚拟地址空间中。

![image](image/7ac80dfa9d19d1da5505638411bedb9b.png)

> 忘记这块内容的同学，可以在回看下笔者之前的文章 [《深入理解虚拟内存管理》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486732&idx=1&sn=435d5e834e9751036c96384f6965b328&chksm=ce77cb4bf900425d33d2adfa632a4684cf7a63beece166c1ffedc4fdacb807c9413e8c73f298&token=1274956236&lang=zh_CN#rd)中的 “ 7.1.4 永久映射区 ” 小节。

同 alloc\_page 函数一样，内核也提供了 \_\_get\_free\_page 用于只分配单个物理内存页的场景，底层还是依赖于 \_\_get\_free\_pages 函数，参数 order 指定为 0 。

    #define __get_free_page(gfp_mask) \
    		__get_free_pages((gfp_mask), 0)


无论是 alloc\_pages 也好还是 \_\_get\_free\_pages 也好，它们申请到的内存页中包含的数据在一开始都不是空白的，而是内核随机产生的一些垃圾信息，但其实这些信息可能并不都是完全随机的，很有可能随机的包含一些敏感的信息。

这些敏感的信息可能会被一些黑客所利用，并对计算机系统产生一些危害行为，==所以从使用安全的角度考虑，内核又提供了一个函数 get\_zeroed\_page，顾名思义，这个函数会将从伙伴系统中申请到内存页全部初始化填充为 0 ，这在分配物理内存页给用户空间使用的时候非常有用。==

    unsigned long get_zeroed_page(gfp_t gfp_mask)
    {
    	return __get_free_pages(gfp_mask | __GFP_ZERO, 0);
    }

==get\_zeroed\_page 函数底层也依赖于 \_\_get\_free\_pages，指定的分配阶 order 也是 0，表示从伙伴系统中只申请一个物理内存页并初始化填充 0 。==

除此之外，内核还提供了一个 \_\_get\_dma\_pages 函数，专门用于从 DMA 内存区域分配适用于 DMA 的物理内存页。其底层也是依赖于 \_\_get\_free\_pages 函数。

    unsigned long __get_dma_pages(gfp_t gfp_mask, unsigned int order);

==这些底层依赖于 \_\_get\_free\_pages 的物理内存分配函数，在遇到内存分配失败的情况下都会返回 0 。==

> 以上介绍的物理内存分配函数，分配的均是在物理上连续的内存页。

当然了，有内存的分配就会有内存的释放，所以内核还提供了两个用于释放物理内存页的函数：

    void __free_pages(struct page *page, unsigned int order);
    void free_pages(unsigned long addr, unsigned int order);


-   ==\_\_free\_pages ： 同 alloc\_pages 函数对应，用于释放一个或者 2 的 order 次幂个内存页，释放的物理内存区域起始地址由该区域中的第一个 page 实例指针表示，也就是参数里的 struct page \*page 指针。==
    
-   ==free\_pages：同 \_\_get\_free\_pages 函数对应，与 \_\_free\_pages 函数的区别是在释放物理内存时，使用了虚拟内存地址而不是 page 指针。==

**在释放内存时需要非常谨慎小心，我们只能释放属于你自己的内存页，传递了错误的 struct page 指针或者错误的虚拟内存地址，或者传递错了 order 值，都可能会导致系统的崩溃。在内核空间中，内核是完全信赖自己的，这点和用户空间不同。**

==另外内核也提供了 \_\_free\_page 和 free\_page 两个宏，专门用于释放单个物理内存页。==

    #define __free_page(page) __free_pages((page), 0)
    #define free_page(addr) free_pages((addr), 0)


到这里，关于内核中对于物理内存分配和释放的接口，笔者就为大家交代完了，但是大家可能会有一个疑问，就是我们在介绍 alloc\_pages 和 \_\_get\_free\_pages 函数的时候，它们的参数中都有 gfp\_t gfp\_mask，之前笔者简单的提过这个 gfp\_mask 掩码：它是内核中定义的一个用于规范物理内存分配行为的掩码。

那么这个掩码究竟规范了哪些物理内存的分配行为 ？并对物理内存的分配有哪些影响呢 ？大家跟着笔者的节奏继续往下看~~~

### 2.规范物理内存分配行为的掩码 gfp\_mask

笔者在 [《深入理解 Linux 物理内存管理》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486879&idx=1&sn=0bcc59a306d59e5199a11d1ca5313743&chksm=ce77cbd8f90042ce06f5086b1c976d1d2daa57bc5b768bac15f10ee3dc85874bbeddcd649d88&token=1274956236&lang=zh_CN#rd)一文中的 “ 4.3 NUMA 节点物理内存区域的划分 ” 小节中曾经为大家详细的介绍了 NUMA 节点中物理内存区域 zone 的划分。

笔者在文章中提到，由于实际的计算机体系结构受到硬件方面的制约，间接限制了页框的使用方式。于是内核会根据不同的物理内存区域的功能不同，将 NUMA 节点内的物理内存划分为：ZONE\_DMA，ZONE\_DMA32，ZONE\_NORMAL，ZONE\_HIGHMEM 这几个物理内存区域。

> ZONE\_MOVABLE 区域是内核从逻辑上的划分，该区域中的物理内存页面来自于上述几个内存区域，目的是避免内存碎片和支持内存热插拔

![image](image/03a0d15d48b34da8ff19c5ace43566e7.png)

当我们调用上小节中介绍的那几个物理内存分配接口时，比如：alloc\_pages 和 \_\_get\_free\_pages。就会遇到一个问题，就是我们申请的这些物理内存到底来自于哪个物理内存区域 zone，假如我们想要从指定的物理内存区域中申请内存，我们该如何告诉内核呢 ?

    struct page *alloc_pages(gfp_t gfp, unsigned int order);
    unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);

==这时，这些物理内存分配接口中的 gfp\_t 参数就派上用场了，前缀 gfp 是 get free page 的缩写，意思是在获取空闲物理内存页的时候需要指定的分配掩码 gfp\_mask。==

==gfp\_mask 中的低 4 位用来表示应该从哪个物理内存区域 zone 中获取内存页 page。==

![image](image/b17fc520328b345a451e0887d2fbc039.png)

gfp\_mask 掩码中这些区域修饰符 zone modifiers 定义在内核 `/include/linux/gfp.h` 文件中：

    #define ___GFP_DMA		    0x01u
    #define ___GFP_HIGHMEM		0x02u
    #define ___GFP_DMA32		0x04u
    #define ___GFP_MOVABLE		0x08u


大家这里可能会感到好奇，为什么没有定义 \_\_\_GFP\_NORMAL 的掩码呢？

==这是因为内核对物理内存的分配主要是落在 ZONE\_NORMAL 区域中，如果我们不指定物理内存的分配区域，那么内核会默认从 ZONE\_NORMAL 区域中分配内存，如果 ZONE\_NORMAL 区域中的空闲内存不够，内核则会降级到 ZONE\_DMA 区域中分配。==

==关于物理内存分配的区域降级策略==，笔者在前面的文章[《深入理解 Linux 物理内存管理》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486879&idx=1&sn=0bcc59a306d59e5199a11d1ca5313743&chksm=ce77cbd8f90042ce06f5086b1c976d1d2daa57bc5b768bac15f10ee3dc85874bbeddcd649d88&token=1274956236&lang=zh_CN#rd)的 “ 5.1 物理内存区域中的预留内存 ” 小节中已经详细地为大家介绍过了，但是之前的介绍只是停留在理论层面，那么这个物理内存区域降级策略是在哪里实现的呢？接下来的内容笔者就为大家揭晓~~~

==内核在 `/include/linux/gfp.h` 文件中定义了一个叫做 gfp\_zone 的函数，这个函数用于将我们在物理内存分配接口中指定的 gfp\_mask 掩码转换为物理内存区域，返回的这个物理内存区域是内存分配的最高级内存区域，如果这个最高级内存区域不足以满足内存分配的需求，则按照 `ZONE_HIGHMEM -> ZONE_NORMAL -> ZONE_DMA` 的顺序依次降级。==

```C
static inline enum zone_type gfp_zone(gfp_t flags)
{
	enum zone_type z;
	int bit = (__force int) (flags & GFP_ZONEMASK);

	z = (GFP_ZONE_TABLE >> (bit * GFP_ZONES_SHIFT)) &
					 ((1 << GFP_ZONES_SHIFT) - 1);
	VM_BUG_ON((GFP_ZONE_BAD >> bit) & 1);
	return z;
}
```


上面的这个 gfp\_zone 函数是在内核 5.19 版本中的实现，在高版本的实现中用大量的移位操作替换了低版本中的实现，目的是为了提高程序的性能，但是带来的却是可读性的大幅下降。

笔者写到这里觉得给大家分析清楚每一步移位操作的实现对大家理解这个函数的主干逻辑并没有什么实质意义上的帮助，并且和本文主题偏离太远，所以我们退回到低版本 2.6.24 中的实现，在这一版中直击 gfp\_zone 函数原本的面貌。

```C
static inline enum zone_type gfp_zone(gfp_t flags)
{
	int base = 0;

#ifdef CONFIG_NUMA
	if (flags & __GFP_THISNODE)
		base = MAX_NR_ZONES;
#endif

#ifdef CONFIG_ZONE_DMA
	if (flags & __GFP_DMA)
		return base + ZONE_DMA;
#endif
#ifdef CONFIG_ZONE_DMA32
	if (flags & __GFP_DMA32)
		return base + ZONE_DMA32;
#endif
	if ((flags & (__GFP_HIGHMEM | __GFP_MOVABLE)) ==
			(__GFP_HIGHMEM | __GFP_MOVABLE))
		return base + ZONE_MOVABLE;
#ifdef CONFIG_HIGHMEM
	if (flags & __GFP_HIGHMEM)
		return base + ZONE_HIGHMEM;
#endif
    // 默认从 normal 区域中分配内存
	return base + ZONE_NORMAL;
}
```

==我们看到在内核 2.6.24 版本中的 gfp\_zone 函数实现逻辑就非常的清晰了，核心逻辑主要如下：==

-   只要掩码 flags 中设置了 \_\_GFP\_DMA，则不管 \_\_GFP\_HIGHMEM 有没有设置，内存分配都只会在 ZONE\_DMA 区域中分配。
    
-   如果掩码只设置了 ZONE\_HIGHMEM，则在物理内存分配时，优先在 ZONE\_HIGHMEM 区域中进行分配，如果容量不够则降级到 ZONE\_NORMAL 中，如果还是不够则进一步降级至 ZONE\_DMA 中分配。
    
-   如果掩码既没有设置 ZONE\_HIGHMEM 也没有设置 \_\_GFP\_DMA，则走到最后的分支，默认优先从 ZONE\_NORMAL 区域中进行内存分配，如果容量不够则降级至 ZONE\_DMA 区域中分配。
    
-   ==单独设置 \_\_GFP\_MOVABLE 其实并不会影响内核的分配策略，我们如果想要让内核在 ZONE\_MOVABLE 区域中分配内存需要同时指定 \_\_GFP\_MOVABLE 和 \_\_GFP\_HIGHMEM 。==

ZONE\_MOVABLE 只是内核定义的一个虚拟内存区域，目的是避免内存碎片和支持内存热插拔。上述介绍的 ZONE\_HIGHMEM，ZONE\_NORMAL，ZONE\_DMA 才是真正的物理内存区域，ZONE\_MOVABLE 虚拟内存区域中的物理内存来自于上述三个物理内存区域。

在 32 位系统中 ZONE\_MOVABLE 虚拟内存区域中的物理内存页来自于 ZONE\_HIGHMEM。

在64 位系统中 ZONE\_MOVABLE 虚拟内存区域中的物理内存页来自于 ZONE\_NORMAL 或者 ZONE\_DMA 区域。

下面是不同的 gfp\_t 掩码设置方式与其对应的内存区域降级策略汇总列表：

gfp\_t 掩码

内存区域降级策略

什么都没有设置

ZONE\_NORMAL -> ZONE\_DMA

\_\_GFP\_DMA

ZONE\_DMA

\_\_GFP\_DMA & \_\_GFP\_HIGHMEM

ZONE\_DMA

\_\_GFP\_HIGHMEM

ZONE\_HIGHMEM -> ZONE\_NORMAL -> ZONE\_DMA

除了上述介绍 gfp\_t 掩码中的这四个物理内存区域修饰符之外，内核还定义了一些规范内存分配行为的修饰符，这些行为修饰符并不会限制内核从哪个物理内存区域中分配内存，而是会限制物理内存分配的行为，那么具体会限制哪些内存分配的行为呢？让我们接着往下看~~~

这些内存分配行为修饰符同样也是定义在 `/include/linux/gfp.h` 文件中：

    #define ___GFP_RECLAIMABLE	0x10u
    #define ___GFP_HIGH		0x20u
    #define ___GFP_IO		0x40u
    #define ___GFP_FS		0x80u
    #define ___GFP_ZERO		0x100u
    #define ___GFP_ATOMIC		0x200u
    #define ___GFP_DIRECT_RECLAIM	0x400u
    #define ___GFP_KSWAPD_RECLAIM	0x800u
    #define ___GFP_NOWARN		0x2000u
    #define ___GFP_RETRY_MAYFAIL	0x4000u
    #define ___GFP_NOFAIL		0x8000u
    #define ___GFP_NORETRY		0x10000u
    #define ___GFP_HARDWALL		0x100000u
    #define ___GFP_THISNODE		0x200000u
    #define ___GFP_MEMALLOC		0x20000u
    #define ___GFP_NOMEMALLOC	0x80000u


-   \_\_\_GFP\_RECLAIMABLE 用于指定分配的页面是可以回收的，\_\_\_GFP\_MOVABLE 则是用于指定分配的页面是可以移动的，这两个标志会影响底层的伙伴系统从哪个区域中去获取空闲内存页，这块内容我们会在后面讲解伙伴系统的时候详细介绍。
    
-   \_\_\_GFP\_HIGH 表示该内存分配请求是高优先级的，内核急切的需要内存，如果内存分配失败则会给系统带来非常严重的后果，设置该标志通常内存是不允许分配失败的，如果空闲内存不足，则会从紧急预留内存中分配。
    

> 关于物理内存区域中的紧急预留内存相关内容，笔者在之前文章 [《深入理解 Linux 物理内存管理》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486879&idx=1&sn=0bcc59a306d59e5199a11d1ca5313743&chksm=ce77cbd8f90042ce06f5086b1c976d1d2daa57bc5b768bac15f10ee3dc85874bbeddcd649d88&token=1274956236&lang=zh_CN#rd)一文中的 “ 5.1 物理内存区域中的预留内存 ” 小节中已经详细介绍过了。

-   \_\_\_GFP\_IO 表示内核在分配物理内存的时候可以发起磁盘 IO 操作。什么意思呢？比如当内核在进行内存分配的时候，发现物理内存不足，这时需要将不经常使用的内存页置换到 SWAP 分区或者 SWAP 文件中，这时就涉及到了 IO 操作，如果设置了该标志，表示允许内核将不常用的内存页置换出去。
    
-   \_\_\_GFP\_FS 允许内核执行底层文件系统操作，在与 VFS 虚拟文件系统层相关联的内核子系统中必须禁用该标志，否则可能会引起文件系统操作的循环递归调用，因为在设置 \_\_\_GFP\_FS 标志分配内存的情况下，可能会引起更多的文件系统操作，而这些文件系统的操作可能又会进一步产生内存分配行为，这样一直递归持续下去。
    
-   \_\_\_GFP\_ZERO 在内核分配内存成功之后，将内存页初始化填充字节 0 。
    
-   \_\_\_GFP\_ATOMIC 该标志的设置表示内存在分配物理内存的时候不允许睡眠必须是原子性地进行内存分配。比如在中断处理程序中，就不能睡眠，因为中断程序不能被重新调度。同时也不能在持有自旋锁的进程上下文中睡眠，因为可能导致死锁。**综上所述这个标志只能用在不能被重新安全调度的进程上下文中**。
    
-   \_\_\_GFP\_DIRECT\_RECLAIM 表示内核在进行内存分配的时候，可以进行直接内存回收。当剩余内存容量低于水位线 \_watermark\[WMARK\_MIN\] 时，说明此时的内存容量已经非常危险了，如果进程在这时请求内存分配，内核就会进行直接内存回收，直到内存水位线恢复到 \_watermark\[WMARK\_HIGH\] 之上。
    

![image](image/131791da32a45a4b20f0657793cb7d64.png)

-   \_\_\_GFP\_KSWAPD\_RECLAIM 表示内核在分配内存的时候，如果剩余内存容量在 \_watermark\[WMARK\_MIN\] 与 \_watermark\[WMARK\_LOW\] 之间时，内核就会唤醒 kswapd 进程开始异步内存回收，直到剩余内存高于 \_watermark\[WMARK\_HIGH\] 为止。
    
-   \_\_\_GFP\_NOWARN 表示当内核分配内存失败时，抑制内核的分配失败错误报告。
    
-   \_\_\_GFP\_RETRY\_MAYFAIL 在内核分配内存失败的时候，允许重试，但重试仍然可能失败，重试若干次后停止。与其对应的是 \_\_\_GFP\_NORETRY 标志表示分配内存失败时不允许重试。
    
-   \_\_\_GFP\_NOFAIL 在内核分配失败时一直重试直到成功为止。
    
-   \_\_\_GFP\_HARDWALL 该标志限制了内核分配内存的行为只能在当前进程分配到的 CPU 所关联的 NUMA 节点上进行分配，当进程可以运行的 CPU 受限时，该标志才会有意义，如果进程允许在所有 CPU 上运行则该标志没有意义。
    
-   \_\_\_GFP\_THISNODE 该标志限制了内核分配内存的行为只能在当前 NUMA 节点或者在指定 NUMA 节点中分配内存，如果内存分配失败不允许从其他备用 NUMA 节点中分配内存。
    
-   \_\_\_GFP\_MEMALLOC 允许内核在分配内存时可以从所有内存区域中获取内存，包括从紧急预留内存中获取。但使用该标示时需要保证进程在获得内存之后会很快的释放掉内存不会过长时间的占用，尤其要警惕避免过多的消耗紧急预留内存区域中的内存。
    
-   \_\_\_GFP\_NOMEMALLOC 标志用于明确禁止内核从紧急预留内存中获取内存。\_\_\_GFP\_NOMEMALLOC 标识的优先级要高于 \_\_\_GFP\_MEMALLOC
    

好了到现在为止，我们已经知道了 gfp\_t 掩码中包含的内存区域修饰符以及内存分配行为修饰符，是不是感觉头有点大了，事实上确实很让人头大，因为内核在不同场景下会使用不同的组合，这么多的修饰符总是以组合的形式出现，如果我们每次使用的时候都需要单独指定，那就会非常繁杂也很容易出错。

于是内核将各种标准情形下用到的 gfp\_t 掩码组合，提前为大家定义了一些标准的分组，方便大家直接使用。

    #define GFP_ATOMIC	(__GFP_HIGH|__GFP_ATOMIC|__GFP_KSWAPD_RECLAIM)
    #define GFP_KERNEL	(__GFP_RECLAIM | __GFP_IO | __GFP_FS)
    #define GFP_NOWAIT	(__GFP_KSWAPD_RECLAIM)
    #define GFP_NOIO	(__GFP_RECLAIM)
    #define GFP_NOFS	(__GFP_RECLAIM | __GFP_IO)
    #define GFP_USER	(__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
    #define GFP_DMA		__GFP_DMA
    #define GFP_DMA32	__GFP_DMA32
    #define GFP_HIGHUSER	(GFP_USER | __GFP_HIGHMEM)


-   GFP\_ATOMIC 是掩码 \_\_GFP\_HIGH，\_\_GFP\_ATOMIC，\_\_GFP\_KSWAPD\_RECLAIM 的组合，表示内存分配行为必须是原子的，是高优先级的。在任何情况下都不允许睡眠，如果空闲内存不够，则会从紧急预留内存中分配。该标志适用于中断程序，以及持有自旋锁的进程上下文中。
    
-   GFP\_KERNEL 是内核中最常用的标志，该标志设置之后内核的分配内存行为可能会阻塞睡眠，可以允许内核置换出一些不活跃的内存页到磁盘中。适用于可以重新安全调度的进程上下文中。
    
-   GFP\_NOIO 和 GFP\_NOFS 分别禁止内核在分配内存时进行磁盘 IO 和 文件系统 IO 操作。
    
-   GFP\_USER 用于映射到用户空间的内存分配，通常这些内存可以被内核或者硬件直接访问，比如硬件设备会将 Buffer 直接映射到用户空间中
    
-   GFP\_DMA 和 GFP\_DMA32 表示需要从 ZONE\_DMA 和 ZONE\_DMA32 内存区域中获取适用于 DMA 的内存页。
    
-   GFP\_HIGHUSER 用于给用户空间分配高端内存，因为在用户虚拟内存空间中，都是通过页表来访问非直接映射的高端内存区域，所以用户空间一般使用的是高端内存区域 ZONE\_HIGHMEM。
    

现在我们算是真正理解了，在本小节开始时，介绍的那几个内存分配接口函数中关于内存分配掩码 gfp\_mask 的所有内容，其中包括用于限制内核从哪个内存区域中分配内存，内核在分配内存过程中的行为，以及内核在各种标准分配场景下预先定义的掩码组合。

这时我们在回过头来看内核中关于物理内存分配的这些接口函数是不是感觉了如指掌了：

    struct page *alloc_pages(gfp_t gfp, unsigned int order)
    unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order)
    unsigned long get_zeroed_page(gfp_t gfp_mask)
    unsigned long __get_dma_pages(gfp_t gfp_mask, unsigned int order)


好了，现在我们已经清楚了这些内存分配接口的使用，那么这些接口又是如何实现的呢 ？让我们再一次深入到内核源码中去探索内核到底是如何分配物理内存的~~

### 3\. 物理内存分配内核源码实现

> 本文基于内核 5.19 版本讨论

在介绍 Linux 内核关于内存分配的源码实现之前，我们需要先找到内存分配的入口函数在哪里，在上小节中为大家介绍的众多内存分配接口的依赖层级关系如下图所示：

![image](image/acd5c6814aef8a9d360b6f2e81a93b20.png)

我们看到内存分配的任务最终会落在 alloc\_pages 这个接口函数中，在 alloc\_pages 中会调用 alloc\_pages\_node 进而调用 \_\_alloc\_pages\_node 函数，最终通过 \_\_alloc\_pages 函数正式进入内核内存分配的世界~~

> \_\_alloc\_pages 函数为 Linux 内核内存分配的核心入口函数

    static inline struct page *alloc_pages(gfp_t gfp_mask, unsigned int order)
    {
    	return alloc_pages_node(numa_node_id(), gfp_mask, order);
    }


    static inline struct page *
    __alloc_pages_node(int nid, gfp_t gfp_mask, unsigned int order)
    {
        // 校验指定的 NUMA 节点 ID 是否合法，不要越界
        VM_BUG_ON(nid < 0 || nid >= MAX_NUMNODES);
        // 指定节点必须是有效在线的
        VM_WARN_ON((gfp_mask & __GFP_THISNODE) && !node_online(nid));
    
        return __alloc_pages(gfp_mask, order, nid, NULL);
    }


\_\_alloc\_pages\_node 函数参数中的 nid 就是我们在上篇文章 [《深入理解 Linux 物理内存管理》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486879&idx=1&sn=0bcc59a306d59e5199a11d1ca5313743&chksm=ce77cbd8f90042ce06f5086b1c976d1d2daa57bc5b768bac15f10ee3dc85874bbeddcd649d88&token=1274956236&lang=zh_CN#rd)的 “ 4.1 内核如何统一组织 NUMA 节点 ” 小节介绍的 NUMA 节点 id。

![image](image/bf75561e72ae2d0afda353fe36559fba.png)

内核使用了一个大小为 MAX\_NUMNODES 的全局数组 node\_data\[\] 来管理所有的 NUMA 节点，数组的下标即为 NUMA 节点 Id 。

    #ifdef CONFIG_NUMA
    extern struct pglist_data *node_data[];
    #define NODE_DATA(nid)		(node_data[(nid)])


这里指定 nid 是为了告诉内核应该在哪个 NUMA 节点上分配内存，我们看到在  
alloc\_pages 函数中通过 numa\_node\_id() 获取运行当前进程的 CPU 所在的 NUMA 节点。并通过 `!node_online(nid)` 确保指定的 NUMA 节点是有效在线的。

> 关于 NUMA 节点的状态信息，大家可回看上篇文章的 《4.5 NUMA 节点的状态 node\_states》小节。

![image](image/c2403fdad16aa3890ba643d6be3b2b22.png)

#### 3.1 内存分配行为标识掩码 ALLOC\_\*

在我们进入 \_\_alloc\_pages 函数之前，笔者先来为大家介绍几个影响内核分配内存行为的标识，这些重要标识定义在内核文件 `/mm/internal.h` 中：

    #define ALLOC_WMARK_MIN     WMARK_MIN
    #define ALLOC_WMARK_LOW     WMARK_LOW
    #define ALLOC_WMARK_HIGH    WMARK_HIGH
    #define ALLOC_NO_WATERMARKS 0x04 /* don't check watermarks at all */
    
    #define ALLOC_HARDER         0x10 /* try to alloc harder */
    #define ALLOC_HIGH       0x20 /* __GFP_HIGH set */
    #define ALLOC_CPUSET         0x40 /* check for correct cpuset */
    
    #define ALLOC_KSWAPD        0x800 /* allow waking of kswapd, __GFP_KSWAPD_RECLAIM set */


我们先来看前四个标识内存水位线的常量含义，这四个内存水位线标识表示内核在分配内存时必须考虑内存的水位线，在不同的水位线下内存的分配行为也会有所不同。

> 笔者在上篇文章 [《深入理解 Linux 物理内存管理》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486879&idx=1&sn=0bcc59a306d59e5199a11d1ca5313743&chksm=ce77cbd8f90042ce06f5086b1c976d1d2daa57bc5b768bac15f10ee3dc85874bbeddcd649d88&token=1274956236&lang=zh_CN#rd)的 “ 5.2 物理内存区域中的水位线 ” 小节中曾详细地介绍了各个水位线的含义以及在不同水位线下内存分配的不同表现。

上篇文章中我们提到，内核会为 NUMA 节点中的每个物理内存区域 zone 定制三条用于指示内存容量的水位线，它们分别是：WMARK\_MIN（页最小阈值）， WMARK\_LOW （页低阈值），WMARK\_HIGH（页高阈值）。

这三个水位线定义在 `/include/linux/mmzone.h` 文件中：

    enum zone_watermarks {
    	WMARK_MIN,
    	WMARK_LOW,
    	WMARK_HIGH,
    	NR_WMARK
    };


三条水位线对应的 watermark 具体数值存储在每个物理内存区域 struct zone 结构中的 \_watermark\[NR\_WMARK\] 数组中。

    struct zone {
        // 物理内存区域中的水位线
        unsigned long _watermark[NR_WMARK];
    }


物理内存区域中不同水位线的含义以及内存分配在不同水位线下的行为如下图所示：

![image](image/6d13708ae065ecc953f4ab7abb8f1e74.png)

-   当该物理内存区域的剩余内存容量高于 \_watermark\[WMARK\_HIGH\] 时，说明此时该物理内存区域中的内存容量非常充足，内存分配完全没有压力。
    
-   当剩余内存容量在 \_watermark\[WMARK\_LOW\] 与\_watermark\[WMARK\_HIGH\] 之间时，说明此时内存有一定的消耗但是还可以接受，能够继续满足进程的内存分配需求。
    
-   当剩余内存容量在 \_watermark\[WMARK\_MIN\] 与 \_watermark\[WMARK\_LOW\] 之间时，说明此时内存容量已经有点危险了，内存分配面临一定的压力，但是还可以满足进程此时的内存分配要求，当给进程分配完内存之后，就会唤醒 kswapd 进程开始内存回收，直到剩余内存高于 \_watermark\[WMARK\_HIGH\] 为止。
    

> 在这种情况下，进程的内存分配会触发内存回收，但请求进程本身不会被阻塞，由内核的 kswapd 进程异步回收内存。

-   ==当剩余内存容量低于 \_watermark\[WMARK\_MIN\] 时，说明此时的内存容量已经非常危险了，如果进程在这时请求内存分配，内核就会进行**直接内存回收**，这时内存回收的任务将会由请求进程同步完成。==

> 注意：上面提到的物理内存区域 zone 的剩余内存是需要刨去 lowmem\_reserve 预留内存大小（用于紧急内存分配）。也就是说 zone 里被伙伴系统所管理的内存并不包含 lowmem\_reserve 预留内存。

好了，在我们重新回顾了内存分配行为在这三条水位线：\_watermark\[WMARK\_HIGH\]，\_watermark\[WMARK\_LOW\]，_watermark\[WMARK\_MIN\] 下的不同表现之后，我们在回过来看本小节开始处提到的那几个 ALLOC_\* 内存分配标识。

==ALLOC\_NO\_WATERMARKS 表示在内存分配过程中完全不会考虑上述三个水位线的影响。==

==ALLOC\_WMARK\_HIGH 表示在内存分配的时候，当前物理内存区域 zone 中剩余内存页的数量至少要达到 \_watermark\[WMARK\_HIGH\] 水位线，才能进行内存的分配。==

==ALLOC\_WMARK\_LOW 和 ALLOC\_WMARK\_MIN 要表达的内存分配语义也是一样，当前物理内存区域 zone 中剩余内存页的数量至少要达到水位线 \_watermark\[WMARK\_LOW\] 或者 \_watermark\[WMARK\_MIN\]，才能进行内存的分配。==

==ALLOC\_HARDER 表示在内存分配的时候，会放宽内存分配规则的限制，所谓的放宽规则就是降低 \_watermark\[WMARK\_MIN\] 水位线，努力使内存分配最大可能成功。==

当我们在 gfp\_t 掩码中设置了 \_\_\_GFP\_HIGH 时，ALLOC\_HIGH 标识才起作用，该标识表示当前内存分配请求是高优先级的，内核急切的需要内存，如果内存分配失败则会给系统带来非常严重的后果，设置该标志通常内存是不允许分配失败的，如果空闲内存不足，则会从紧急预留内存中分配。

ALLOC\_CPUSET 表示内存只能在当前进程所允许运行的 CPU 所关联的 NUMA 节点中进行分配。比如使用 cgroup 限制进程只能在某些特定的 CPU 上运行，那么进程所发起的内存分配请求，只能在这些特定 CPU 所在的 NUMA 节点中进行。

==ALLOC\_KSWAPD 表示允许唤醒 NUMA 节点中的 KSWAPD 进程，异步进行内存回收。==

内核会为每个 NUMA 节点分配一个 kswapd 进程用于回收不经常使用的页面。

    typedef struct pglist_data {
            .........
        // 页面回收进程
        struct task_struct *kswapd;
            ..........
    } pg_data_t;


#### 3.2 内存分配的心脏 \_\_alloc\_pages

好了，在为大家介绍完这些影响内存分配行为的相关标识掩码：`GFP_*`，`ALLOC_*` 之后，下面就该来介绍本文的主题——物理内存分配的核心函数 \_\_alloc\_pages ，从下面内核源码的注释中我们可以看出，这个函数正是伙伴系统的核心心脏，它是内核内存分配的核心入口函数，整个内存分配的完整过程全部封装在这里。

> 该函数的逻辑比较复杂，因为在内存分配过程中需要涉及处理各种 `GFP_*`，`ALLOC_*` 标识，然后根据上述各种标识的含义来决定内存分配该如何进行。所以大家需要多点耐心，一步一步跟着笔者的思路往下走~~~

```C
/*
 * This is the 'heart' of the zoned buddy allocator.
 */
struct page *__alloc_pages(gfp_t gfp, unsigned int order, int preferred_nid,
                            nodemask_t *nodemask)
{
    // 用于指向分配成功的内存
    struct page *page;
    // 内存区域中的剩余内存需要在 WMARK_LOW 水位线之上才能进行内存分配，否则失败（初次尝试快速内存分配）
    unsigned int alloc_flags = ALLOC_WMARK_LOW;
    // 之前小节中介绍的内存分配掩码集合
    gfp_t alloc_gfp; 
    // 用于在不同内存分配辅助函数中传递参数
    struct alloc_context ac = { };

    // 检查用于向伙伴系统申请内存容量的分配阶 order 的合法性
    // 内核定义最大分配阶 MAX_ORDER -1 = 10，也就是说一次最多只能从伙伴系统中申请 1024 个内存页。
    if (WARN_ON_ONCE_GFP(order >= MAX_ORDER, gfp))
        return NULL;
    // 表示在内存分配期间进程可以休眠阻塞
    gfp &= gfp_allowed_mask;

    alloc_gfp = gfp;
    // 初始化 alloc_context，并为接下来的快速内存分配设置相关 gfp
    if (!prepare_alloc_pages(gfp, order, preferred_nid, nodemask, &ac,
            &alloc_gfp, &alloc_flags))
        // 提前判断本次内存分配是否能够成功，如果不能则尽早失败
        return NULL;

    // 避免内存碎片化的相关分配标识设置，可暂时忽略
    alloc_flags |= alloc_flags_nofragment(ac.preferred_zoneref->zone, gfp);

    // 内存分配快速路径：第一次尝试从底层伙伴系统分配内存，注意此时是在 WMARK_LOW 水位线之上分配内存
    page = get_page_from_freelist(alloc_gfp, order, alloc_flags, &ac);
    if (likely(page))
        // 如果内存分配成功则直接返回
        goto out;
    // 流程走到这里表示内存分配在快速路径下失败
    // 这里需要恢复最初的内存分配标识设置，后续会尝试更加激进的内存分配策略
    alloc_gfp = gfp;
    // 恢复最初的 node mask 因为它可能在第一次内存分配的过程中被改变
    // 本函数中 nodemask 起初被设置为 null
    ac.nodemask = nodemask;

    // 在第一次快速内存分配失败之后，说明内存已经不足了，内核需要做更多的工作
    // 比如通过 kswap 回收内存，或者直接内存回收等方式获取更多的空闲内存以满足内存分配的需求
    // 所以下面的过程称之为慢速分配路径
    page = __alloc_pages_slowpath(alloc_gfp, order, &ac);

out:
    // 内存分配成功，直接返回 page。否则返回 NULL
    return page;
}
```


\_\_alloc\_pages 函数中的内存分配整体逻辑如下：

-   首先内核会尝试在内存水位线 WMARK\_LOW 之上快速的进行一次内存分配。这一点我们从开始的 `unsigned int alloc_flags = ALLOC_WMARK_LOW` 语句中可以看得出来。

![image](image/762c019c3ea053708dbe119107bc5ea7.png)

-   校验本次内存分配指定伙伴系统的分配阶 order 的有效性，伙伴系统在内核中的最大分配阶定义在 `/include/linux/mmzone.h` 文件中，最大分配阶 MAX\_ORDER -1 = 10，也就是说一次最多只能从伙伴系统中申请 1024 个内存页，对应 4M 大小的连续物理内存。

    /* Free memory management - zoned buddy allocator.  */
    #ifndef CONFIG_FORCE_MAX_ZONEORDER
    #define MAX_ORDER 11
    
-   调用 prepare\_alloc\_pages 初始化 alloc\_context ，用于在不同内存分配辅助函数中传递内存分配参数。为接下来即将进行的快速内存分配做准备。

    struct alloc_context {
        // 运行进程 CPU 所在 NUMA  节点以及其所有备用 NUMA 节点中允许内存分配的内存区域
        struct zonelist *zonelist;
        // NUMA  节点状态掩码
        nodemask_t *nodemask;
        // 内存分配优先级最高的内存区域 zone
        struct zoneref *preferred_zoneref;
        // 物理内存页的迁移类型分为：不可迁移，可回收，可迁移类型，防止内存碎片
        int migratetype;
    
        // 内存分配最高优先级的内存区域 zone
        enum zone_type highest_zoneidx;
        // 是否允许当前 NUMA 节点中的脏页均衡扩散迁移至其他 NUMA 节点
        bool spread_dirty_pages;
    };
    
-   调用 get\_page\_from\_freelist 方法首次尝试在伙伴系统中进行内存分配，这次内存分配比较快速，只是快速的扫描一下各个内存区域中是否有足够的空闲内存能够满足本次内存分配，如果有则立马从伙伴系统中申请，如果没有立即返回， page 设置为 null，进行后续慢速内存分配处理。

> 这里需要注意的是：首次尝试的快速内存分配是在 WMARK\_LOW 水位线之上进行的。

-   当快速内存分配失败之后，情况就会变得非常复杂，内核将不得不做更多的工作，比如开启 kswapd 进程异步内存回收，更极端的情况则需要进行直接内存回收，或者直接内存整理以获取更多的空闲连续内存。这一切的复杂逻辑全部封装在 \_\_alloc\_pages\_slowpath 函数中。

> __alloc\_pages\_slowpath 函数复杂在于需要结合前边小节中介绍的 GFP_\*，ALLOC_\* 这些内存分配标识，根据不同的标识进入不同的内存分配逻辑分支，涉及到的情况比较繁杂。这里大家只需要简单了解，后面笔者会详细介绍~~~

以上介绍的 \_\_alloc\_pages 函数内存分配逻辑以及与对应的内存水位线之间的关系如下图所示：

![image](image/8d22f8b4688ed03e5321fa86c7e737a5.png)

总体流程介绍完之后，我们接着来看一下以上内存分配过程涉及到的三个重要内存分配辅助函数：prepare\_alloc\_pages，\_\_alloc\_pages\_slowpath，get\_page\_from\_freelist 。

#### 3.3 prepare\_alloc\_pages

prepare\_alloc\_pages 初始化 alloc\_context ，用于在不同内存分配辅助函数中传递内存分配参数，为接下来即将进行的快速内存分配做准备。

```C
static inline bool prepare_alloc_pages(gfp_t gfp_mask, unsigned int order,
        int preferred_nid, nodemask_t *nodemask,
        struct alloc_context *ac, gfp_t *alloc_gfp,
        unsigned int *alloc_flags)
{
    // 根据 gfp_mask 掩码中的内存区域修饰符获取内存分配最高优先级的内存区域 zone
    ac->highest_zoneidx = gfp_zone(gfp_mask);
    // 从 NUMA 节点的备用节点链表中一次性获取允许进行内存分配的所有内存区域
    ac->zonelist = node_zonelist(preferred_nid, gfp_mask);
    ac->nodemask = nodemask;
    // 从 gfp_mask 掩码中获取页面迁移属性，迁移属性分为：不可迁移，可回收，可迁移。这里只需要简单知道，后面在相关章节会细讲
    ac->migratetype = gfp_migratetype(gfp_mask);

   // 如果使用 cgroup 将进程绑定限制在了某些 CPU 上，那么内存分配只能在
   // 这些绑定的 CPU 相关联的 NUMA 节点中进行
    if (cpusets_enabled()) {
        *alloc_gfp |= __GFP_HARDWALL;
        if (in_task() && !ac->nodemask)
            ac->nodemask = &cpuset_current_mems_allowed;
        else
            *alloc_flags |= ALLOC_CPUSET;
    }
      
    // 如果设置了允许直接内存回收，那么内存分配进程则可能会导致休眠被重新调度 
    might_sleep_if(gfp_mask & __GFP_DIRECT_RECLAIM);
    // 提前判断本次内存分配是否能够成功，如果不能则尽早失败
    if (should_fail_alloc_page(gfp_mask, order))
        return false;
    // 获取最高优先级的内存区域 zone
    // 后续内存分配则首先会在该内存区域中进行分配
    ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
                    ac->highest_zoneidx, ac->nodemask);

    return true;
}
```


prepare\_alloc\_pages 主要的任务就是在快速内存分配开始之前，做一些准备初始化的工作，其中最核心的就是从指定 NUMA 节点中，根据 gfp\_mask 掩码中的内存区域修饰符获取可以进行内存分配的所有内存区域 zone （包括其他备用 NUMA 节点中包含的内存区域）。

之前笔者已经在 [《深入理解 Linux 物理内存管理》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486879&idx=1&sn=0bcc59a306d59e5199a11d1ca5313743&chksm=ce77cbd8f90042ce06f5086b1c976d1d2daa57bc5b768bac15f10ee3dc85874bbeddcd649d88&token=1274956236&lang=zh_CN#rd)一文中的 “ 4.3 NUMA 节点物理内存区域的划分 ” 小节为大家已经详细介绍了 NUMA 节点的数据结构 struct pglist\_data。

struct pglist\_data 结构中不仅包含了本 NUMA 节点中的所有内存区域，还包括了其他备用 NUMA 节点中的物理内存区域，当本节点中内存不足的情况下，内核会从备用 NUMA 节点中的内存区域进行跨节点内存分配。

```C
typedef struct pglist_data {
    // NUMA 节点中的物理内存区域个数
    int nr_zones; 
    // NUMA 节点中的物理内存区域
    struct zone node_zones[MAX_NR_ZONES];
    // NUMA 节点的备用列表，其中包含了所有 NUMA 节点中的所有物理内存区域 zone，按照访问距离由近到远顺序依次排列
    struct zonelist node_zonelists[MAX_ZONELISTS];
} pg_data_t;
```


我们可以根据 nid 和 gfp\_mask 掩码中的物理内存区域描述符利用 node\_zonelist 函数一次性获取允许进行内存分配的所有内存区域（所有 NUMA 节点）。

```C
static inline struct zonelist *node_zonelist(int nid, gfp_t flags)
{
	return NODE_DATA(nid)->node_zonelists + gfp_zonelist(flags);
}
```


### 4\. 内存慢速分配入口 alloc\_pages\_slowpath

正如前边小节我们提到的那样，alloc\_pages\_slowpath 函数非常的复杂，其中包含了内存分配的各种异常情况的处理，并且会根据前边介绍的 GFP\__，ALLOC\__ 等各种内存分配策略掩码进行不同分支的处理，这样就变得非常的庞大而繁杂。

alloc\_pages\_slowpath 函数包含了整个内存分配的核心流程，本身非常的繁杂庞大，为了能够给大家清晰的梳理清楚这些复杂的内存分配流程，所以笔者决定还是以 `总 - 分 - 总` 的结构来给大家呈现。

下面这段伪代码是笔者提取出来的 alloc\_pages\_slowpath 函数的主干框架，其中包含的一些核心分支以及核心步骤笔者都通过注释的形式为大家标注出来了，这里我先从总体上大概浏览下 alloc\_pages\_slowpath 主要分为哪几个逻辑处理模块，它们分别处理了哪些事情。

> 还是那句话，这里大家只需要总体把握，不需要掌握每个细节，关于细节的部分，笔者后面会带大家逐个击破！！！

```C
static inline struct page *
__alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
                        struct alloc_context *ac)
{
        ......... 初始化慢速内存分配路径下的相关参数 .......

retry_cpuset:

        ......... 调整内存分配策略 alloc_flags 采用更加激进方式获取内存 ......
        ......... 此时内存分配主要是在进程所允许运行的 CPU 相关联的 NUMA 节点上 ......
        ......... 内存水位线下调至 WMARK_MIN ...........
        ......... 唤醒所有 kswapd 进程进行异步内存回收  ...........
        ......... 触发直接内存整理 direct_compact 来获取更多的连续空闲内存 ......

retry:

        ......... 进一步调整内存分配策略 alloc_flags 使用更加激进的非常手段进行内存分配 ...........
        ......... 在内存分配时忽略内存水位线 ...........
        ......... 触发直接内存回收 direct_reclaim ...........
        ......... 再次触发直接内存整理 direct_compact ...........
        ......... 最后的杀手锏触发 OOM 机制  ...........

nopage:
        ......... 经过以上激进的内存分配手段仍然无法满足内存分配就会来到这里 ......
        ......... 如果设置了 __GFP_NOFAIL 不允许内存分配失败，则不停重试上述内存分配过程 ......

fail:
        ......... 内存分配失败，输出告警信息 ........

      warn_alloc(gfp_mask, ac->nodemask,
            "page allocation failure: order:%u", order);
got_pg:
        ......... 内存分配成功，返回新申请的内存块 ........

      return page;
}
```


#### 4.1 初始化内存分配慢速路径下的相关参数

```C
static inline struct page *
__alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
                        struct alloc_context *ac)
{
    // 在慢速内存分配路径中可能会导致内核进行直接内存回收
    // 这里设置 __GFP_DIRECT_RECLAIM 表示允许内核进行直接内存回收
    bool can_direct_reclaim = gfp_mask & __GFP_DIRECT_RECLAIM;
    // 本次内存分配是否是针对大量内存页的分配，内核定义 PAGE_ALLOC_COSTLY_ORDER = 3
    // 也就是说内存请求内存页的数量大于 2 ^ 3 = 8 个内存页时，costly_order = true，后续会影响是否进行 OOM
    const bool costly_order = order > PAGE_ALLOC_COSTLY_ORDER;
    // 用于指向成功申请的内存
    struct page *page = NULL;
    // 内存分配标识，后续会根据不同标识进入到不同的内存分配逻辑处理分支
    unsigned int alloc_flags;
    // 后续用于记录直接内存回收了多少内存页
    unsigned long did_some_progress;
    // 关于内存整理相关参数
    enum compact_priority compact_priority;
    enum compact_result compact_result;
    int compaction_retries;
    // 记录重试的次数，超过一定的次数（16次）则内存分配失败
    int no_progress_loops;
    // 临时保存调整后的内存分配策略
    int reserve_flags;

    // 流程现在来到了慢速内存分配这里，说明快速分配路径已经失败了
    // 内核需要对 gfp_mask 分配行为掩码做一些修改，修改为一些更可能导致内存分配成功的标识
    // 因为接下来的直接内存回收非常耗时可能会导致进程阻塞睡眠，不适用原子 __GFP_ATOMIC 内存分配的上下文。
    if (WARN_ON_ONCE((gfp_mask & (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)) ==
                (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)))
        gfp_mask &= ~__GFP_ATOMIC;

retry_cpuset:

retry:

nopage:

fail:

got_pg:

}
```


在内核进入慢速内存分配路径之前，首先会在这里初始化后续内存分配需要的参数，由于笔者已经在各个字段上标注了丰富的注释，所以这里笔者只对那些难以理解的核心参数为大家进行相关细节的铺垫，这里大家对这些参数有个大概印象即可，后续在使用到的时候，笔者还会再次提起~~~

首先我们看 costly\_order 参数，order 表示底层伙伴系统的分配阶，内核只能向伙伴系统申请 2 的 order 次幂个内存页，costly 从字面意思上来说表示有一定代价和消耗的，costly\_order 连起来就表示在内核中 order 分配阶达到多少，在内核看来就是代价比较大的内存分配行为。

这个临界值就是 PAGE\_ALLOC\_COSTLY\_ORDER 定义在 `/include/linux/mmzone.h` 文件中：

```C
#define PAGE_ALLOC_COSTLY_ORDER 3
```


也就是说在内核看来，当请求内存页的数量大于 2 ^ 3 = 8 个内存页时，costly\_order = true，内核就认为本次内存分配是一次成本比较大的行为。后续会根据这个参数 costly\_order 来决定是否触发 OOM 。

```C
    const bool costly_order = order > PAGE_ALLOC_COSTLY_ORDER;
```


当内存严重不足的时候，内核会开启直接内存回收 direct\_reclaim ，参数 did\_some\_progress 表示经过一次直接内存回收之后，内核回收了多少个内存页。这个参数后续会影响是否需要进行内存分配重试。

no\_progress\_loops 用于记录内存分配重试的次数，如果内存分配重试的次数超过最大限制 MAX\_RECLAIM\_RETRIES，则停止重试，开启 OOM。

MAX\_RECLAIM\_RETRIES 定义在 `/mm/internal.h` 文件中：

```C
#define MAX_RECLAIM_RETRIES 16
```


compact\_\* 相关的参数用于直接内存整理 direct\_compact，内核通常会在直接内存回收 direct\_reclaim 之前进行一次 direct\_compact，如果经过 direct\_compact 整理之后有了足够多的空间内存就不需要进行 direct\_reclaim 了。

那么这个 direct\_compact 到底是干什么的呢？它在慢速内存分配过程起了什么作用？

随着系统的长时间运行通常会伴随着不同大小的物理内存页的分配和释放，这种不规则的分配释放，随着系统的长时间运行就会导致内存碎片，内存碎片会使得系统在明明有足够内存的情况下，依然无法为进程分配合适的内存。

![image](image/e3bdec0ced0f60162f4974867f39f39b.png)

如上图所示，假如现在系统一共有 16 个物理内存页，当前系统只是分配了 3 个物理页，那么在当前系统中还剩余 13 个物理内存页的情况下，如果内核想要分配 8 个连续的物理页由于内存碎片的存在则会分配失败。（只能分配最多 4 个连续的物理页）

> 内核中请求分配的物理页面数只能是 2 的次幂！！

为了解决内存碎片化的问题，内核将内存页面分为了：可移动的，可回收的，不可移动的三种类型。

可移动的页面聚集在一起，可回收的的页面聚集在一起，不可移动的的页面聚集也在一起。从而作为去碎片化的基础， 然后进行成块回收。

在回收时把可回收的一起回收，把可移动的一起移动，从而能空出大量连续物理页面。direct\_compact 会扫描内存区域 zone 里的页面，把已分配的页记录下来，然后把所有已分配的页移动到 zone 的一端，这样就会把一个已经充满碎片的 zone 整理成一段完全未分配的区间和一段已经分配的区间，从而腾出大块连续的物理页面供内核分配。

![image](image/b404ec46d13e7a23282ef044c7f3014c.png)

#### 4.2 retry\_cpuset

在介绍完了内存分配在慢速路径下所需要的相关参数之后，下面就正式来到了 alloc\_pages\_slowpath 的内存分配逻辑：

```C
static inline struct page *
__alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
                        struct alloc_context *ac)
{
        ......... 初始化慢速内存分配路径下的相关参数 .......

retry_cpuset:

    // 在之前的快速内存分配路径下设置的相关分配策略比较保守，不是很激进，用于在 WMARK_LOW 水位线之上进行快速内存分配
    // 走到这里表示快速内存分配失败，此时空闲内存严重不足了
    // 所以在慢速内存分配路径下需要重新设置更加激进的内存分配策略，采用更大的代价来分配内存
    alloc_flags = gfp_to_alloc_flags(gfp_mask);

    // 重新按照新的设置按照内存区域优先级计算 zonelist 的迭代起点（最高优先级的 zone）
    // fast path 和 slow path 的设置不同所以这里需要重新计算
    ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
                    ac->highest_zoneidx, ac->nodemask);
    // 如果没有合适的内存分配区域，则跳转到 nopage , 内存分配失败
    if (!ac->preferred_zoneref->zone)
        goto nopage;
    // 唤醒所有的 kswapd 进程异步回收内存
    if (alloc_flags & ALLOC_KSWAPD)
        wake_all_kswapds(order, gfp_mask, ac);

    // 此时所有的 kswapd 进程已经被唤醒，正在异步进行内存回收
    // 之前我们已经在 gfp_to_alloc_flags 方法中重新调整了 alloc_flags
    // 换成了一套更加激进的内存分配策略，注意此时是在 WMARK_MIN 水位线之上进行内存分配
    // 调整后的 alloc_flags 很可能会立即成功，因此这里先尝试一下
    page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
    if (page)
        // 内存分配成功，跳转到 got_pg 直接返回 page
        goto got_pg;

    // 对于分配大内存来说 costly_order = true (超过 8 个内存页)，需要首先进行内存整理，这样内核可以避免直接内存回收从而获取更多的连续空闲内存页
    // 对于需要分配不可移动的高阶内存的情况，也需要先进行内存整理，防止永久内存碎片
    if (can_direct_reclaim &&
            (costly_order ||
               (order > 0 && ac->migratetype != MIGRATE_MOVABLE))
            && !gfp_pfmemalloc_allowed(gfp_mask)) {
        // 进行直接内存整理，获取更多的连续空闲内存防止内存碎片
        page = __alloc_pages_direct_compact(gfp_mask, order,
                        alloc_flags, ac,
                        INIT_COMPACT_PRIORITY,
                        &compact_result);
        if (page)
            goto got_pg;

        if (costly_order && (gfp_mask & __GFP_NORETRY)) {
            // 流程走到这里表示经过内存整理之后依然没有足够的内存供分配
            // 但是设置了 NORETRY 标识不允许重试，那么就直接失败，跳转到 nopage
            if (compact_result == COMPACT_SKIPPED ||
                compact_result == COMPACT_DEFERRED)
                goto nopage;
            // 同步内存整理开销太大，后续开启异步内存整理
            compact_priority = INIT_COMPACT_PRIORITY;
        }
    }

retry:

nopage:

fail:

got_pg:
    return page;
}
```


流程走到这里，说明内核在 《3.2 内存分配的心脏 \_\_alloc\_pages》小节中介绍的快速路径下尝试的内存分配已经失败了，所以才会走到慢速分配路径这里来。

之前我们介绍到快速分配路径是在 WMARK\_LOW 水位线之上进行内存分配，与其相配套的内存分配策略比较保守，目的是快速的在各个内存区域 zone 之间搜索可供分配的空闲内存。

![image](image/91fc0eed3047cf195052a1dcacbfb980.png)

快速分配路径下的失败意味着此时系统中的空闲内存已经不足了，所以在慢速分配路径下内核需要改变内存分配策略，采用更加激进的方式来进行内存分配，首先会把内存分配水位线降低到 WMARK\_MIN 之上，然后将内存分配策略调整为更加容易促使内存分配成功的策略。

而内存分配策略相关的调整逻辑，内核定义在 gfp\_to\_alloc\_flags 函数中：

```C
static inline unsigned int gfp_to_alloc_flags(gfp_t gfp_mask)
{
    // 在慢速内存分配路径中，会进一步放宽对内存分配的限制，将内存分配水位线调低到 WMARK_MIN
    // 也就是说内存区域中的剩余内存需要在 WMARK_MIN 水位线之上才可以进行内存分配
    unsigned int alloc_flags = ALLOC_WMARK_MIN | ALLOC_CPUSET;
    
    // 如果内存分配请求无法运行直接内存回收，或者分配请求设置了 __GFP_HIGH 
    // 那么意味着内存分配会更多的使用紧急预留内存
    alloc_flags |= (__force int)
        (gfp_mask & (__GFP_HIGH | __GFP_KSWAPD_RECLAIM));

    if (gfp_mask & __GFP_ATOMIC) {
        //  ___GFP_NOMEMALLOC 标志用于明确禁止内核从紧急预留内存中获取内存。
        // ___GFP_NOMEMALLOC 标识的优先级要高于 ___GFP_MEMALLOC
        if (!(gfp_mask & __GFP_NOMEMALLOC))
           // 如果允许从紧急预留内存中分配，则需要进一步放宽内存分配限制
           // 后续根据 ALLOC_HARDER 标识会降低 WMARK_LOW 水位线
            alloc_flags |= ALLOC_HARDER;
        // 在这个分支中表示内存分配请求已经设置了  __GFP_ATOMIC （非常重要，不允许失败）
        // 这种情况下为了内存分配的成功，会去除掉 CPUSET 的限制，可以在所有 NUMA 节点上分配内存
        alloc_flags &= ~ALLOC_CPUSET;
    } else if (unlikely(rt_task(current)) && in_task())
         // 如果当前进程不是 real time task 或者不在 task 上下文中
         // 设置 HARDER 标识
        alloc_flags |= ALLOC_HARDER;

    return alloc_flags;
}
```


在调整好的新的内存分配策略 alloc\_flags 之后，就需要根据新的策略来重新获取可供分配的内存区域 zone。

```C
  ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
                    ac->highest_zoneidx, ac->nodemask);
```


从上图中我们可以看出，当剩余内存处于 WMARK\_MIN 与 WMARK\_LOW 之间时，内核会唤醒所有 kswapd 进程来异步回收内存，直到剩余内存重新回到水位线 WMARK\_HIGH 之上。

```C
    if (alloc_flags & ALLOC_KSWAPD)
        wake_all_kswapds(order, gfp_mask, ac);
```


到目前为止，内核已经在慢速分配路径下通过 gfp\_to\_alloc\_flags 调整为更加激进的内存分配策略，并将水位线降低到 WMARK\_MIN，同时也唤醒了 kswapd 进程来异步回收内存。

此时在新的内存分配策略下进行内存分配很可能会一次性成功，所以内核会首先尝试进行一次内存分配。

```C
page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
```


如果首次尝试分配内存失败之后，内核就需要进行直接内存整理 direct\_compact 来获取更多的可供分配的连续内存页。

如果经过 direct\_compact 之后依然没有足够的内存可供分配，那么就会进入 retry 分支采用更加激进的方式来分配内存。如果内存分配策略设置了 \_\_GFP\_NORETRY 表示不允许重试，那么就会直接失败，流程跳转到 nopage 分支进行处理。

![image](image/ba64da9c3ef8293e99e174f0d9d702c3.png)

#### 4.3 retry

内存分配流程来到 retry 分支这里说明情况已经变得非常危急了，在经过 retry\_cpuset 分支的处理，内核将内存水位线下调至 WMARK\_MIN，并开启了 kswapd 进程进行异步内存回收，触发直接内存整理 direct\_compact，在采取了这些措施之后，依然无法满足内存分配的需求。

所以在接下来的分配逻辑中，内核会近一步采取更加激进的非常手段来获取连续的空闲内存，下面我们来一起看下这部分激进的内容：

```C
static inline struct page *
__alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
                        struct alloc_context *ac)
{
        ......... 初始化慢速内存分配路径下的相关参数 .......

retry_cpuset:

        ......... 调整内存分配策略 alloc_flags 采用更加激进方式获取内存 ......
        ......... 此时内存分配主要是在进程所允许运行的 CPU 相关联的 NUMA 节点上 ......
        ......... 内存水位线下调至 WMARK_MIN ...........
        ......... 唤醒所有 kswapd 进程进行异步内存回收  ...........
        ......... 触发直接内存整理 direct_compact 来获取更多的连续空闲内存 ......

retry:
    // 确保所有 kswapd 进程不要意外进入睡眠状态
    if (alloc_flags & ALLOC_KSWAPD)
        wake_all_kswapds(order, gfp_mask, ac);

    // 流程走到这里，说明在 WMARK_MIN 水位线之上也分配内存失败了
    // 并且经过内存整理之后，内存分配仍然失败，说明当前内存容量已经严重不足
    // 接下来就需要使用更加激进的非常手段来尝试内存分配（忽略掉内存水位线），继续修改 alloc_flags 保存在 reserve_flags 中
    reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
    if (reserve_flags)
        alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, reserve_flags);

    // 如果内存分配可以任意跨节点分配（忽略内存分配策略），这里需要重置 nodemask 以及 zonelist。
    if (!(alloc_flags & ALLOC_CPUSET) || reserve_flags) {
        // 这里的内存分配是高优先级系统级别的内存分配，不是面向用户的
        ac->nodemask = NULL;
        ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
                    ac->highest_zoneidx, ac->nodemask);
    }

    // 这里使用重新调整的 zonelist 和 alloc_flags 在尝试进行一次内存分配
    // 注意此次的内存分配是忽略内存水位线的 ALLOC_NO_WATERMARKS
    page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
    if (page)
        goto got_pg;

    // 在忽略内存水位线的情况下仍然分配失败，现在内核就需要进行直接内存回收了
    if (!can_direct_reclaim)
        // 如果进程不允许进行直接内存回收，则只能分配失败
        goto nopage;

    // 开始直接内存回收
    page = __alloc_pages_direct_reclaim(gfp_mask, order, alloc_flags, ac,
                            &did_some_progress);
    if (page)
        goto got_pg;

    // 直接内存回收之后仍然无法满足分配需求，则再次进行直接内存整理
    page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags, ac,
                    compact_priority, &compact_result);
    if (page)
        goto got_pg;

    // 在内存直接回收和整理全部失败之后，如果不允许重试，则只能失败
    if (gfp_mask & __GFP_NORETRY)
        goto nopage;

    // 后续会触发 OOM 来释放更多的内存，这里需要判断本次内存分配是否需要分配大量的内存页（大于 8 ） costly_order = true
    // 如果是的话则内核认为即使执行 OOM 也未必会满足这么多的内存页分配需求.
    // 所以还是直接失败比较好，不再执行 OOM，除非设置 __GFP_RETRY_MAYFAIL
    if (costly_order && !(gfp_mask & __GFP_RETRY_MAYFAIL))
        goto nopage;

    // 流程走到这里说明我们已经尝试了所有措施内存依然分配失败了，此时内存已经非常危急了。
    // 走到这里说明进程允许内核进行重试流程，但在开始重试之前，内核需要判断是否应该进行重试,重试标准：
    // 1 如果内核已经重试了 MAX_RECLAIM_RETRIES (16) 次仍然失败，则放弃重试执行后续 OOM。
    // 2 如果内核将所有可选内存区域中的所有可回收页面全部回收之后，仍然无法满足内存的分配，那么放弃重试执行后续 OOM
    if (should_reclaim_retry(gfp_mask, order, ac, alloc_flags,
                 did_some_progress > 0, &no_progress_loops))
        goto retry;

    // 如果内核判断不应进行直接内存回收的重试，这里还需要判断下是否应该进行内存整理的重试。
    // did_some_progress 表示上次直接内存回收，具体回收了多少内存页
    // 如果 did_some_progress = 0 则没有必要在进行内存整理重试了，因为内存整理的实现依赖于足够的空闲内存量
    if (did_some_progress > 0 &&
            should_compact_retry(ac, order, alloc_flags,
                compact_result, &compact_priority,
                &compaction_retries))
        goto retry;
```


​        // 根据 nodemask 中的内存分配策略判断是否应该在进程所允许运行的所有 CPU 关联的 NUMA 节点上重试
​        if (check_retry_cpuset(cpuset_mems_cookie, ac))
​            goto retry_cpuset;
​    

```C
    // 最后的杀手锏，进行 OOM，选择一个得分最高的进程，释放其占用的内存 
    page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
    if (page)
        goto got_pg;

    // 只要 oom 产生了作用并释放了内存 did_some_progress > 0 就不断的进行重试
    if (did_some_progress) {
        no_progress_loops = 0;
        goto retry;
    }

nopage:

fail:  
      warn_alloc(gfp_mask, ac->nodemask,
            "page allocation failure: order:%u", order);
got_pg:
      return page;
}
```


retry 分支包含的是更加激进的内存分配逻辑，所以在一开始需要调用 \_\_gfp\_pfmemalloc\_flags 函数来重新调整内存分配策略，调整后的策略为：后续内存分配会忽略水位线的影响，并且允许内核从紧急预留内存中获取内存。

```C
static inline int __gfp_pfmemalloc_flags(gfp_t gfp_mask)
{
    // 如果不允许从紧急预留内存中分配，则不改变 alloc_flags
    if (unlikely(gfp_mask & __GFP_NOMEMALLOC))
        return 0;
    // 如果允许从紧急预留内存中分配，则后面的内存分配会忽略内存水位线的限制
    if (gfp_mask & __GFP_MEMALLOC)
        return ALLOC_NO_WATERMARKS;
    // 当前进程处于软中断上下文并且进程设置了 PF_MEMALLOC 标识
    // 则忽略内存水位线
    if (in_serving_softirq() && (current->flags & PF_MEMALLOC))
        return ALLOC_NO_WATERMARKS;
    // 当前进程不在任何中断上下文中
    if (!in_interrupt()) {
        if (current->flags & PF_MEMALLOC)
            // 忽略内存水位线
            return ALLOC_NO_WATERMARKS;
        else if (oom_reserves_allowed(current))
            // 当前进程允许进行 OOM
            return ALLOC_OOM;
    }
    // alloc_flags 不做任何修改
    return 0;
}
```


在调整好更加激进的内存分配策略 alloc\_flags 之后，内核会首先尝试从伙伴系统中进行一次内存分配，这时会有很大概率促使内存分配成功。

> 注意：此次尝试进行的内存分配会忽略内存水位线：ALLOC\_NO\_WATERMARKS

```C
   page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
```


如果在忽略内存水位线的情况下，内存依然分配失败，则进行直接内存回收 direct\_reclaim 。

```C
   page = __alloc_pages_direct_reclaim(gfp_mask, order, alloc_flags, ac,
                            &did_some_progress);
```


经过 direct\_reclaim 之后，仍然没有足够的内存可供分配的话，那么内核会再次进行直接内存整理 direct\_compact 。

```C
    page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags, ac,
                    compact_priority, &compact_result);
```


如果 direct\_compact 之后还是没有足够的内存，那么现在内核已经处于绝境了，是时候使用杀手锏：触发 OOM 机制杀死得分最高的进程以获取更多的空闲内存。

但是在进行 OOM 之前，内核还是需要经过一系列的判断，这时就用到了我们在 《4.1 初始化内存分配慢速路径下的相关参数》小节中介绍的 costly\_order 参数了，它会影响内核是否触发 OOM 。

如果 costly\_order = true，表示此次内存分配的内存页大于 8 个页，内核会认为这是一次代价比较大的分配行为，况且此时内存已经非常危急，严重不足。在这种情况下内核认为即使触发了 OOM，也无法获取这么多的内存，依然无法满足内存分配。

所以当 costly\_order = true 时，内核不会触发 OOM，直接跳转到 nopage 分支，除非设置了 \_\_GFP\_RETRY\_MAYFAIL 内存分配策略：

```C
    if (costly_order && !(gfp_mask & __GFP_RETRY_MAYFAIL))
        goto nopage;
```


下面内核也不会直接开始 OOM，而是进入到重试流程，在重试流程开始之前内核需要调用 should\_reclaim\_retry 判断是否应该进行重试，重试标准：

1.  如果内核已经重试了 MAX\_RECLAIM\_RETRIES (16) 次仍然失败，则放弃重试执行后续 OOM。
    
2.  如果内核将所有可选内存区域中的所有可回收页面全部回收之后，仍然无法满足内存的分配，那么放弃重试执行后续 OOM。
    

如果 should\_reclaim\_retry = false，后面会进一步判断是否应该进行 direct\_compact 的重试。

```C
    if (did_some_progress > 0 &&
            should_compact_retry(ac, order, alloc_flags,
                compact_result, &compact_priority,
                &compaction_retries))
        goto retry;
```


did\_some\_progress 表示上次直接内存回收具体回收了多少内存页,如果 did\_some\_progress = 0 则没有必要在进行内存整理重试了，因为内存整理的实现依赖于足够的空闲内存量。

当这些所有的重试请求都被拒绝时，杀手锏 OOM 就开始登场了：

```C
   page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
    if (page)
        goto got_pg;
```


如果 OOM 之后并没有释放内存，那么就来到 nopage 分支处理。

但是如果 did\_some\_progress > 0 表示 OOM 产生了作用，至少释放了一些内存那么就再次进行重试。

![image](image/a05d3da18150a4490ac820d1d069cc9a.png)

#### 4.4 nopage

到现在为止，内核已经尝试了包括 OOM 在内的所有回收内存的措施，但是仍然没有足够的内存来满足分配要求，看上去此次内存分配就要宣告失败了。

但是这里还有一定的回旋余地，如果内存分配策略中配置了 \_\_GFP\_NOFAIL，则表示此次内存分配非常的重要，不允许失败。内核会在这里不停的重试直到分配成功为止。

我们在 [《深入理解 Linux 物理内存管理》](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486879&idx=1&sn=0bcc59a306d59e5199a11d1ca5313743&chksm=ce77cbd8f90042ce06f5086b1c976d1d2daa57bc5b768bac15f10ee3dc85874bbeddcd649d88&token=1274956236&lang=zh_CN#rd)一文中的 “ 3.2 非一致性内存访问 NUMA 架构 ” 小节，介绍 NUMA 内存架构的时候曾经提到：当 CPU 自己所在的本地 NUMA 节点内存不足时，CPU 就需要跨 NUMA 节点去访问其他内存节点，**这种跨 NUMA 节点分配内存的行为就发生在这里，这种情况下 CPU 访问内存就会慢很多**。

```C
static inline struct page *
__alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
                        struct alloc_context *ac)
{
        ......... 初始化慢速内存分配路径下的相关参数 .......

retry_cpuset:

        ......... 调整内存分配策略 alloc_flags 采用更加激进方式获取内存 ......
        ......... 此时内存分配主要是在进程所允许运行的 CPU 相关联的 NUMA 节点上 ......
        ......... 内存水位线下调至 WMARK_MIN ...........
        ......... 唤醒所有 kswapd 进程进行异步内存回收  ...........
        ......... 触发直接内存整理 direct_compact 来获取更多的连续空闲内存 ......

retry:

        ......... 进一步调整内存分配策略 alloc_flags 使用更加激进的非常手段尽心内存分配 ...........
        ......... 在内存分配时忽略内存水位线 ...........
        ......... 触发直接内存回收 direct_reclaim ...........
        ......... 再次触发直接内存整理 direct_compact ...........
        ......... 最后的杀手锏触发 OOM 机制  ...........

nopage:
    // 流程走到这里表明内核已经尝试了包括 OOM 在内的所有回收内存的动作。
    // 但是这些措施依然无法满足内存分配的需求，看上去内存分配到这里就应该失败了。
    // 但是如果设置了 __GFP_NOFAIL 表示不允许内存分配失败，那么接下来就会进入 if 分支进行处理
    if (gfp_mask & __GFP_NOFAIL) {
        // 如果不允许进行直接内存回收，则跳转至 fail 分支宣告失败
        if (WARN_ON_ONCE_GFP(!can_direct_reclaim, gfp_mask))
            goto fail;

        // 此时内核已经无法通过回收内存来获取可供分配的空闲内存了
        // 对于 PF_MEMALLOC 类型的内存分配请求，内核现在无能为力，只能不停的进行 retry 重试。
        WARN_ON_ONCE_GFP(current->flags & PF_MEMALLOC, gfp_mask);

        // 对于需要分配 8 个内存页以上的大内存分配，并且设置了不可失败标识 __GFP_NOFAIL
        // 内核现在也无能为力，毕竟现实是已经没有空闲内存了，只是给出一些告警信息
        WARN_ON_ONCE_GFP(order > PAGE_ALLOC_COSTLY_ORDER, gfp_mask);

       // 在 __GFP_NOFAIL 情况下，尝试进行跨 NUMA 节点内存分配
        page = __alloc_pages_cpuset_fallback(gfp_mask, order, ALLOC_HARDER, ac);
        if (page)
            goto got_pg;
        // 在进行内存分配重试流程之前，需要让 CPU 重新调度到其他进程上
        // 运行一会其他进程，因为毕竟此时内存已经严重不足
        // 立马重试的话只能浪费过多时间在搜索空闲内存上，导致其他进程处于饥饿状态。
        cond_resched();
        // 跳转到 retry 分支，重试内存分配流程
        goto retry;
    }

fail:
      warn_alloc(gfp_mask, ac->nodemask,
            "page allocation failure: order:%u", order);
got_pg:
      return page;
}
```


这里笔者需要着重强调的一点就是，在 nopage 分支中决定开始重试之前，内核不能立即进行重试流程，因为之前已经经历过那么多严格激进的内存回收策略仍然没有足够的内存，内存现状非常紧急。

所以我们有理由相信，如果内核立即开始重试的话，依然没有什么效果，反而会浪费过多时间在搜索空闲内存上，导致其他进程处于饥饿状态。

所以在开始重试之前，内核会调用 `cond_resched()` 让 CPU 重新调度到其他进程上，让其他进程也运行一会，与此同时 kswapd 进程一直在后台异步回收着内存。

当 CPU 重新调度回当前进程时，说不定 kswapd 进程已经回收了足够多的内存，重试成功的概率会大大增加同时又避免了资源的无谓消耗。

### 5\. \_\_alloc\_pages 内存分配流程总览

到这里为止，笔者就为大家完整地介绍完内核分配内存的整个流程，现在笔者再把内存分配的完整流程图放出来，我们在结合完整的内存分配相关源码，整体在体会一下：

![image](image/a3e57951c1f4b5765eeed65759954ad5.png)

```C
static inline struct page *
__alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
                        struct alloc_context *ac)
{
    // 在慢速内存分配路径中可能会导致内核进行直接内存回收
    // 这里设置 __GFP_DIRECT_RECLAIM 表示允许内核进行直接内存回收
    bool can_direct_reclaim = gfp_mask & __GFP_DIRECT_RECLAIM;
    // 本次内存分配是否是针对大量内存页的分配，内核定义 PAGE_ALLOC_COSTLY_ORDER = 3
    // 也就是说内存请求内存页的数量大于 2 ^ 3 = 8 个内存页时，costly_order = true，后续会影响是否进行 OOM
    const bool costly_order = order > PAGE_ALLOC_COSTLY_ORDER;
    // 用于指向成功申请的内存
    struct page *page = NULL;
    // 内存分配标识，后续会根据不同标识进入到不同的内存分配逻辑处理分支
    unsigned int alloc_flags;
    // 后续用于记录直接内存回收了多少内存页
    unsigned long did_some_progress;
    // 关于内存整理相关参数
    enum compact_priority compact_priority;
    enum compact_result compact_result;
    int compaction_retries;
    int no_progress_loops;
    unsigned int cpuset_mems_cookie;
    int reserve_flags;

    // 流程现在来到了慢速内存分配这里，说明快速分配路径已经失败了
    // 内核需要对 gfp_mask 分配行为掩码做一些修改，修改为一些更可能导致内存分配成功的标识
    // 因为接下来的直接内存回收非常耗时可能会导致进程阻塞睡眠，不适用原子 __GFP_ATOMIC 内存分配的上下文。
    if (WARN_ON_ONCE((gfp_mask & (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)) ==
                (__GFP_ATOMIC|__GFP_DIRECT_RECLAIM)))
        gfp_mask &= ~__GFP_ATOMIC;

retry_cpuset:

    // 在之前的快速内存分配路径下设置的相关分配策略比较保守，不是很激进，用于在 WMARK_LOW 水位线之上进行快速内存分配
    // 走到这里表示快速内存分配失败，此时空闲内存严重不足了
    // 所以在慢速内存分配路径下需要重新设置更加激进的内存分配策略，采用更大的代价来分配内存
    alloc_flags = gfp_to_alloc_flags(gfp_mask);

    // 重新按照新的设置按照内存区域优先级计算 zonelist 的迭代起点（最高优先级的 zone）
    // fast path 和 slow path 的设置不同所以这里需要重新计算
    ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
                    ac->highest_zoneidx, ac->nodemask);
    // 如果没有合适的内存分配区域，则跳转到 nopage , 内存分配失败
    if (!ac->preferred_zoneref->zone)
        goto nopage;
    // 唤醒所有的 kswapd 进程异步回收内存
    if (alloc_flags & ALLOC_KSWAPD)
        wake_all_kswapds(order, gfp_mask, ac);

    // 此时所有的 kswapd 进程已经被唤醒，正在异步进行内存回收
    // 之前我们已经在 gfp_to_alloc_flags 方法中重新调整了 alloc_flags
    // 换成了一套更加激进的内存分配策略，注意此时是在 WMARK_MIN 水位线之上进行内存分配
    // 调整后的 alloc_flags 很可能会立即成功，因此这里先尝试一下
    page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
    if (page)
        // 内存分配成功，跳转到 got_pg 直接返回 page
        goto got_pg;

    // 对于分配大内存来说 costly_order = true (超过 8 个内存页)，需要首先进行内存整理，这样内核可以避免直接内存回收从而获取更多的连续空闲内存页
    // 对于需要分配不可移动的高阶内存的情况，也需要先进行内存整理，防止永久内存碎片
    if (can_direct_reclaim &&
            (costly_order ||
               (order > 0 && ac->migratetype != MIGRATE_MOVABLE))
            && !gfp_pfmemalloc_allowed(gfp_mask)) {
        // 进行直接内存整理，获取更多的连续空闲内存防止内存碎片
        page = __alloc_pages_direct_compact(gfp_mask, order,
                        alloc_flags, ac,
                        INIT_COMPACT_PRIORITY,
                        &compact_result);
        if (page)
            goto got_pg;

        if (costly_order && (gfp_mask & __GFP_NORETRY)) {
            // 流程走到这里表示经过内存整理之后依然没有足够的内存供分配
            // 但是设置了 NORETRY 标识不允许重试，那么就直接失败，跳转到 nopage
            if (compact_result == COMPACT_SKIPPED ||
                compact_result == COMPACT_DEFERRED)
                goto nopage;
            // 同步内存整理开销太大，后续开启异步内存整理
            compact_priority = INIT_COMPACT_PRIORITY;
        }
    }

retry:
    // 确保所有 kswapd 进程不要意外进入睡眠状态
    if (alloc_flags & ALLOC_KSWAPD)
        wake_all_kswapds(order, gfp_mask, ac);

    // 流程走到这里，说明在 WMARK_MIN 水位线之上也分配内存失败了
    // 并且经过内存整理之后，内存分配仍然失败，说明当前内存容量已经严重不足
    // 接下来就需要使用更加激进的非常手段来尝试内存分配（忽略掉内存水位线），继续修改 alloc_flags 保存在 reserve_flags 中
    reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
    if (reserve_flags)
        alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, reserve_flags);

    // 如果内存分配可以任意跨节点分配（忽略内存分配策略），这里需要重置 nodemask 以及 zonelist。
    if (!(alloc_flags & ALLOC_CPUSET) || reserve_flags) {
        // 这里的内存分配是高优先级系统级别的内存分配，不是面向用户的
        ac->nodemask = NULL;
        ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
                    ac->highest_zoneidx, ac->nodemask);
    }

    // 这里使用重新调整的 zonelist 和 alloc_flags 在尝试进行一次内存分配
    // 注意此次的内存分配是忽略内存水位线的 ALLOC_NO_WATERMARKS
    page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
    if (page)
        goto got_pg;

    // 在忽略内存水位线的情况下仍然分配失败，现在内核就需要进行直接内存回收了
    if (!can_direct_reclaim)
        // 如果进程不允许进行直接内存回收，则只能分配失败
        goto nopage;

    // 开始直接内存回收
    page = __alloc_pages_direct_reclaim(gfp_mask, order, alloc_flags, ac,
                            &did_some_progress);
    if (page)
        goto got_pg;

    // 直接内存回收之后仍然无法满足分配需求，则再次进行直接内存整理
    page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags, ac,
                    compact_priority, &compact_result);
    if (page)
        goto got_pg;

    // 在内存直接回收和整理全部失败之后，如果不允许重试，则只能失败
    if (gfp_mask & __GFP_NORETRY)
        goto nopage;

    // 后续会触发 OOM 来释放更多的内存，这里需要判断本次内存分配是否需要分配大量的内存页（大于 8 ） costly_order = true
    // 如果是的话则内核认为即使执行 OOM 也未必会满足这么多的内存页分配需求.
    // 所以还是直接失败比较好，不再执行 OOM，除非设置 __GFP_RETRY_MAYFAIL
    if (costly_order && !(gfp_mask & __GFP_RETRY_MAYFAIL))
        goto nopage;

    // 流程走到这里说明我们已经尝试了所有措施内存依然分配失败了，此时内存已经非常危急了。
    // 走到这里说明进程允许内核进行重试流程，但在开始重试之前，内核需要判断是否应该进行重试,重试标准：
    // 1 如果内核已经重试了 MAX_RECLAIM_RETRIES (16) 次仍然失败，则放弃重试执行后续 OOM。
    // 2 如果内核将所有可选内存区域中的所有可回收页面全部回收之后，仍然无法满足内存的分配，那么放弃重试执行后续 OOM
    if (should_reclaim_retry(gfp_mask, order, ac, alloc_flags,
                 did_some_progress > 0, &no_progress_loops))
        goto retry;

    // 如果内核判断不应进行直接内存回收的重试，这里还需要判断下是否应该进行内存整理的重试。
    // did_some_progress 表示上次直接内存回收具体回收了多少内存页
    // 如果 did_some_progress = 0 则没有必要在进行内存整理重试了，因为内存整理的实现依赖于足够的空闲内存量
    if (did_some_progress > 0 &&
            should_compact_retry(ac, order, alloc_flags,
                compact_result, &compact_priority,
                &compaction_retries))
        goto retry;
```


​        // 根据 nodemask 中的内存分配策略判断是否应该在进程所允许运行的所有 CPU 关联的 NUMA 节点上重试
​        if (check_retry_cpuset(cpuset_mems_cookie, ac))
​            goto retry_cpuset;
​    

```C
    // 最后的杀手锏，进行 OOM，选择一个得分最高的进程，释放其占用的内存 
    page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
    if (page)
        goto got_pg;

    // 只要 oom 产生了作用并释放了内存 did_some_progress > 0 就不断的进行重试
    if (did_some_progress) {
        no_progress_loops = 0;
        goto retry;
    }

nopage:
    // 流程走到这里表明内核已经尝试了包括 OOM 在内的所有回收内存的动作。
    // 但是这些措施依然无法满足内存分配的需求，看上去内存分配到这里就应该失败了。
    // 但是如果设置了 __GFP_NOFAIL 表示不允许内存分配失败，那么接下来就会进入 if 分支进行处理
    if (gfp_mask & __GFP_NOFAIL) {
        // 如果不允许进行直接内存回收，则跳转至 fail 分支宣告失败
        if (WARN_ON_ONCE_GFP(!can_direct_reclaim, gfp_mask))
            goto fail;

        // 此时内核已经无法通过回收内存来获取可供分配的空闲内存了
        // 对于 PF_MEMALLOC 类型的内存分配请求，内核现在无能为力，只能不停的进行 retry 重试。
        WARN_ON_ONCE_GFP(current->flags & PF_MEMALLOC, gfp_mask);

        // 对于需要分配 8 个内存页以上的大内存分配，并且设置了不可失败标识 __GFP_NOFAIL
        // 内核现在也无能为力，毕竟现实是已经没有空闲内存了，只是给出一些告警信息
        WARN_ON_ONCE_GFP(order > PAGE_ALLOC_COSTLY_ORDER, gfp_mask);

       // 在 __GFP_NOFAIL 情况下，尝试进行跨 NUMA 节点内存分配
        page = __alloc_pages_cpuset_fallback(gfp_mask, order, ALLOC_HARDER, ac);
        if (page)
            goto got_pg;
        // 在进行内存分配重试流程之前，需要让 CPU 重新调度到其他进程上
        // 运行一会其他进程，因为毕竟此时内存已经严重不足
        // 立马重试的话只能浪费过多时间在搜索空闲内存上，导致其他进程处于饥饿状态。
        cond_resched();
        // 跳转到 retry 分支，重试内存分配流程
        goto retry;
    }
fail:
    warn_alloc(gfp_mask, ac->nodemask,
            "page allocation failure: order:%u", order);
got_pg:
    return page;
}
```


现在内存分配流程中涉及到的三个重要辅助函数：prepare\_alloc\_pages，\_\_alloc\_pages\_slowpath，get\_page\_from\_freelist 。笔者已经为大家介绍了两个了。prepare\_alloc\_pages，\_\_alloc\_pages\_slowpath 函数主要是根据不同的空闲内存剩余容量调整内存的分配策略，尽量使内存分配行为尽最大可能成功。

理解了以上两个辅助函数的逻辑，我们就相当于梳理清楚了整个内存分配的链路流程。但目前我们还没有涉及到具体内存分配的真正逻辑，而内核中执行具体内存分配动作是在 get\_page\_from\_freelist 函数中，这也是掌握内存分配的最后一道关卡。

由于 get\_page\_from\_freelist 函数执行的是具体的内存分配动作，所以它和内核中的伙伴系统有着千丝万缕的联系，而本文的主题更加侧重描述整个物理内存分配的链路流程，考虑到文章篇幅的关系，笔者把伙伴系统这部分的内容放在下篇文章为大家讲解。

### 总结

本文首先从 Linux 内核中常见的几个物理内存分配接口开始，介绍了这些内存分配接口的各自的使用场景，以及接口函数中参数的含义。

![image](image/67c35ff0215a21ec4c132052c7e0d696.png)

并以此为起点，结合 Linux 内核 5.19 版本源码详细讨论了物理内存分配在内核中的整个链路实现。在整个链路中，内存的分配整体分为了两个路径：

1.  快速路径 fast path：该路径的下，内存分配的逻辑比较简单，主要是在 WMARK\_LOW 水位线之上快速的扫描一下各个内存区域中是否有足够的空闲内存能够满足本次内存分配，如果有则立马从伙伴系统中申请，如果没有立即返回。
    
2.  慢速路径 slow path：慢速路径下的内存分配逻辑就变的非常复杂了，其中包含了内存分配的各种异常情况的处理，并且会根据文中介绍的 GFP\_，ALLOC\_ 等各种内存分配策略掩码进行不同分支的处理，整个链路非常庞大且繁杂。
    

本文铺垫了大量的内存分配细节，但是整个内存分配链路流程的精髓，笔者绘制在了下面这副流程图中，方便大家忘记的时候回顾。

![image](image/aee5086d07533e5c18a511a28fa6ea74.png)

## 参考

[深入理解 Linux 物理内存分配全链路实现_服务器内存链路指什么-CSDN博客](https://blog.csdn.net/weixin_47282449/article/details/128519787)