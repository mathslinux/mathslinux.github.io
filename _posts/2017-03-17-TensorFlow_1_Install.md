---
layout: post
tag: Machine_Learning
date: '\[2017-03-17 五 15:50\]'
setupfile: '\~/Dropbox/Doc/Org\_Templates/level-1.org'
title: 'TensorFlow Note (1) - 环境搭建'
---

由于 tensorflow 采用 python 开发，是一个 python 的 library, 因此借助于
pip, 在各个平台上安装 tensorflow 都非常方便, 不过要想取得更高的性能,
需要安装 gpu 的版本(包括安装 CUDA 相关包等).

安装 CUDA
=========

[CUDA](https://en.wikipedia.org/wiki/CUDA) 是 NVIDIA
提供的并行计算平台和编程模型, 应用程序可以利用 CUDA, 进行
高性能的并行计算.

[CUDNN](https://developer.nvidia.com/cudnn) (DNN means Deep Neural
Network) 是 Nvidia 基于 GPU 开发的一套深度学习 库.

目前, 官方提供了 Linux, MacOS, Windows 平台的各种格式的 CUDA 安装包,
这里的 安装以 ubuntu 16.04为例.

验证支持
--------

到 <https://developer.nvidia.com/cuda-gpus> 下查看显卡是否支持CUDA.

安装 CUDA
---------

到 [CUDA 的下载页面](https://developer.nvidia.com/cuda-downloads)
下载相应的格式(这里为 deb 格式), 然后安装:

``` bash
$ sudo dpkg -i cuda-repo-ubuntu1604-8-0-local_8.0.44-1_amd64.deb 
$ sudo apt-get update
$ sudo apt-get install cuda
```

安装 CUDNN
----------

到 [CUDNN 的下载页面](https://developer.nvidia.com/cudnn)
注册并下载(一般下载最新版本的):

``` bash
$ tar xvzf cudnn-8.0-linux-x64-v5.1.tgz 
$ sudo cp -P cuda/include/cudnn.h /usr/local/cuda/include/
$ sudo cp -P cuda/lib64/libcudnn* /usr/local/cuda/lib64
$ sudo chmod a+r /usr/local/cuda/include/cudnn.h /usr/local/cuda/lib64/libcudnn*
```

安装 TensorFlow
===============

使用 python 的虚拟环境来安装 tensorflow:

``` bash
$ virtualenv --system-site-packages ~/Develop/ML/tensorflow
$ source ~/Develop/ML/tensorflow/bin/activate
$ pip install tensorflow-gpu
```

验证:

``` bash
>>> import tensorflow as tf
>>> hello = tf.constant('Hello, TensorFlow!'); sess = tf.Session(); print(sess.run(hello))
Hello, TensorFlow!
>>> a = tf.constant(10); b = tf.constant(32); print(sess.run(a + b))
42
```
