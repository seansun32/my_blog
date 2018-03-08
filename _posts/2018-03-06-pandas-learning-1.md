---
layout: post
title:  "Pandas学习笔记1"
date:   2018-03-06 15:18:29 +0800
categories: 机器学习
tag: Pandas学习
---

* content
{:toc}


pandas是一种列存数据分析api，它是用于处理和分析输入数据的强大工具。
很多机器学习框架都支持将pandas数据结构作为输入。

pandas中主要数据结构被实现为以下两类：
* DataFrame: 可以将它想像成一个关系型数据表格，其中包含多个行和已命名的列。
* Series: 它是单一列。DataFrame中包含一个或多个Series，每个Series均有一个名称。



代码分析：
===

pandas中dataframe的一些基本操作
---


```python

#导入pandas API
import pandas as pd
#导入numpy API
import numpy as np


#建立一个dict数据
web_stat={'Day':[1,2,3,4,5,6],
        'Visitors':[43,53,34,45,64,34],
        'Bounce_Rate':[65,72,62,64,54,66]}


#将web_stat转为dataframe
df=pd.DataFrame(web_stat)
print(df)

#将day的series作为index
print(df.set_index('Day'))
#没有设置inplace=True，所以不会修改原始数据
print(df)

#设置了inplace=True，修改了原始数据
df.set_index('Day',inplace=True)
print(df)

#获取visitor series的两种方式
print(df['Visitors'])
print(df.Visitors)

#获取该dataframe的两个series
print(df[['Bounce_Rate','Visitors']])

#将visitor的series转化为list
print(df.Visitors.tolist())

#将两列series转化为array
print(np.array(df[['Visitors','Bounce_Rate']]))

#将df的index重新排列，如果不含该index，则填入NaN
print(df.reindex([2,0,1]))


#创建Series
city_names=pd.Series(['San Francisco','San Jose','Sacramento'])
population=pd.Series([852469,1015785,485199])

#通过series来创建dataframe
cities=pd.DataFrame({'City name':city_names,'Population':population})


```

