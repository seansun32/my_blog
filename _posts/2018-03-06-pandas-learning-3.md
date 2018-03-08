---
layout: post
title:  "Pandas学习笔记3"
date:   2018-03-06 17:28:29 +0800
categories: 机器学习
tag: Pandas学习
---

* content
{:toc}



添加，删除dataframe的相关方法


源码展示
===

```pythone
import pandas as pd

df1 = pd.DataFrame({'HPI':[80,85,88,85],
                    'Int_rate':[2, 3, 2, 2],
                    'US_GDP_Thousands':[50, 55, 65, 55]},
                   index = [2001, 2002, 2003, 2004])

df2 = pd.DataFrame({'HPI':[80,85,88,85],
                    'Int_rate':[2, 3, 2, 2],
                    'US_GDP_Thousands':[50, 55, 65, 55]},
                   index = [2005, 2006, 2007, 2008])

df3 = pd.DataFrame({'HPI':[80,85,88,85],
                    'Int_rate':[2, 3, 2, 2],
                    'Low_tier_HPI':[50, 52, 50, 53]},
                   index = [2001, 2002, 2003, 2004])


concat=pd.concat([df1,df2,df3])
print(concat)


```
结果如图,即将不重合的添加在dataframe尾部
![test img](../../../../styles/images/pandas_learning/3/1.png)

```python
df4=df1.append(df3)
print(df4)
```
结果如图
![test img](../../../../styles/images/pandas_learning/3/2.png)

```python
#新建一行，加入到df1的dataframe中
s=pd.Series([80,2,50],index=['HPI','Int_rate','US_GDP_Thousands'])
df4=df1.append(s,ignore_index=True)
print(df4)
```

PS. `concat`通常来说比`append`更快


```python
#以HPI列为公共列，进行合并
df4=pd.merge(df1,df2,on='HPI')
print(df4)
```
![test img](../../../../styles/images/pandas_learning/3/3.png)


```python
df1.set_index('HPI',inplace=True)
df3.set_index('HPI',inplace=True)
#因为除了HPI列，还有相同的Int_rate所以合并会失败
joined=df1.join(df3)
print(joined)
```



```python
df1 = pd.DataFrame({'Year':[2001,2002,2003,2004],
                    'Int_rate':[2, 3, 2, 2],
                    'US_GDP_Thousands':[50, 55, 65, 55]})


df3 = pd.DataFrame({'Year':[2001,2003,2004,2005],
                    'Unemployment':[7, 8, 9, 6],
                    'Low_tier_HPI':[50, 52, 50, 53]})

#以左边df1的Year为基准进行合并
merged=pd.merge(df1,df3,on='Year',how='left')
merged.set_index('Year',inplace=True)
print(merged)
```
![test img](../../../../styles/images/pandas_learning/3/4.png)

```python
#默认是以全部都不漏的方式合并
merged=pd.merge(df1,df3,on='Year',how='outer')
merged.set_index('Year',inplace=True)
print(merged)
```
![test img](../../../../styles/images/pandas_learning/3/5.png)

