---
title: "【论文笔记】Optimus论文阅读笔记"
description: 
date: 2022-11-15T15:06:14+08:00
image: https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202211151515591.png
math: 
license: 
hidden: false
comments: true
draft: true
categories: 
    - 论文
tags: 
    - 论文
    - Optimus
---

# Optimus 论文阅读笔记

## Abstract

- Optimus
  - 一个为深度学习集群定制的作业调度器，它基于在线资源性能模型使作业训练时间最小化。
  - Optimus使用在线拟合来预测训练期间的模型收敛，并设置性能模型来准确估计训练速度，作为每个作业分配资源的函数。
  - 基于这些模型，我们设计了一个简单而有效的方法，用于动态分配资源和放置深度学习任务，以尽量减少作业完成时间。
  - 实验
    - 我们在Kubernetes（一个用于容器编排的集群管理器）之上实现了Optimus
    - 并在一个有7个CPU服务器和6个GPU服务器的深度学习集群上进行了实验
    - 使用MXNet框架运行9个训练作业
    - 结果显示，Optimus在作业完成时间和makespan方面分别比有代表性的集群调度器高出约139%和63%