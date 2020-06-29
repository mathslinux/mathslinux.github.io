---
layout: post
tag: Machine_Learning
date: '\[2017-02-27 一 16:50\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: TensorFlow 杂记
---

初体验
======

基本概念
--------

TensorFlow 是一个编程系统, 使用图来表示计算任务. 图中的节点被称之为 op
(operation 的缩写). 一个 op 获得 0 个或多个 Tensor, 执行计算, 产生 0
个或多个 Tensor.

一个 TensorFlow 图描述了计算的过程. 为了进行计算, 图必须在会话里被启动.
会话将图的 op 分发到诸如 CPU 或 GPU 之类的 设备 上, 同时提供执行 op
的方法. 这些方法执行后, 产生的 tensor 返回.

Graph
:   表示计算任务, Graph 中的节点称为 `op(operation的缩写)`

Session
:   在 Session 中执行图

tensor
:   使用 tensor 表示数据, 每个 Tensor 是一个类型化的多维数组. 例如,
    可以将一小组图像集表示为一个四维浮点数数组, 这四个维度分别是
    \[batch, height, width, channels\]

Variable
:   用来维护状态

fetch
:   使用 fetch 为任意的操作从其中获取数据.

feed
:   使用 feed 为任意的操作 赋值

运行机制: TensorFlow 程序通常被组织成一个构建阶段和一个执行阶段.
在构建阶段, op 的执行步骤 被描述成一个图. 在执行阶段,
使用会话执行执行图中的 op.

构建阶段

:   第一步, 是创建源 op (source op). 源 op 不需要任何输入 （e.g. 常量
    onstant). 源 op 的输出被传递给其它 op 做运算. TensorFlow Python
    库有一个默认图 (default graph), op 构造器可以为其增加节点.

    ``` python
    import tensorflow as tf
    # 创建一个常量 op(1x2 矩阵). 这个 op 被作为一个节点
    # 加到默认图中.
    matrix1 = tf.constant([[3., 3.]])
    # 创建另外一个常量 op(一个 2x1 矩阵)
    matrix2 = tf.constant([[2.],[2.]])
    # 创建一个矩阵乘法 matmul op , 'matrix1' 和 'matrix2' 作为输入.
    # 返回值 'product' 代表矩阵乘法的结果.
    product = tf.matmul(matrix1, matrix2)
    ```

执行阶段

:   在执行阶段, 使用会话执行执行图中的 op.

    ``` python
    # 启动默认图.
    sess = tf.Session()

    # 调用 sess::run() 执行矩阵乘法 op, 传入 'product' 作为该方法的参数. 
    # 函数调用 'run(product)' 触发了图中三个 op (两个常量 op 和一个矩阵乘法 op) 的执行.

    result = sess.run(product)
    print result # 得到一个 1x1 的矩阵 [[ 12.]]

    sess.close()
    ```

变量

:   维护图执行过程中的状态信息.

    ``` python
    # -*- coding: utf-8 -*-
    import tensorflow as tf

    # 创建一个变量, 初始化为标量 0.
    state = tf.Variable(0, name="counter")

    # 创建一个 op, 用来使 state 增加 1
    one = tf.constant(1)
    new_value = tf.add(state, one)
    update = tf.assign(state, new_value)

    # 启动图后, 变量必须先经过 *初始化* (init) op 初始化,
    # 所以必须得先加一个 *初始化* op 到图中.
    init_op = tf.initialize_all_variables()

    with tf.Session() as sess:
        # 运行 'init' op
        sess.run(init_op)
        # 打印 'state' 的初始值
        print sess.run(state)
        # 运行 op, 更新 'state', 并打印 'state'
        for _ in range(3):
            sess.run(update)
            print sess.run(state)
    # 输出:
    # 0
    # 1
    # 2
    # 3
    ```

一个例子
--------

``` python
# -*- coding: utf-8 -*-

import tensorflow as tf
import numpy as np

# 模拟生成100对数据对, 对应的函数为y = x * 0.1 + 0.3
x_data = np.random.rand(100).astype("float32")
y_data = x_data * 0.1 + 0.3

# 指定w和b变量的取值范围
W = tf.Variable(tf.random_uniform([1], -1.0, 1.0))
b = tf.Variable(tf.zeros([1]))
y = W * x_data + b

# 最小化均方误差
loss = tf.reduce_mean(tf.square(y - y_data))
optimizer = tf.train.GradientDescentOptimizer(0.5)
train = optimizer.minimize(loss)

# 初始化TensorFlow参数
init = tf.global_variables_initializer()

# 运行数据流图（注意在这一步才开始执行计算过程）
sess = tf.Session()
sess.run(init)

# 训练 200次，每 20 次观察一下w和b的拟合值和误差
for step in xrange(201):
    sess.run(train)
    if step % 20 == 0:
        print step, sess.run(W), sess.run(b), sess.run(loss)
```

查看运行结果

``` python
(tensorflow)$ python tf.py
0 [-0.24480659] [ 0.61340892] 0.033027
20 [-0.00973047] [ 0.35396883] 0.000939542
40 [ 0.07003028] [ 0.31474003] 7.00853e-05
60 [ 0.09181464] [ 0.30402583] 5.22802e-06
80 [ 0.0977644] [ 0.30109954] 3.89987e-07
100 [ 0.09938942] [ 0.3003003] 2.90904e-08
120 [ 0.09983326] [ 0.30008203] 2.16961e-09
140 [ 0.09995446] [ 0.30002242] 1.61868e-10
160 [ 0.09998757] [ 0.30000612] 1.20645e-11
180 [ 0.0999966] [ 0.30000168] 9.0381e-13
200 [ 0.09999908] [ 0.30000046] 6.62848e-14
```
