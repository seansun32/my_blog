---
layout:     post
title:      "Nginx源码学习2---数据结构:ngx_queue_t"
date:       2017-07-06 13:59:45 +0800
categories: 源码学习
tag:        Nginx系列
---

* content
{: toc}

前言：
===

今天学习了ngx_queue_t的数据结构及其相关部分的源码。总的来说，
ngx_queue_t就是一个**双向循环链表**,并且是**轻量级**，这种链表的写法值得借鉴，
其具体实现类似于linux kernel里的list.h。下面就来具体学习。

---

数据结构
===

ngx_queue_t结构
---

```c
struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};

typedef struct ngx_queue_s  ngx_queue_t;
```

从数据结构可以看到，其结构只包含指向前、后node的指针，并不包含数据区，这就是所谓的**轻量级**，
将数据区和链表分离，对于链表只用关心它本身的相关操作，并不需要关心节点具体的数据。所以这种链表
的节点的数据区可以关联任何形式的数据结构。

比如有两个不同的数据结构

```c
struct int_s{
    int v;
};

struct str_s{
    char* s;
};
```
我们只需将数据结构稍加改变：

```c
struct int_s{
    int v;
    ngx_queue_t queue;//链表节点
};

struct int_s{
    char* s;
    ngx_queue_t queue;//链表节点
};
```
实例化这两个数据结构，调用相关接口就能实现对链表的添加，删除等操作

---

相关函数
===

在nginx的源码中，使用宏定义即完成了链表的相关接口，非常类似于linux kernel中链表的实现。

```c

//初始化链表
#define ngx_queue_init(q)                                                     \
    (q)->prev = q;                                                            \
    (q)->next = q


//判断链表是否为空
#define ngx_queue_empty(h)                                                    \
    (h == (h)->prev)


//将节点加入到链表头,注意到该链表为双向循环链表，且包含哨兵节点。所以哨兵节点的next节点，即为链表头节点
#define ngx_queue_insert_head(h, x)                                           \
    (x)->next = (h)->next;                                                    \
    (x)->next->prev = x;                                                      \
    (x)->prev = h;                                                            \
    (h)->next = x


#define ngx_queue_insert_after   ngx_queue_insert_head


//将节点加入到链表尾，注意到该链表为双向循环链表，且包含哨兵节点。所以哨兵节点的prev节点，即为链表尾节点
#define ngx_queue_insert_tail(h, x)                                           \
    (x)->prev = (h)->prev;                                                    \
    (x)->prev->next = x;                                                      \
    (x)->next = h;                                                            \
    (h)->prev = x


//返回链表头节点，即哨兵节点的next节点
#define ngx_queue_head(h)                                                     \
    (h)->next


//返回链表尾节点，即哨兵节点的prev节点
#define ngx_queue_last(h)                                                     \
    (h)->prev


//返回链表的哨兵节点（无实际意义，可作为链表遍历的终止判定，见后面示例）
#define ngx_queue_sentinel(h)                                                 \
    (h)


//返回当前节点的next节点
#define ngx_queue_next(q)                                                     \
    (q)->next


//返回当前节点的prev节点
#define ngx_queue_prev(q)                                                     \
    (q)->prev



//删除当前节点
#define ngx_queue_remove(x)                                                   \
    (x)->next->prev = (x)->prev;                                              \
    (x)->prev->next = (x)->next
```

