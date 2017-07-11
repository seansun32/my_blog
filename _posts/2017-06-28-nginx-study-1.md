---
layout:     post
title:      "Nginx源码学习1---数据结构:内存池"
date:       2017-06-28 14:55:56 +0800
categories: 源码学习
tag:        Nginx系列
---

* content
{: toc}

今天开始，系统学习nginx源码，并写测试代码加深理解，测试代码将放至github。
该博客用于本人自己学习和纪录，如有错误，欢迎指正。

前言：
===

nginx的内存池是一个值得研究的地方，nginx所有的内存分配、释放，均由内存池管理。
总的来说，nginx的内存池就是一个链表，对于要分配的内存大小，从链表头开始查找，
如果在链表节点有多余空间，则在此节点分配该空间。如果所有节点均无可分配空间，
则在链表中新加入链表节点。

如果待分配的空间超过链表空间大小，则需要用`ngx_pool_large_t`,此为单独出来的链表，
用于分配大内存空间。



数据结构：
===

ngx_pool_t结构
---

```c
struct ngx_pool_s {
    ngx_pool_data_t       d;
    size_t                max;
    ngx_pool_t           *current;
    ngx_chain_t          *chain;
    ngx_pool_large_t     *large;
    ngx_pool_cleanup_t   *cleanup;
    ngx_log_t            *log;
};
```
*   d:内存池的数据区
*   max:内存池最大分配空间,如果待分配空间超过max，则需用大块内存链表
*   current:指向当前内存池的链表节点
*   chain:指向nginx_chain_t结构，目前还不清楚其作用，待研究
*   large：指向大块内存链表
*   cleanup:链表，用于释放外部资源
*   log:日志


接下来具体来看ngx_pool_t里面的元素。

---

ngx_pool_data_t结构
---

```c
typedef struct {
    u_char               *last;
    u_char               *end;
    ngx_pool_t           *next;
    ngx_uint_t            failed;
} ngx_pool_data_t;
```
内存池链表节点的数据结构

*   last:在内存池的一个链表节点中，上一次分配内存空间的结束位置，即下一次分配内存空间的开始位置
*   end:内存池的链表节点结束地址。注意，end-last即为该链表节点可分配的内存空间
*   next：指向下一个链表节点
*   failed：在一个链表节点中，分配内存失败的次数（如果失败次数大于5，则认为该链表不能再分配空间（已满））

