---
layout:     post
title:      "pthread problem 探寻"
date:       2017-12-01 10:27:52 +0800
categories: 经验总结
tag:        问题探究 
---


*content
{: toc}




```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>



static void* thread_runner(void* i){
    int* j=(int*)i;
    while(1){
        if(*j==1){
            printf("aaaaaa\n"); 
        }else{
            printf("bbbbbb\n"); 
        } 
        sleep(3);
    }
}


int main(){
    pthread_t threadid[2];
    int a=1,b=2;
    printf("111111\n");
    pthread_create(&threadid[0],NULL,thread_runner,&a);
    pthread_create(&threadid[1],NULL,thread_runner,&b);

    pthread_join(threadid[0],NULL);
    pthread_join(threadid[1],NULL);

    printf("222222\n");
    return 0;
}

```

必须要加上pthread_join,thread线程才会执行，需要花时间探究thread机制，以及fork的问题
