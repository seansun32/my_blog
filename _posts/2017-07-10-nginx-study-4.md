---
layout:     post
title:      "Nginx源码学习4---数据结构:ngx_list_t"
date:       2017-07-10 16:42:50 +0800
categories: 源码学习
tag:        Nginx系列
---

* content
{: toc}

前言：
===

前面学习了ngx_array_t的数据结构及其相关代码，看到当array数组满时，
会重新分配一个2倍原数组大小的新数组，而旧的并没有释放，这样会存在内存的浪费。
所以这里介绍了新的数据结构`ngx_list_t`。它结合了数组和链表的优点，并且当
节点中的数组满时，只需要新加一个节点，而不会像ngx_array_t重新分配。
(存疑：nginx的源码什么时候用`ngx_array_t`,什么时候用`ngx_list_t`?)

---

数据结构
===

ngx_list_t结构
---

```c
typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
```

ngx_list_part_t是链表节点的数据结构
* elts:     链表节点中，指向数组首地址的指针,一个节点相当于一个数组
* nelts:    数组中已分配元素个数
* next:     指向链表下一个节点

```c
typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
```
* last:     指向链表最后一个节点
* part:     链表第一个节点(遍历链表时的起点)
* size:     链表节点中，数组元素的大小
* nalloc:   数组最大能分配元素个数，当nelts大于nalloc时，增加链表节点扩容
* pool:     内存池

内存的结构见图片

[图片来源](http://blog.csdn.net/chen19870707/article/details/40400719)

![ngx pic](../../../../styles/images/nginx/nginx4/ngx_4_1.png)

---

相关函数
===

创建链表及其初始化
---

```c
ngx_list_t *
ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size)
{
    ngx_list_t  *list;
    
    //内存池分配ngx_list_t
    list = ngx_palloc(pool, sizeof(ngx_list_t));
    if (list == NULL) {
        return NULL;
    }
    
    //初始化
    if (ngx_list_init(list, pool, n, size) != NGX_OK) {
        return NULL;
    }

    return list;
}

static ngx_inline ngx_int_t
ngx_list_init(ngx_list_t *list, ngx_pool_t *pool, ngx_uint_t n, size_t size)
{   
    //为链表头节点分配空间，即数组大小
    list->part.elts = ngx_palloc(pool, n * size);
    if (list->part.elts == NULL) {
        return NGX_ERROR;
    }
    
    //初始化链表
    list->part.nelts = 0;
    list->part.next = NULL;
    list->last = &list->part;
    list->size = size;
    list->nalloc = n;
    list->pool = pool;

    return NGX_OK;
}
```

向链表节点中的数组添加元素
---

```c
void *
ngx_list_push(ngx_list_t *l)
{
    void             *elt;
    ngx_list_part_t  *last;

    last = l->last;
    
    //如果链表尾节点数组已满，则新添加链表节点
    if (last->nelts == l->nalloc) {

        /* the last part is full, allocate a new list part */
        
        //为新节点分配空间
        last = ngx_palloc(l->pool, sizeof(ngx_list_part_t));
        if (last == NULL) {
            return NULL;
        }
        
        //为新节点的数组分配空间
        last->elts = ngx_palloc(l->pool, l->nalloc * l->size);
        if (last->elts == NULL) {
            return NULL;
        }

        last->nelts = 0;
        last->next = NULL;
        
        //将新节点加入到链表
        l->last->next = last;
        l->last = last;
    }

    //elt指向新加入的数组元素地址
    elt = (char *) last->elts + l->size * last->nelts;
    last->nelts++;

    return elt;
}
```

测试代码
===

[见github](https://github.com/seansun32/nginx_leanring_practice_code/tree/master/test_ngx_list)

参考资料
===

[菜鸟nginx源码剖析数据结构篇（三）单向链表ngx_lig_t](http://blog.csdn.net/chen19870707/article/details/40400719)


注意到，和`ngx_array_t`一样，`ngx_list_t`同样只有添加新元素，没有删除新元素（需要思考这样做的原因？）
但是和`ngx_array_t`不同的是，`ngx_list_t`没有list_destroy接口，内存的回收
统一由内存池管理。