```c
//分隔链表:将链表分为两个双向循环链表，一个为节点q到到链表为节点(n为哨兵节点)，另一个为链表头节点(原哨兵节点)到节点q的prev节点
#define ngx_queue_split(h, q, n)                                              \
    (n)->prev = (h)->prev;                                                    \
    (n)->prev->next = n;                                                      \
    (n)->next = q;                                                            \
    (h)->prev = (q)->prev;                                                    \
    (h)->prev->next = h;                                                      \
    (q)->prev = n;
```
逻辑见下图
[图片来源](http://blog.csdn.net/livelylittlefish/article/details/6607324)

![ngx pic](../../../../styles/images/nginx/nginx2/ngx_2_1.jpeg)


```c
//合并链表:将哨兵节点分别为h和n的两个双向循环链表合并，并保留哨兵节点h，舍弃哨兵节点n
#define ngx_queue_add(h, n)                                                   \
    (h)->prev->next = (n)->next;                                              \
    (n)->next->prev = (h)->prev;                                              \
    (h)->prev = (n)->prev;                                                    \
    (h)->prev->next = h;


#define ngx_queue_data(q, type, link)                                         \
    (type *) ((u_char *) q - offsetof(type, link))

```
逻辑见下图
[图片来源](http://blog.csdn.net/livelylittlefish/article/details/6607324)

![ngx pic](../../../../styles/images/nginx/nginx2/ngx_2_2.jpeg)

---

测试用例
===

下面通过例子来说明ngx_queue_t的用法

```c
int main(){
    test_ngx_queue_t queue_g;
    test_ngx_queue_t *q, *prev, *next, tail; 

    int i=0;
    test_ngx_queue_node_t *tmp_node=NULL;

    //init queue
    test_ngx_queue_init(&queue_g);

    
    //set value to queue node, and add node to queue
    for(i=0;i<10;i++){
        tmp_node=calloc(1,sizeof(test_ngx_queue_node_t));
        tmp_node->val=i;
        
        if(i%2==0){
            test_ngx_queue_insert_head(&queue_g,&tmp_node->queue);
        }else{
            test_ngx_queue_insert_tail(&queue_g,&tmp_node->queue); 
        }
        tmp_node=NULL;
    }

    q=test_ngx_queue_head(&queue_g);
    if(q==test_ngx_queue_last(&queue_g)){
        printf("no node\n");
        return -1;
    }


    //traverse queue, delete node with value 5
    for(q=test_ngx_queue_head(&queue_g);q!=test_ngx_queue_sentinel(&queue_g);q=next){
        //prev=test_ngx_queue_prev(q);
        next=test_ngx_queue_next(q);
        tmp_node=test_ngx_queue_data(q,test_ngx_queue_node_t,queue);
        if(tmp_node->val==5){
            test_ngx_queue_remove(&tmp_node->queue);
            free(tmp_node);
        }
    }

    
    //traverse queue, print value of each node
    for(q=test_ngx_queue_head(&queue_g);q!=test_ngx_queue_sentinel(&queue_g);q=next){
        next=test_ngx_queue_next(q);
        tmp_node=test_ngx_queue_data(q,test_ngx_queue_node_t,queue);
        printf("node value:%d\n",tmp_node->val);
    }


    //split queue at value=4
    for(q=test_ngx_queue_head(&queue_g);q!=test_ngx_queue_sentinel(&queue_g);q=next){
        next=test_ngx_queue_next(q);
        tmp_node=test_ngx_queue_data(q,test_ngx_queue_node_t,queue);
        if(tmp_node->val==4){
            test_ngx_queue_split(&queue_g,&tmp_node->queue,&tail);//tail is the new sentinel node for new splited list 
            break;
        }
    }
    for(q=test_ngx_queue_head(&queue_g);q!=test_ngx_queue_sentinel(&queue_g);q=next){
        next=test_ngx_queue_next(q);
        tmp_node=test_ngx_queue_data(q,test_ngx_queue_node_t,queue);
        printf("split-1 node value:%d\n",tmp_node->val); 
    }
    for(q=test_ngx_queue_head(&tail);q!=test_ngx_queue_sentinel(&tail);q=next){
        next=test_ngx_queue_next(q);
        tmp_node=test_ngx_queue_data(q,test_ngx_queue_node_t,queue);
        printf("split-2 node value:%d\n",tmp_node->val); 
    }


    //add queue of these two split queue
    test_ngx_queue_add(&queue_g,&tail);
    for(q=test_ngx_queue_head(&queue_g);q!=test_ngx_queue_sentinel(&queue_g);q=next){
        next=test_ngx_queue_next(q);
        tmp_node=test_ngx_queue_data(q,test_ngx_queue_node_t,queue);
        printf("after add node value:%d\n",tmp_node->val); 
    }
     

    //destroy queue
    for(q=test_ngx_queue_head(&queue_g);q!=test_ngx_queue_sentinel(&queue_g);q=next){
        next=test_ngx_queue_next(q);
        tmp_node=test_ngx_queue_data(q,test_ngx_queue_node_t,queue);
        test_ngx_queue_remove(&tmp_node->queue);
        free(tmp_node);
    }
    if(test_ngx_queue_empty(&queue_g)){
        printf("queue is empty now\n"); 
    }else{
        printf("queue is not empty\n"); 
    }

    return 0;

}
```
---

参考资料
===

[nginx源码分析—队列结构ngx_queue_t](http://blog.csdn.net/livelylittlefish/article/details/6607324)
