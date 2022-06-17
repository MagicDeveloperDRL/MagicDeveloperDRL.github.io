---

layout:     post
title:      「环境配置」在Pytorch上进行单机多GPU设置
subtitle:   Windows系统
date:       2022-01-04
author:     MRL Liu
header-img: img/the-first.png
catalog: True
tags: [环境配置]
   
---

​		训练较大的深度学习模型往往需要多个GPU设备，大部分实验室都使用`单机多GPU设置`，本文讲述在Windows系统上如何进行Pytorch平台上的单机多GPU设置。

## 一、相关函数

​		如何在单机设备上实现多个GPU并行计算，PyTorch提供了两个函数可实现简单高效的并行GPU计算：

```python
# 函数一
nn.parallel.data_parallel(module, inputs, device_ids=None, output_device=None, dim=0, module_kwargs=None)
# 函数二
nn.DataParallel(module, device_ids=None, output_device=None, dim=0)
```
​		二者的参数十分相似，通过device_ids参数可以指定在哪些GPU上进行优化，output_device指定输出到哪个GPU上。

​		不同点在于前者直接利用多GPU并行计算得出结果，而后者则返回一个新的模型，能够自动在多GPU上进行并行加速。

​		常用的是函数二，DataParallel()将模型相关的所有数据以浅复制的形式复制多份（输入一个batch的数据均分成多份），分别送到对应的GPU进行计算，然后将各个GPU计算得到的梯度累加得到结果。

## 二、代码演示

### 1、启用CUDA设备

```python
import os
device_ids = [0,1,2] # 设置当前可被Python检测到的显卡ID
os.environ["CUDA_VISIBLE_DEVICES"] = ','.join(map(str, device_ids))
```

### 2、分配网络模型

```python
# 方式 1
output = nn.parallel.data_parallel(model, input, device_ids=[0, 1])
# 方式 2
model = model.cuda()
model = nn.DataParallel(model, device_ids=[0,1,2])
output = model(input)
```

注意，`os.environ["CUDA_VISIBLE_DEVICES"]= "1, 2"` 会规定使用后两块显卡，但会将原本编号为1的显卡序号更改为0，原来序号为2的显卡更改为1，所以在nn.DataParallel()函数里device_ids序号不需要更改，必须写0,1。

## 三、实践问题

作者在实际的修改中，出现过如下问题：

#### 1、AssertionError: Invalid device id

原因是DataParallel()函数输入了无法使用的cuda设备ID，解决步骤：

（1）检查当前平台是否支持CUDA训练

（2）检查当前机器是否可访问多个GPU

（3）检查DataParallel()函数调用前是否进行过多余的CUDA设置操作

#### 2、UserWarning: PyTorch is not compiled with NCCL support

该警告表明当前PyTorch不支持NCCL编译，经过查询可能与Windows系统有关。
