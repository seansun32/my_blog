---
layout: post
title:  "Pandas学习笔记2"
date:   2018-03-06 16:48:29 +0800
categories: 机器学习
tag: Pandas学习
---

* content
{:toc}


这里学习pandas从csv文件导入导出数据



源码展示
===

```python

import pandas as pd
import numpy as np

#将csv读入df，以dataframe形式存储
df=pd.read_csv("https://storage.googleapis.com/mledu-datasets/california_housing_train.csv")
#显示dataframe的前5个记录
print(df.head())

#以逗号作为分隔符来读取csv数据
df=pd.read_csv("https://storage.googleapis.com/mledu-datasets/california_housing_train.csv",sep=",")
print(df.head())

#将dataframe导出，写在ca_house_train2.csv文件中
df.to_csv("ca_house_train2.csv")

#index_col=3,以dataframe的第三列作为index列
df=pd.read_csv('ca_house_train2.csv',index_col=3)
print(df.head())

#将dataframe每一列的name分别替换为1,2,...,9
df.columns=['1','2','3','4','5','6','7','8','9']
print(df.head())


#除去每一列的名字后，导出到文件Tch4.csv
df.to_csv('Tch4.csv',header=False)

#每一列赋予名字A，B，... I,读入到df
df=pd.read_csv('Tch4.csv',names=['A','B','C','D','E','F','G','H','I'],index_col=0)


#将df转化为html文件
df.to_html('test.html')

```
