---
layout: post
title: "memcached过期策略问题追查"
date: 2014-09-30 00:26:03 +0800
comments: true
categories: memcache 
---
##背景
线上热点数据几乎都存放在memcached里，采用的经典方案，优先从memcached获取数据，如果获取失败，再从MySQL获取，同时回填memcached。随着业务的飞速增长，数据量已经超过了memcached设置的最大内存，因为出现了内存置换出的情况，往往2天前的热点数据会被唤出，这也很正常。

因为新需求，缓存中需要存放新的数据。但是实际测试发现，缓存中的数据几分钟就会被失效，导致MySQL压力很大。

为什么同一个memcached的数据，有的缓存（后面简称为旧数据）要2天才会被置换出，有的缓存（后面简称新数据）几分钟就会被换出？
<!--more-->
##分析
首先分析新旧数据的不同：

1，key肯定不同

2，value大小上，旧数据value较大，新数据value很小

memcached是按照slabs作为内存单元来分配。新旧数据value差异较大，肯定位于不同的chunk里面。考虑到memcached内存已经占满，会不停置换内存。为什么总是新数据被置换出来，而旧数据不容易被置换呢？只能去看memcached的代码找寻答案。

主要代码位于`memcached\items.c`的`do_item_alloc`。[参考这里](https://github.com/memcached/memcached/blob/e73bc2e5c0794cccd6f8ece63bc16433c40ed766/items.c)，核心代码摘录如下：

``` c
item *do_item_alloc(char *key, const size_t nkey, const int flags,
                    const rel_time_t exptime, const int nbytes,
                    const uint32_t cur_hv) 
{
	//.....
	 /* Expired or flushed */
        if ((search->exptime != 0 && search->exptime < current_time)
            || (search->time <= oldest_live && oldest_live <= current_time)) {
            itemstats[id].reclaimed++;
            if ((search->it_flags & ITEM_FETCHED) == 0) {
                itemstats[id].expired_unfetched++;
            }
            it = search;
            slabs_adjust_mem_requested(it->slabs_clsid, ITEM_ntotal(it), ntotal);
            do_item_unlink_nolock(it, hv);
            /* Initialize the item block: */
            it->slabs_clsid = 0;
        } else if ((it = slabs_alloc(ntotal, id)) == NULL) {
            tried_alloc = 1;
            if (settings.evict_to_free == 0) {
                itemstats[id].outofmemory++;
            } else {
                itemstats[id].evicted++;
                itemstats[id].evicted_time = current_time - search->time;
                if (search->exptime != 0)
                    itemstats[id].evicted_nonzero++;
                if ((search->it_flags & ITEM_FETCHED) == 0) {
                    itemstats[id].evicted_unfetched++;
                }
                it = search;
                slabs_adjust_mem_requested(it->slabs_clsid, ITEM_ntotal(it), ntotal);
                do_item_unlink_nolock(it, hv);
                /* Initialize the item block: */
                it->slabs_clsid = 0;
                //.....
		}
	}
}
```
1, 首先从LRU队列中寻找是否有过期的item可用（代码7-17行）。需要说明的是，这里的LRU队列是每一chunk一个队列，而不是全局统一一个。

2，如果LRU没有过期数据，尝试初始化一个新的slab（代码18行），并分配给该chunk使用。

3，如果第二步失败（比如内存不够用了），则只能从LRU队列中淘汰最旧未使用的item了（代码23-34行）。

原因到此基本查明了，缓存数据的过期时间都没有设置，因此默认就是30天。这样当内存写满的情况下，分配一个item，前两步都不会满足，走到第三步。

对于旧数据，因为跑了很久，该chunk已经占用了很多的slabs，所以通过LRU置换，问题并不大。

对于新数据，因为value大小差异较大，自然用的是一个没多少slabs的chunk, 通过LRU置换，就会出现问题，导致频繁被置换。

可以想到，如果这时候重启了memcached，这样新旧数据会比较公平，一段时间后都会分配差不多的item（假设新旧数据使用频率差不多），这样LRU换出的话，问题也不大。

##解决
1，重启memcached，解决这种新旧数据不公平的情况。

2，分配更大的memcached，避免出现换出。

##后续
1，memcached可以使用stats 看evictions 的数据，如果不为0，说明此时memcached分配内存出现了换出。

2，如果数据使用频率差异很大，还是会发生这种情况。这时候就会麻烦一些，可以考虑分不同的memcached存储，或者预先用假数据预热缓存，目的就是占住LRU的位置。
