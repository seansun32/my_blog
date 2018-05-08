---
layout: post
title:  "python+numpy实现卷积神经网络2"
date:   2018-04-21 15:18:29 +0800
categories: 机器学习
tag: DeepLearning学习
---


* content
{:toc}

本节会实现卷积网络中，卷积层，池化层的前向传播和反向传播

这里的实现采取的是多层loop，优点是比较直观容易理解，缺点是计算量非常大(计算卷积是非常缓慢)
还有一种快速卷积的方法。卷积的操作其实就是参数共享+舍弃某些连接的全连接操作，所以可以将
点乘通过一定变换转换成矩阵相乘的方法，这里不做实现。主要思想是把image转换为列向量，filter转换为行向量，
image和filter的点乘，即为列向量与行向量的矩阵相乘

参见图例：
![test img](../../../../styles/images/deep_learning/2/fastconv.jpg)


我会在github中上传相应的方法。

卷积层的前向传播:
===

```python
def conv_forward(x,w,b,conv_param):
    '''
    x--(N,C,H,W)
    w--(F,C,HH,WW)
    b--(F,)
    out--(N,F,H_out,W_out)
    '''
    
    N,C,H,W=x.shape
    F,_,HH,WW=w.shape
    stride=conv_param.get('stride') 
    pad=conv_param.get('pad')
    
    H_out=1+(H-HH+2*pad)/stride
    W_out=1+(W-WW+2*pad)/stride

    x_pad=np.pad(x,((0,0),(0,0),(pad,pad),(pad,pad)),'constant',constant_values=0)
    out=np.zeros(N,F,H_out,W_out)

    for n in range(N):
        for f in range(F):
            for i in range(H_out):
                for j in range(W_out):
                    out[n,f,i,j]=np.sum(x_pad[n,:,i*stride:i*stride+HH,j*stride:j*stride+WW]*w[f,:,:,:])+b[f]
            
    
    cache=x,w,b,conv_param
    return out,cache
```

卷基层的反向传播:
===

```python
def conv_backward(dout,cache):
    '''
    dout--(N,F,H_out,W_out)
    '''

    x,w,b,conv_param=cache
    N,C,H,W=x.shape
    F,_,HH,WW=w.shape
    stride=conv_param.get('stride') 
    pad=conv_param.get('pad')
    
    H_out=1+(H-HH+2*pad)/stride
    W_out=1+(W-WW+2*pad)/stride

    x_pad=np.pad(x,((0,0),(0,0),(pad,pad),(pad,pad)),'constant',constant_values=0)
    dx_pad=np.zeros(x_pad.shape)
    dw=np.zeros(w.shape)
    db=np.zeros(b.shape)

    for n in range(N):
        for f in range(F):
            for i in range(H_out):
                for j in range(W_out):
                    dx_pad[n,:,i*stride:i*stride+HH,j*stride:j*stride+WW]+=dout[n,f,i,j]*w[f,:,:,:]
                    dw[f,:,:,:]+=dout[n,f,i,j]*x_pad[n,:,i*stride:i*stride+HH,j*stride:j*stride+WW]
                    db[f]+=dout[n,f,j,k]
    
    dx=dx_pad[:,:,pad:pad+H,pad:pad+W]

    return dx,dw,db
```

池化层的正向传播:
===

```python
def maxpool_forward(x,pool_param):
    '''
    x--(N,C,H,W)
    '''
    
    N,C,H,W=x.shape
    stride=pool_param.get('stride',2)
    HH=pool_param.get('pool_height',2)
    WW=pool_param.get('pool_width',2)

    H_out=1+(H-HH)//stride
    W_out=1+(W_WW)//stride

    out=np.zeros(N,C,H_out,W_out)
    
    for n in range(N):
        for c in range(C):
            for i in range(H_out):
                for j in range(W_out):
                    out[n,c,i,j]=np.max(x[n,c,i*stride:i*stride+HH,j*stride:j*stride+WW])

    cache=x,pool_param

    return out,cache

```

池化层的反向传播:
===

```python
def maxpool_backward(dout,cache):
    '''
    dout--(N,C,H_out,W_out)
    '''
    x,pool_param=cache

    N,C,H,W=x.shape
    stride=pool_param.get('stride',2)
    HH=pool_param.get('pool_height',2)
    WW=pool_param.get('pool_width',2)

    H_out=1+(H-HH)//stride
    W_out=1+(W_WW)//stride
    
    dx=np.zeros(x.shape)

    for n in range(N):
        for c in range(C):
            for i in range(H_out):
                for j in range(W_out):
                    ind_H,ind_W=np.unravel_index(np.argmax(x[n,c,i*stride:i*stride+HH,j*stride:j*stride+WW]),(HH,WW))
                    dx[n,c,i*stride+ind_H,j*stride+ind_W]=dout[n,c,i,j]

    return dx
```

