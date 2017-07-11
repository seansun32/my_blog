---
layout:     post
title:      "Nginx源码学习3---数据结构:ngx_array_t"
date:       2017-07-10 11:10:25 +0800
categories: 源码学习
tag:        Nginx系列
---

* content
{: toc}

前言：
===

ngx_array_t是一个设计精巧的动态数组，既有数组通过下标直接访问的优点，
又能够动态的扩容，值得学习和借鉴。


---

数据结构
===

ngx_array_t结构
---

```c
typedef struct {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
} ngx_array_t;
```

* elts:     指向数组的首地址
* nelts:    数组已分配元素个数
* size:     数组元素的大小
* nalloc:   数组最大能分配元素个数，当nelts大于nalloc时扩容
* pool:     内存池

---

相关函数
===

创建数组及其初始化
---

```c
ngx_array_t *
ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size)
{
    ngx_array_t *a;
    
    //通过内存池分配ngx_array_t结构的指针
    a = ngx_palloc(p, sizeof(ngx_array_t));
    if (a == NULL) {
        return NULL;
    }
    
    //初始化
    if (ngx_array_init(a, p, n, size) != NGX_OK) {
        return NULL;
    }

    return a;
}

static ngx_inline ngx_int_t
ngx_array_init(ngx_array_t *array, ngx_pool_t *pool, ngx_uint_t n, size_t size)
{
    /*
     * set "array->nelts" before "array->elts", otherwise MSVC thinks
     * that "array->nelts" may be used without having been initialized
     */
    
    //初始化数据
    array->nelts = 0;
    array->size = size;
    array->nalloc = n;
    array->pool = pool;
    
    //分配ngx_array_t结构指向的数组指针，大小为n*size
    array->elts = ngx_palloc(pool, n * size);
    if (array->elts == NULL) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

添加数组元素
---

```
//添加一个数组元素，并返回新添加元素的指针
void *
ngx_array_push(ngx_array_t *a)
{
    void        *elt, *new;
    size_t       size;
    ngx_pool_t  *p;
    

    //当数组已满时
    if (a->nelts == a->nalloc) {

        /* the array is full */

        size = a->size * a->nalloc;

        p = a->pool;
        
        //如果内存池last指针指向数组末尾，且当前内存池节点还能分配一个数组元素大小①
        if ((u_char *) a->elts + size == p->d.last
            && p->d.last + a->size <= p->d.end)
        {
            /*
             * the array allocation is the last in the pool
             * and there is space for new allocation
             */

            p->d.last += a->size;
            a->nalloc++;

        } else {//否则重新分配一个原来数组大小2倍的空间②
            /* allocate a new array */

            new = ngx_palloc(p, 2 * size);
            if (new == NULL) {
                return NULL;
            }

            ngx_memcpy(new, a->elts, size);
            a->elts = new;
            a->nalloc *= 2;
        }
    }

    elt = (u_char *) a->elts + a->size * a->nelts;
    a->nelts++;

    return elt;
}
```

* ①:为什么要有条件是:内存池last指针指向数组末尾？因为如果内存池分配了数组后，又为其他数据分配了空间，
    会造成数组空间不连续，导致访问出错，适用于不断push元素到数组的情况
* ②:直接重新从内存池分配2倍原来数组大小的数组，注意到原来旧的数组并没有释放，所以可能会有内存的浪费

**但是**，通过前面对内存池的学习，我们会发现if的情况永远无法满足，因为内存池分配的空间全部在ngx_pool_large的链表节点里，
而本身的链表节点只存储ngx_pool_large_t的结构体，所以结论是，当数组空间不够时，一定会重新分配2倍大小的空间


```c
//添加n个数组元素
void *
ngx_array_push_n(ngx_array_t *a, ngx_uint_t n)
{
    void        *elt, *new;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *p;

    size = n * a->size;

    if (a->nelts + n > a->nalloc) {

        /* the array is full */

        p = a->pool;

        if ((u_char *) a->elts + a->size * a->nalloc == p->d.last
            && p->d.last + size <= p->d.end)
        {
            /*
             * the array allocation is the last in the pool
             * and there is space for new allocation
             */

            p->d.last += size;
            a->nalloc += n;

        } else {
            /* allocate a new array */

            nalloc = 2 * ((n >= a->nalloc) ? n : a->nalloc);

            new = ngx_palloc(p, nalloc * a->size);
            if (new == NULL) {
                return NULL;
            }

            ngx_memcpy(new, a->elts, a->nelts * a->size);
            a->elts = new;
            a->nalloc = nalloc;
        }
    }

    elt = (u_char *) a->elts + a->size * a->nelts;
    a->nelts += n;

    return elt;
}
```
同ngx_array_push()类似



动态数组的销毁
---

按照nginx的设计，内存的分配和销毁统一由内存池管理，但是这里却有接口来做销毁的操作，
带着这样的疑问来看这个接口

```c
void
ngx_array_destroy(ngx_array_t *a)
{
    ngx_pool_t  *p;

    p = a->pool;
    
    //如果内存池last指针指向数组末尾，则通过移动内存池指针的方法，来销毁分配的数组元素
    if ((u_char *) a->elts + a->size * a->nalloc == p->d.last) {
        p->d.last -= a->size * a->nalloc;
    }
    
    //继续销毁存储的结构体
    if ((u_char *) a + sizeof(ngx_array_t) == p->d.last) {
        p->d.last = (u_char *) a;
    }
}
```
想了很久作者写这个接口的目的，可能是因为ngx_push_array时，一旦发生扩容时，就会重新分配2倍大小的数组，而原来数组没有被释放造成了空间的浪费，
所以作者增加这个接口来手动释放数组

**但是**，再复述遍前面说的内容：通过前面对内存池的学习，我们会发现接口中if的情况永远无法满足，因为内存池分配的空间全部在ngx_pool_large的链表节点里，
而本身的链表节点只存储ngx_pool_large_t的结构体，所以结论是，此接口无用，因为条件永远无法满足。并且搜索nginx整个源码，也没有发现此接口的调用。




结语：
===

ngx_array_t是一个能动态扩容的数组结构，由内存池统一管理分配的内存，效率高。
对源码的解读可能也有不对的地方，欢迎大家能够帮助发现和提出，一起进步


测试代码
===

[见github](https://github.com/seansun32/nginx_leanring_practice_code/tree/master/test_ngx_array)

