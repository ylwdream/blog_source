---
layout: post
title: "c++ alloc 函数"
date: 2014-10-31 22:16:41 +0800
comments: true
categories: 
- C/C++
---
下面看看allocate()函数里的几个函数，里面设计到内存池的管理,refill()函数，当他发现free_list中没有可用的内存块时就会被调用来查找新的内存快。
<!--more-->
## SGL 源码
下面是SGI STL中的一部分源码

``` C++ refill.cpp
/* Returns an object of size n, and optionally adds to size n free list.*/
/* We assume that n is properly aligned.                                */
/* We hold the allocation lock.                                         */
template <bool threads, int inst>
void* __default_alloc_template<threads, inst>::refill(size_t n)
{
    int nobjs = 20;
    char * chunk = chunk_alloc(n, nobjs); //从内存池中分配20个大小的块
    obj * __VOLATILE * my_free_list;
    obj * result;
    obj * current_obj, * next_obj;
    int i;

    if (1 == nobjs) return(chunk);
    my_free_list = free_list + FREELIST_INDEX(n);

    /* Build free list in chunk */
      result = (obj *)chunk;   //这一块给客端使用
      *my_free_list = next_obj = (obj *)(chunk + n); //剩下的头插法插入到相应的位置，这里的n的大小为8，16，32等
      for (i = 1; ; i++) {                      
        current_obj = next_obj;
        next_obj = (obj *)((char *)next_obj + n);
        if (nobjs - 1 == i) {
            current_obj -> free_list_link = 0;
            break;
        } else {
            current_obj -> free_list_link = next_obj;
        }
      }
    return(result);
}
```
## 分配池函数
chunk_alloc 函数的作用流程如下：先查看剩下的内存池中完全满足要求，则分配需要的量，否者分配大于1个size的量。如果内存池中没有可用的一块size的大小。则先调用malloc从堆中分配大小。否则就从剩余的其他链表空间递归的查找。如果还没满足的，就最后调用第一级的分配器。其实，这个思想和人的想法是一样的。

``` C++ chunk_alloc
template <bool threads, int inst>
char*
__default_alloc_template<threads, inst>::chunk_alloc(size_t size, int& nobjs)
{
    char * result;
    size_t total_bytes = size * nobjs;
    size_t bytes_left = end_free - start_free;

    if (bytes_left >= total_bytes) {
        result = start_free;
        start_free += total_bytes;
        return(result);
    } else if (bytes_left >= size) {
        nobjs = bytes_left/size;
        total_bytes = size * nobjs;
        result = start_free;
        start_free += total_bytes;
        return(result);
    } else {
        size_t bytes_to_get = 2 * total_bytes + ROUND_UP(heap_size >> 4);
        // Try to make use of the left-over piece.
        if (bytes_left > 0) {   //内存中的零头分配到free_list中
            obj * __VOLATILE * my_free_list =
                        free_list + FREELIST_INDEX(bytes_left);

            ((obj *)start_free) -> free_list_link = *my_free_list;
            *my_free_list = (obj *)start_free;
        }
        start_free = (char *)malloc(bytes_to_get);
        if (0 == start_free) {
            int i;
            obj * __VOLATILE * my_free_list, *p;
            // Try to make do with what we have.  That can't
            // hurt.  We do not try smaller requests, since that tends
            // to result in disaster on multi-process machines.
            for (i = size; i <= __MAX_BYTES; i += __ALIGN) {
                my_free_list = free_list + FREELIST_INDEX(i);
                p = *my_free_list;
                if (0 != p) {
                    *my_free_list = p -> free_list_link;
                    start_free = (char *)p;
                    end_free = start_free + i;
                    return(chunk_alloc(size, nobjs));
                    // Any leftover piece will eventually make it to the
                    // right free list.
                }
            }
	    end_free = 0;	// In case of exception.
            start_free = (char *)malloc_alloc::allocate(bytes_to_get);
            // This should either throw an
            // exception or remedy the situation.  Thus we assume it
            // succeeded.
        }
        heap_size += bytes_to_get;
        end_free = start_free + bytes_to_get;
        return(chunk_alloc(size, nobjs));
    }
}
```

## 总结

感觉内存分配的函数写的十分精妙，先看内存的大小是否是128bytes大小，
因为一般的大小都没有那么大，所以对系统的调用还是很少的。减少系统调用的次数，增加c++内存分配的效率