可以通过一副图来说明ngx_pool_t和ngx_pool_data_t的关系
[图片来源](http://blog.csdn.net/chen19870707/article/details/41015613)

![nginx pic1](../../../../styles/images/nginx/nginx1/ngx_pool_1.jpg)

图中可以看到，在该内存池的链表中，只有链表头节点才有max，current，large等元素，
其他节点除了ngx_pool_data_t元素，剩下的均用来分配空间

---

ngx_pool_large_t结构
---


```c
struct ngx_pool_large_s {
    ngx_pool_large_t     *next;
    void                 *alloc;
};
```

内存池大块内存链表的数据结构

*   next:指向像一个大块内存链表节点
*   alloc:指向大块内存分配的地址

继续用图片来说明
[图片来源](http://blog.csdn.net/chen19870707/article/details/41015613)

![nginx pic1](../../../../styles/images/nginx/nginx1/ngx_pool_2.png)


---

ngx_pool_cleanup_t结构
--

```c
struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt   handler;
    void                 *data;
    ngx_pool_cleanup_t   *next;
};
```

释放外部数据的链表

*   handler:注册的回掉函数以释放资源,typedef void (*ngx_pool_cleanup_pt)(void *data);
*   data:需释放的外部数据
*   next:指向下一个链表节点

数据结构的关系见图
[图片来源](http://blog.csdn.net/chen19870707/article/details/41015613)

![nginx pic1](../../../../styles/images/nginx/nginx1/ngx_pool_3.png)



通过以上数据结构可以看出，通过内存池，我们可以操作各个相关链表，通过对这些链表的操作，
即可以做到统一对内存的分配释放作出管理。


相关函数：
===


创建内存池
---

```c
ngx_pool_t *
ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;
    
    //分配size大小的空间，以NGX_POOL_ALIGNMENT大小的字节对齐。linux平台下，ngx_memalign为posix_memalign的封装
    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
    if (p == NULL) {
        return NULL;
    }

    //last指针指向即将分配的内存的起始位置，第一次创建内存池时，即ngx_pool_t之后即可分配
    p->d.last = (u_char *) p + sizeof(ngx_pool_t);
    //end指向内存池中链表节点的结束位置。end-last即为可分配空间大小
    p->d.end = (u_char *) p + size;
    p->d.next = NULL;
    p->d.failed = 0;

    //可用的size需减去存储该结构体本身的空间
    size = size - sizeof(ngx_pool_t);
    //设置p->max,如过需分配的空间超过p->max,则需用大块内存链表
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;
    
    //初始化各元素
    p->current = p;
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;

    return p;
}
```
该接口的目的为创建内存池，即建立了内存池链表


销毁内存池
---

```c
void
ngx_destroy_pool(ngx_pool_t *pool)
{
    ngx_pool_t          *p, *n;
    ngx_pool_large_t    *l;
    ngx_pool_cleanup_t  *c;

    //通过cleanup链表来释放相关的data
    for (c = pool->cleanup; c; c = c->next) {
        if (c->handler) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "run cleanup: %p", c);
            c->handler(c->data);
        }
    }
    
    //释放大块内存链表
    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            ngx_free(l->alloc);
        }
    }
    
    //释放内存池链表
    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next) {
        ngx_free(p);

        if (n == NULL) {
            break;
        }
    }
}
```
该接口通过对各链表的遍历，来释放已分配的内存


分配内存
---

```c

void *
ngx_palloc(ngx_pool_t *pool, size_t size)
{
    //直接调用ngx_palloc_large分配,具体原因见后面分析
    return ngx_palloc_large(pool, size);
}
```

```c
static void *
ngx_palloc_large(ngx_pool_t *pool, size_t size)
{
    void              *p;
    ngx_uint_t         n;
    ngx_pool_large_t  *large;
    
    //malloc的封装，分配size大小的内存
    p = ngx_alloc(size, pool->log);
    if (p == NULL) {
        return NULL;
    }

    n = 0;
    
    //遍历大块内存链表，如有alloc为空的，则将分配的p挂在此节点
    for (large = pool->large; large; large = large->next) {
        if (large->alloc == NULL) {
            large->alloc = p;
            return p;
        }
        
        //如果头三个节点都不为空，则跳出循环，准备新建大块内存链表节点
        if (n++ > 3) {
            break;
        }
    }
    
    //新建大块内存链表节点
    large = ngx_palloc_small(pool, sizeof(ngx_pool_large_t), 1);
    if (large == NULL) {
        ngx_free(p);
        return NULL;
    }
    
    //将新建立的节点加在链表头部
    large->alloc = p;
    large->next = pool->large;
    pool->large = large;

    return p;
}
```

```c
static ngx_inline void *
ngx_palloc_small(ngx_pool_t *pool, size_t size, ngx_uint_t align)
{
    u_char      *m;
    ngx_pool_t  *p;

    p = pool->current;
    
    //从current处往后遍历内存池链表，current之前的表示链表节点已分配满（可能有内存浪费）
    do {
        m = p->d.last;

        if (align) {
            //对内存地址以NGX_ALIGNMENT对齐
            m = ngx_align_ptr(m, NGX_ALIGNMENT);
        }
        
        //如果节点可用空间大于待分配空间，则返回此节点
        if ((size_t) (p->d.end - m) >= size) {
            p->d.last = m + size;

            return m;
        }

        p = p->d.next;

    } while (p);

    //如果现有链表没有满足size的节点，则新建立链表节点
    return ngx_palloc_block(pool, size);
}
```

说明:

```c
#define ngx_align_ptr(p, a)                                                   \
    (u_char *) (((uintptr_t) (p) + ((uintptr_t) a - 1)) & ~((uintptr_t) a - 1))
```

`&~`的作用是抹去last后面的a-1位的值,比如(char *)(((unsigned long long)p) & (~(7))),就是对低三位清0，
这样得到的为地址为8的整数倍，即8byte对齐



```c
static void *
ngx_palloc_block(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    size_t       psize;
    ngx_pool_t  *p, *new;
    
    //分配大小跟内存池头节点大小一致
    psize = (size_t) (pool->d.end - (u_char *) pool);
    
    //分配psize的内存，对齐
    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
    if (m == NULL) {
        return NULL;
    }

    new = (ngx_pool_t *) m;

    new->d.end = m + psize;
    new->d.next = NULL;
    new->d.failed = 0;

    m += sizeof(ngx_pool_data_t);
    m = ngx_align_ptr(m, NGX_ALIGNMENT);

    //size 通常为ngx_pool_large_t的大小，不会超过psize
    new->d.last = m + size;
    
    //每调用一次ngx_palloc_block,说明pool->current之后的节点均不满足要分配的空间，failed+1
    //failed超过4时，认为该节点已满，不能再分配空间，current向后移动，指向“未满”的节点
    //节点是否“满”，靠失败次数决定，可能造成内存浪费
    for (p = pool->current; p->d.next; p = p->d.next) {
        if (p->d.failed++ > 4) {
            pool->current = p->d.next;
        }
    }

    p->d.next = new;

    return m;
}
```

以上我们可以发现，调用`ngx_palloc_small`,分配的只是`ngx_pool_large_t`结构的空间(**分配空间用ngx_alloc**)，
所以，内存池链表节点中的`ngx_pool_data_t`只是存放ngx_pool_large_t的空间(**分配空间用ngx_memalign**)，
而真正用户要分配的空间，则统统存在大块内存链表中。所以前面的`ngx_palloc`接口（用户调用来分配空间），
直接返回`ngx_palloc_large`


重置内存池
---

```c
void
ngx_reset_pool(ngx_pool_t *pool)
{
    ngx_pool_t        *p;
    ngx_pool_large_t  *l;
    
    //遍历大块内存链表，并逐一释放
    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            ngx_free(l->alloc);
        }
    }
    
    //重置内存池链表，注意此时没有释放
    for (p = pool; p; p = p->d.next) {
        
        //注意，没有用sizeof(ngx_pool_data_t)
        p->d.last = (u_char *) p + sizeof(ngx_pool_t);
        p->d.failed = 0;
    }

    pool->current = pool;
    pool->chain = NULL;
    pool->large = NULL;
}
```
---

大块内存链表节点释放
---

```c
ngx_int_t
ngx_pfree(ngx_pool_t *pool, void *p)
{
    ngx_pool_large_t  *l;

    for (l = pool->large; l; l = l->next) {
        if (p == l->alloc) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "free: %p", l->alloc);
            ngx_free(l->alloc);
            l->alloc = NULL;

            return NGX_OK;
        }
    }

    return NGX_DECLINED;
```

因为之前大块内存链表节点中，头三个节点不是空闲的，就会新建大块内存节点，
所以即时的释放，防止过多不必要的内存消耗

---

外部资源释放
---

```c
ngx_pool_cleanup_t *
ngx_pool_cleanup_add(ngx_pool_t *p, size_t size)
{
    ngx_pool_cleanup_t  *c;
    
    //新建cleanup链表节点
    c = ngx_palloc(p, sizeof(ngx_pool_cleanup_t));
    if (c == NULL) {
        return NULL;
    }

    if (size) {
        c->data = ngx_palloc(p, size);
        if (c->data == NULL) {
            return NULL;
        }

    } else {
        c->data = NULL;
    }
    //将链表节点加入链表头
    c->handler = NULL;
    c->next = p->cleanup;

    p->cleanup = c;

    ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, p->log, 0, "add cleanup: %p", c);

    return c;
}

void
ngx_pool_run_cleanup_file(ngx_pool_t *p, ngx_fd_t fd)
{
    ngx_pool_cleanup_t       *c;
    ngx_pool_cleanup_file_t  *cf;

    for (c = p->cleanup; c; c = c->next) {

        //如果handler注册的回调为ngx_pool_cleanup_file,则执行此回调，即close(fd)的操作
        if (c->handler == ngx_pool_cleanup_file) {

            cf = c->data;

            if (cf->fd == fd) {
                c->handler(cf);
                c->handler = NULL;
                return;
            }
        }
    }
}


void
ngx_pool_cleanup_file(void *data)
{
    ngx_pool_cleanup_file_t  *c = data;

    ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, c->log, 0, "file cleanup: fd:%d",
                   c->fd);

    if (ngx_close_file(c->fd) == NGX_FILE_ERROR) {
        ngx_log_error(NGX_LOG_ALERT, c->log, ngx_errno,
                      ngx_close_file_n " \"%s\" failed", c->name);
    }
}
```
ngx_pool_t在销毁内存时，有时需要同时清理其他的外部资源，比如打开的文件，
此时就需要用ngx_pool_cleanup_t的链表，调用handler注册的函数进行清理

---


测试代码
===

[见github](https://github.com/seansun32/nginx_leanring_practice_code/tree/master/test_ngx_pool)


参考资料
===

[菜鸟nginx源码剖析数据结构篇（九） 内存池ngx_pool_t](http://blog.csdn.net/chen19870707/article/details/41015613)

