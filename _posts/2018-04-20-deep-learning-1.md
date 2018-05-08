---
layout: post
title:  "python+numpy实现卷积神经网络1"
date:   2018-04-20 15:18:29 +0800
categories: 机器学习
tag: DeepLearning学习
---



* content
{:toc}

目前正在学习deep-learning，卷积神经网络是深度学习中的一个重要概念，
卷积神经网络概念的提出大大提高了图像识别的精确率。我基于斯坦福大学cs231n
的课程，使用python+numpy从底层实现了卷积神经网络的框架，加深了对
卷积神经网络的框架，前向传播和反向传播，参数优化等概念的理解。

在此借助blog对我的学习做一个记录。

本篇我会实现全连接层的前向传播和反向传播，激活函数(relu)的前向传播和反向传播，
还有损失函数(softmax)的实现




代码分析：
===

全连接网络的前向传播:
---

```python
def fc_forward(x,w,b):
    ''' 
    dimension of all params
    x--(N,D1,D2,...Dk)
    w--(H,C)
    b--(C,)

    out--(N,C)
    '''

    x_reshape=np.reshape(x,(x.shape[0],-1)) 
    out=x_reshape.dot(w)+b

    #cache保存的数据会用于反向传播
    cache=(x,w,b)

    return out,cache
```

全连接网络的反向传播:
---

```python
def fc_backward(dout,cache):
    '''
    dout--(N,C)
    cache:=x,w,b
    '''

    x,w,b=cache
    x_reshape=np.reshape(x,(x.shape[0],-1))

    dx=dout.dot(w.T)
    dx=np.reshape(dx,x.shape)
    dw=x.T.dot(dout)
    db=np.sum(dout,axis=0)


    areturn dx,dw,db
```

根据链式法则，注意到这里的dout，如果是在output层，dout为损失函数对输入的导数
如果在中间层，则dout为后一层网络的dx

这里可以看到，dw由dout决定(相邻后一层网络的dx),而dx由dout和w决定

所以，当激活函数选为sigmoid时，dx可能会处于饱和而接近0，导致dw也会过小，造成梯度消失
当初始化的w取值过小或者过大，dx也会过小或过大，从而导致dw过小(造成梯度消失)或过大(造成梯度爆炸)


激活层的前向传播:
---

激活函数在网络中引入了非线性结构，使得神经网络能够学习非线性的数据
这里使用最流行的relu作为激活函数

```python
def relu_forward(x):
    mask=x>0
    out=x*mask
    cache=x,mask

    return out,cache
```

激活层的反向传播:
---

```python
def relu_backward(dout,cache):
   x,mask=cache
   dx=dout*mask

   return dx
```

损失函数:
---

这里采用softmax作为损失函数，softmax对于多分类问题是很好的选择

```
def softmax_loss(x,y):
    num_input=x.shape[0]
    x -= np.max(x,axis=1) #prevent overflow
    correct_x=x[np.arange(num_input),y]
    exp_x=np.exp(x)
    exp_correct=np.exp(correct_x)

    prob=exp_correct/np.sum(exp_x,axis=1)
    loss=np.sum(-1*np.log(prob),axis=0)
    loss /= num_input

    dout=np.zeros(x.shape)
    dout=exp_x/np.sum(exp_x,axis=1)
    dout[np.arange(num_input),y] -= 1
    dout /= num_input 
``` 

具体公式可以自己手动推导，通常通过维度检测来检查反向传播是否正确,
因为例如x和dx的维度一定是一样的


在下一节将实现卷积神经网络相关的前向传播和反向传播方法



