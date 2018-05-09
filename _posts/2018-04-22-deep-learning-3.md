---
layout: post
title:  "python+numpy实现卷积神经网络3"
date:   2018-04-22 15:18:29 +0800
categories: 机器学习
tag: DeepLearning学习
---


* content
{:toc}

本节会实现sgd,adam的参数优化方法，一个好的优化方法能够使
cnn更快的收敛，目前adam是最流行的方法之一。


sgd的实现：
===

```python
def sgd(w,dw,optim_config):
    learning_rate=optim_config.get('learning_rate',1e-3)
    w -= learning_rate*dw

    return w,optim_config
```



adam的实现：
===

```python
def adam(x,dx,config=None):
    if config is None: config={}
    config.setdefault('learning_rate',1e-3)
    config.setdefault('beta1',0.9)
    config.setdefault('beta2',0.999)
    config.setdefault('epsilon',1e-8)
    config.setdefault('m',np.zeros_like(x))
    config.setdefault('v',np.zeros_like(x))
    config.setdefault('t',0)
    
    next_x=None
    beta1,beta2,eps=config['beta1'],config['beta2'],config['epsilon']
    t,m,v=config['t'],config['m'],config[v]
    m=beta1*m+(1-beta1)*dx
    v=beta2*v+(1-beta2)*(dx*dx)
    t += 1
    alpha=config['learning_rate']*np.sqrt(1-beta2**t)/(1-beta1**t)
    x -= alpha*(m/(np.sqrt(v)+eps))
    config['t']=t
    config['m']=m
    config['v']=v

    next_x=x

    return next_x,config

```

