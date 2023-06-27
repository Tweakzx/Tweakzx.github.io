---
title: "【论文笔记】MLsys Paper List"
author: "Tweakzx"
description: 
date: 2023-05-21T21:38:00+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories: 
tags: 
---

# **A reading list for machine learning systems**

## **Frameworks**

- **[VLDB '20] PyTorch Distributed: Experiences on Accelerating Data Parallel Training**
- **[NeurIPS '19] PyTorch: An Imperative Style, High-Performance Deep Learning Library**
- **[OSDI '18] Ray: A Distributed Framework for Emerging AI Applications**
- **[OSDI '16] TensorFlow: A System for Large-Scale Machine Learning**

## **Parallelism & Distributed Systems**

- **[NSDI '23] ARK: GPU-driven Code Execution for Distributed Deep Learning**
- **[OSDI '22] Unity: Accelerating DNN Training Through Joint Optimization of Algebraic Transformations and Parallelization**
- **[EuroSys '22] Varuna: Scalable, Low-cost Training of Massive Deep Learning Models**
- **[SC '21'] Chimera: Efficiently Training Large-Scale Neural Networks with Bidirectional Pipelines**
- **[ICML '21] PipeTransformer: Automated Elastic Pipelining for Distributed Training of Large-scale Models**
- **[OSDI '20] A Unified Architecture for Accelerating Distributed DNN Training in Heterogeneous GPU/CPU Clusters**
- **[ATC '20] HetPipe: Enabling Large DNN Training on (Whimpy) Heterogeneous GPU Clusters through Integration of Pipelined Model Parallelism and Data Parallelism**
- **[NeurIPS '19] GPipe: Efficient Training of Giant Neural Networks using Pipeline Parallelism**
- **[SOSP '19] A Generic Communication Scheduler for Distributed DNN Training Acceleration**
- **[SOSP '19] PipeDream: Generalized Pipeline Parallelism for DNN Training**
- **[EuroSys '19] Parallax: Sparsity-aware Data Parallel Training of Deep Neural Networks**
- **[arXiv '18] Horovod: fast and easy distributed deep learning in TensorFlow**
- **[ATC '17] Poseidon: An Efficient Communication Architecture for Distributed Deep Learning on GPU Clusters**
- **[EuroSys '16] STRADS: A Distributed Framework for Scheduled Model Parallel Machine Learning**
- **[EuroSys '16] GeePS: Scalable Deep Learning on Distributed GPUs with a GPU-specialized Parameter Server**
- **[OSDI '14] Scaling Distributed Machine Learning with the Parameter Server**
- **[NIPS '12] Large Scale Distributed Deep Networks**

## **GPU Cluster Management**

- **[NSDI'23] Transparent GPU Sharing in Container Clouds for Deep Learning Workloads**
  - 容器广泛用于数据中心中的资源管理。
    - 支持容器云中的深度学习(DL)训练的一个常见实践是静态地将 GPU 完全绑定到容器上。
    - 由于生产中 DL 作业的资源需求多种多样，大量的 GPU 未得到充分利用。
    - 因此，GPU 集群具有较低的 GPU 利用率，由于排队，导致作业完成时间较长。
  - 我们提出 TGS (透明 GPU 共享) ，一个系统，提供透明的 GPU 共享到集装箱云中的 DL 训练。
    - 与最近 GPU 共享的应用层解决方案形成鲜明对比的是，TGS 在容器下的 OS 层运行。
    - 透明性允许用户使用任何软件来开发模型并在其容器中运行作业。
    - TGS 利用自适应速率控制和透明统一内存，同时实现高 GPU 利用率和性能隔离。
    - 它确保生产作业不会受到共享 GPU 上机会作业的严重影响。
    - 我们已经建立了 TGS，并将他与docker和kubernetes整合。实验结果表明: 
      - TGS 对生产作业的吞吐量影响不大
      - TGS 为机会作业提供了与最先进的应用层解决方案 AntMan 相似的吞吐量，并且与现有的 OS 层解决方案 MPS 相比提高了15倍的吞吐量。
- **[ASPLOS '23]  Lucid: A Non-intrusive, Scalable and Interpretable Scheduler for Deep Learning Training Jobs**
  - 虽然最近的深度学习工作负载调度器表现出很好的性能，但在实际应用中很难部署它们，是由于
    - 缺乏灵活的入侵方式、
    - 过高的集成和维护成本
    - 有限的可伸缩性
    - 不透明的决策过程
  - Lucid: 基于可解释模型的非侵入式深度学习工作负载调度器 。
    - 它由三个创新模块组成。
      - 首先，为了有效地收集作业度量和及时调试作业反馈，引入了一个二维优化剖析器。
      - 其次，Lucid 利用一种惰性包装策略来规避干扰。
      - 第三，Lucid 基于估计的作业优先级值和共享分数来编排资源，以实现有效的调度。
    - 此外，Lucid 通过设计良好的系统优化器促进模型性能维护和系统透明调整。
    - 我们的评估表明，与最先进的抢占式调度器 Tiresias 相比，Lucid 将平均作业完成时间减少了1.3倍。此外，它为实际部署提供了明确的系统解释和优秀的可伸缩性。
- **[EuroSys '23] Lyra: Elastic Scheduling for Deep Learning Clusters**
    - 训练和推理中的问题: 
      - 当流量负载较低时，推理集群的 GPU 利用率较低
      - 由于缺乏资源，训练作业往往需要较长的排队时间
    - 我们引入了 Lyra ，一个新的集群调度器来解决这些问题。
        - Lyra 引入了容量贷款，将空闲推理 GPU 服务器贷款给训练工作。它进一步利用弹性扩展来扩展培训作业的 GPU 分配，以更好地利用借出的资源。
        - 容量借贷和弹性扩展为集群管理带来了新的挑战。
          - 当需要返回借出的服务器时，我们需要最小化作业抢占的数量
          - 当更多的 GPU 可用时，我们需要将它们分配到弹性作业，并最小化作业完成时间(JCT)
        - Lyra 使用基于原则的启发式方法来解决这些组合问题。
          - 它引入了服务器抢占成本的概念，并在服务器回收期间使用贪婪的方法降低这一成本。
          - 它进一步依赖于为弹性工作的每个额外工人定义的 JCT 缩减值，以多选择背包问题解决调度问题。
        - 在64-GPU 测试平台上的原型实现和超过50,000个生产作业的15天跟踪的大规模模拟表明
          - Aryl 在平均排队时间和 JCT 方面带来了1.53 x 和1.50 x 的减少
          - 集群调度器提高了高达26.9% 的集群使用率
- **[OSDI '22] Looking Beyond GPUs for DNN Scheduling on Multi-Tenant Clusters**
    - 此文作者发现，
      - 尽管 GPU 是 DNN 训练所需要的最主要的资源，
      - 但是 CPU 和 MEM 的分配策略同样也会显著影响到集群的资源利用效率和训练任务的吞吐量。
    - Synergy
        - 针对不同类型的 DNN，深入分析了 CPU 和 MEM 的不同组合对其吞吐量的影响
        - 然后将最优的组合方案应用到了调度策略中
- **[NSDI '22] MLaaS in the Wild: Workload Analysis and Scheduling in Large-Scale Heterogeneous GPU Clusters**
    - 随着机器学习 (ML) 技术的持续进步和最近大量数据集的可用性，科技公司正在部署大型 ML 即服务 (MLaaS) 云，通常使用异构 GPU 来提供大量 ML 应用程序。然而，在异构 GPU 集群中运行不同的 ML 工作负载会带来许多挑战。
    - 在本文中
      - 我们对从阿里巴巴拥有 6,000 多个 GPU 的生产 MLaaS 集群中收集的为期两个月的工作负载跟踪进行了表征研究。
      - 我们解释了集群调度所面临的挑战
        - GPU 利用率低
        - 排队延迟长
        - 需要高端 GPU 
        - 调度要求苛刻的任务难以调度
        - 异构机器之间的负载不平衡
        - 潜在的CPU瓶颈
      - 我们描述了我们当前的解决方案，并呼吁进一步调查仍未解决的挑战。
      - 我们已经发布了公共访问的跟踪，就工作负载和集群规模而言，这是最全面的。
- **[arXiv '22] Singularity: Planet-Scale, Preemptive and Elastic Scheduling of AI Workloads**
  - Singularity
    - 微软的全球分布式调度服务，高效和可靠地执行深度学习训练和推理工作负载
    - 核心：是一个新颖的、**工作负载感知的调度器**，它可以透明地抢占和弹性地扩展深度学习工作负载
      - 以提高利用率
      - 而且不会影响它们在全球 AI 加速器(GPU/FPGA)中的正确性或性能
    - 所有作业都是**可抢占的**、**可迁移的**，并且在默认可以动态调整大小(**弹性)** : 一个活动的作业可以被动态和透明地
      - 被抢占和被迁移到不同的节点集、集群、数据中心或者区域，并且可以从执行被抢占的地方精确地恢复
      - 在给定类型的一组不同的加速器上调整大小
    - 机制透明：不要求用户对代码进行任何更改，也不要求使用任何可能限制灵活性的自定义库。
    - 可靠：利用Singularity可以获得效率和可靠性增益，而对稳态性能的影响可以忽略不计。
    - 我们的设计方法是对DNN 网络架构不感知的，并且可以处理各种并行策略(数据/流水线/模型并行)
- **[OSDI '21] Pollux: Co-adaptive Cluster Scheduling for Goodput-Optimized Deep Learning**
    - Pollux通过在每个作业层面和整个集群层面自适应地共同优化相互依赖的因素，改善深度学习（DL）集群的调度性能。
        - 在训练过程中监测每个作业的状态，Pollux建模了他们的有效吞吐量随着添加或删除资源而发生的变化
        - Pollux动态地重新分配资源，以提高整个集群的好产量，同时尊重公平性，不断优化每个DL作业，以更好地利用这些资源。
      - 在真实的DL作业和跟踪驱动的模拟实验
        - Pollux相对于最先进的DL调度器将平均作业完成时间减少了37-50%
        - Pollux促进了竞争资源的DL作业之间的公平性
- **[NSDI '21] Elastic Resource Sharing for Distributed Deep Learning**

- **[OSDI '20] Heterogeneity-Aware Cluster Scheduling Policies for Deep Learning Workloads**
    - 背景
      - 加速器，如 GPU、 TPU、 FPGA 和定制 ASIC 表现出跨模型架构的异构性能行为。
      - 现有的针对加速器集群的调度器(用于在许多用户之间仲裁这些昂贵的培训资源)已经展示了如何针对各种多任务、多用户目标(如公平性和完成时间)进行优化。不幸的是，现有的调度程序基本上不考虑性能异构性。
    - 异构感知调度器 Gavel
        - 它系统地推广了大量**现有的调度策略**。Gavel 将这些策略表示为**优化问题**，然后使用我们称为有效吞吐量的抽象将这些问题系统地转换为能够识别异构性的版本。
        - 然后，Gavel 使用一种**基于循环的调度机制**，以确保在给定目标调度策略的情况下，作业能够得到理想的分配。
        - Gavel 的异质性感知策略允许异质集群维持更高的输入负载，并且与异质性不可知策略相比，最终目标如完成时间和平均工作完成时间分别提高了1.4倍和3.5倍。
- **[OSDI '20] AntMan: Dynamic Scaling on GPU Clusters for Deep Learning**
    - 如何在大规模GPU集群上有效调度深度学习工作， 对于**工作性能**，**系统吞吐量**和**硬件利用率**至关重要。
        - 随着深度学习的工作量变得更加复杂，它变得越来越具有挑战性。
    - 本文将介绍AntMan， 这是一种深入学习的基础设施，该基础架构共同设计了集群调度程序，并已在阿里巴巴部署在生产中，以管理数以万计的每日深度学习工作。
        - AntMan适应深度学习训练的波动资源需求。因此，它利用备用GPU资源在共享GPU上共同执行多个作业。
        - AntMan利用深度学习训练的独特特征，在深度学习框架内为显存和计算资源引入动态缩放机制。这允许job之间的细粒度协调并防止工作干扰。
        - 评估表明，AntMan在我们的多租户集群中不损害公平性的情况下，整体将显存利用率提高了42％，计算资源利用率提高了34％，为有效利用大规模的GPU提出了新方法。
- **[NSDI '20] Themis: Fair and Efficient GPU Cluster Scheduling**
    - ML 训练工作负载通常是需要批量调度的长时间运行的作业，并且它们的性能对任务的相对位置很敏感
    - Themis 是一种ML 训练工作负载的新调度框架。
      - 该机制是一种以完成时间进行公平调度的GPU分配策略(在一个共享的集群中有N个应用程序的运行时间与在一个1/N集群中单独运行的运行时间的比率)。
      - Themis 的目标是在有效利用集群 GPU 的同时，保证各工作负载调度的公平性并最大限度地减少所有 ML 应用程序的完成时间。
      - Themis 包含两级调度架构
        - 其中 ML 作业在调度引擎中对可用资源进行投标，这样可以使调度引擎可以捕获布局敏感性并确保效率。
        - 调度引擎的分配是通过在短期内将GPU 分配给中标者以换取更多效率，但在长期内仍然确保所有工作完成时间的公平性。
      - Themis 在 Apache YARN 3.2.0 上实现，并通过重放大型企业跟踪中的工作负载进行评估，公平性提高了 2.25 倍以上，集群效率提高了 ~5% 到 ~250%。
- **[EuroSys '20] Balancing Efficiency and Fairness in Heterogeneous GPU Clusters for Deep Learning**
    - Gandiva-fair
        - 效率与公平的平衡
          - 效率：Gandiva-fair 提供用户之间的性能隔离，使多个用户可以共享一个集群，从而最大限度地提高集群效率
          - 公平：在活跃用户之间公平分配集群范围 GPU 时间
        - 集群异质性
          - 背景：新一代用户面临更高的需求，老一代 GPU 的利用率很低，从而降低了集群效率
          - 解决方案： Gandiva-fair 描述了来自较新 GPU 的各种作业的可变边际效用，并通过一种新颖的资源交易机制透明地激励用户使用较旧的 GPU
          - 效果：该机制不影响任何用户的公平性保证的情况下最大限度地提高集群效率
- **[NSDI '19] Tiresias: A GPU Cluster Manager for Distributed Deep Learning**
    - 深度学习 (DL) 训练作业给现有的集群管理器带来了一些独特的挑战，例如
      - 不可预测的训练时间
      - 全有或全无的执行模型
      - GPU 共享的不灵活性
    - 我们对生产中的大型 GPU 集群的分析表明，现有的大数据调度程序会导致
        - 较长的排队延迟
        - 较低的整体性能
    - 我们介绍了 Tiresias
        - 这是一个为分布式 DL 训练作业量身定制的 GPU 集群管理器，它可以有效地安排和放置 DL 作业以减少它们的作业完成时间 (JCT)。
        - 鉴于 DL 作业的执行时间通常是不可预测的，我们提出了两种调度算法——
          - **离散化二维Gittins索引**：基于部分信息
          - **离散化二维 LAS**： 与信息无关，旨在最小化平均 JCT
        - 此外，我们描述了何时可以放宽合并放置约束，并提出了一种**放置算法**来利用这些观察结果而无需任何用户输入。
    - 在具有 60 个 P100 GPU 的密歇根 ConFlux 集群上进行的实验和大规模跟踪驱动模拟表明，
        - 与生产中使用的基于 Apache YARN 的资源管理器相比，Tiresias 将平均 JCT 提高了 5.5 倍。
        - 更重要的是，Tiresias 的性能与假设完美知识的解决方案的性能相当。
- **[ATC '19] Analysis of Large-Scale Multi-Tenant GPU Clusters for DNN Training Workloads**
  - 调度框架旨在提供如下特性
    - 高效率
    - 资源隔离
    - 用户间公平共享
  - 然而，基于深度神经网络 (DNN) 的工作负载主要在 GPU 上训练，与传统的大数据有两个显着差异数据分析工作负载
    - 首先，从集群利用率的角度来看，GPU 代表了无法在用户之间以细粒度共享的单一资源
    - 其次，从工作负载的角度来看，深度学习框架需要gang schedule，这降低了调度的灵活性，并使作业本身在运行时对故障没有弹性

  - 在本文中
    - 我们展示了来自 Microsoft 多租户 GPU 集群的长达两个月的跟踪的详细工作负载特征
    - 通过将调度程序日志与来自各个作业的日志相关联，我们研究了影响多租户集群上 DNN 训练工作负载的集群利用率的三个不同问题：
      - (1) 队列调度和位置约束的影响
      - (2) 位置的影响关于 GPU 利用率
      - (3) 训练期间的失败。
    - 根据我们运行大规模操作的经验，我们提供了与用于 DNN 训练工作负载的下一代集群调度器相关的设计指南
- **[OSDI '18] Gandiva: Introspective cluster scheduling for deep learning**
    - Gandiva: 一个集群调度框架，使用特定领域知识，优化了GPU集群训练深度学习模型的延迟与效率
      - 深度学习job的特征
        - 1）反馈驱动的探索：
          - 一个用户经常运行一组作业(或 a multi-job)来获得特定任务的最佳结果
          - 并使用关于准确性的早期反馈来动态优先考虑或杀死一个作业子集
          - 同步发生的多个作业的早期反馈是至关重要的
        - 2）深度学习工作在资源使用方面的**异构**，这使得它很难实现最适合的先验。
        - 3）作业内可预测性：因为作业会重复执行叫做mini-batch的迭代
          - Gandiva利用这个特征解决了1）2）两个问题
          - 利用可预测性对GPU进行多个job间进行时分复用， 这提供了低延迟
          - 这种预测性还可以用于内省job性能并动态迁移到最合适的GPU上，提高了集群效率
      
      - 我们通过一个原型实现和微基准测试表明
        - Gandiva 可以在深度学习过程**加快超参数搜索**一个数量级
        - 并通过**透明迁移**和**job时分**实现更好的利用，使job与资源的更好地匹配。
        - 在一个运行在180-GPU 集群中的实际工作负载中，Gandiva 将集群的总利用率提高了26% 
        - 这为深度学习提供了一种管理大型 GPU 集群的新方法。

## **Memory Management for Machine Learning**

- **[ASPLOS '23] DeepUM: Tensor Migration and Prefetching in Unified Memory**
- **[ATC '22] Memory Harvesting in Multi-GPU Systems with Hierarchical Unified Virtual Memory**
- **[MobiSys '22] Memory-efficient DNN Training on Mobile Devices**
- **[HPCA '22] Enabling Efficient Large-Scale Deep Learning Training with Cache Coherent Disaggregated Memory Systems**
- **[ASPLOS '20] Capuchin: Tensor-based GPU Memory Management for Deep Learning**
- **[ASPLOS '20] SwapAdvisor: Push Deep Learning Beyond the GPU Memory Limit via Smart Swapping**
- **[ISCA '19] Interplay between Hardware Prefetcher and Page Eviction Policy in CPU-GPU Unified Virtual Memory**
- **[ISCA '18] Gist: Efficient Data Encoding for Deep Neural Network Training**
- **[PPoPP '18] SuperNeurons: Dynamic GPU Memory Management for Training Deep Neural Networks**
- **[MICRO '16] vDNN: Virtualized Deep Neural Networks for Scalable, Memory-Efficient Neural Network Design**

## **Scheduling & Resource Management**

- **[ASPLOS '23] ElasticFlow: An Elastic Serverless Training Platform for Distributed Deep Learning**
- **[arXiv '22] EasyScale: Accuracy-consistent Elastic Training for Deep Learning**
  - 分布式同步GPU训练通常被用于深度学习。
    - 使用固定GPU的资源约束
      - 使得大规模的深度学习训练工作受到影响
      - 降低了集群的利用率
    - 纳入资源弹性
      - 往往会引入模型精度的非确定性<-----缺乏隔离能力
  - 本文介绍EasyScale，
    - 这是一个弹性框架
      - 可以在异构GPU上扩展分布式训练
      - 同时产生确定性的深度学习模型
    - 实现了弹性的精度一致的模型训练。
      - EasyScale严格遵循数据并行训练流程
      - 仔细追踪与精度相关的因素
      - 有效利用深度学习特性进行上下文切换
    - 为了使异构GPU的计算能力达到饱和
      - EasyScale根据我们的作业内和作业间调度策略动态地分配工人
      - 最大限度地减少GPU的空闲时间
      - 并相应地提高综合作业的吞吐量。
    - 实验
      - 部署在CompanyA的一个在线服务集群中
      - EasyScale为弹性深度学习训练作业提供动力，使其适时地利用空闲的GPU
      - 在不违反SLA的情况下将集群的整体利用率提高了62.1%
- **[MLSys '22] VirtualFlow: Decoupling Deep Learning Models from the Underlying Hardware**
- **[SIGCOMM '22] Multi-resource interleaving for deep learning training**
- **[EuroSys '22] Out-Of-Order BackProp: An Effective Scheduling Technique for Deep Learning**
- **[ATC '21] Zico: Efficient GPU Memory Sharing for Concurrent DNN Training**
- **[NeurIPS '20] Nimble: Lightweight and Parallel GPU Task Scheduling for Deep Learning**
- **[OSDI' 20] KungFu: Making Training in Distributed Machine Learning Adaptive**
- **[OSDI '20] PipeSwitch: Fast Pipelined Context Switching for Deep Learning Applications**
- **[MLSys '20] Salus: Fine-Grained GPU Sharing Primitives for Deep Learning Applications**
  - GPU 利用率不足
    - 现代 GPU 本身不支持细粒度共享原语
    - 分时和抢占是代价昂贵的
    - DL 应用程序不能完全使用 GPU 的资源， 且这些资源无法被共享
  - Salus 
    - 支持两个 GPU 共享原语，以实现多个 DL 应用程序之间的细粒度 GPU 共享， 原语是：
      - 快速作业切换
      - 内存共享
    - Salus 实现了一个高效的、统一的执行服务，将 GPU 共享给不同的 DL 应用程序，并通过执行迭代调度和解决相关的内存管理问题来实现细粒度共享。
    - 我们展示了这些原语可以用来实现灵活的共享策略，比如公平性、优先级排序和为各种用例打包。
    - 将 Salus 与 TensorFlow 相结合，对流行的 DL 工作进行评估，结果表明 Salus 可以提高 DL 培训工作的平均完成时间3.19 × ，超参数调整的 GPU 利用率2.38 × ，DL 推理应用的 GPU 利用率42 × 以上，不共享 GPU 和7 × 以上 NVIDIA MPS，且开销较小。
- **[SOSP '19] Generic Communication Scheduler for Distributed DNN Training Acceleration**
- **[EuroSys '18] Optimus: An Efficient Dynamic Resource Scheduler for Deep Learning Clusters**
  - Optimus
    - 一个为深度学习集群定制的作业调度器，它基于在线资源性能模型使作业训练时间最小化。
    - Optimus使用在线拟合来预测训练期间的模型收敛，并设置性能模型来准确估计训练速度，作为每个作业分配资源的函数。
    - 基于这些模型，我们设计了一个简单而有效的方法，用于动态分配资源和放置深度学习任务，以尽量减少作业完成时间。
    - 实验
      - 我们在Kubernetes（一个用于容器编排的集群管理器）之上实现了Optimus
      - 并在一个有7个CPU服务器和6个GPU服务器的深度学习集群上进行了实验
      - 使用MXNet框架运行9个训练作业
      - 结果显示，Optimus在作业完成时间和makespan方面分别比有代表性的集群调度器高出约139%和63%
- **[HPCA '18] Applied Machine Learning at Facebook: A Datacenter Infrastructure Perspective**

## **Serving Systems (& inference acceleration)**

- **[EuroSys '23] Fast and Efficient Model Serving Using Multi-GPUs with Direct-Host-Access**
- **[MICRO '22] DFX: A Low-latency Multi-FPGA Appliance for Accelerating Transformer-based Text Generation**
- **[ATC '22] Serving Heterogeneous Machine Learning Models on Multi-GPU Servers with Spatio-Temporal Sharing**
- **[OSDI '22] Orca: A Distributed Serving System for Transformer-Based Language Generation Tasks**
- **[OSDI '22] Achieving μs-scale Preemption for Concurrent GPU-accelerated DNN Inferences**
- **[ATC '21] INFaaS: Automated Model-less Inference Serving**
- **[OSDI '20] Serving DNNs like Clockwork: Performance Predictability from the Bottom Up**
- **[ISCA '20] MLPerf Inference Benchmark**
- **[SOSP '19] Nexus: A GPU Cluster Engine for Accelerating DNN-Based Video Analysis**
- **[ISCA '19] MnnFast: a fast and scalable system architecture for memory-augmented neural networks**
- **[EuroSys '19] μLayer: Low Latency On-Device Inference Using Cooperative Single-Layer Acceleration and Processor-Friendly Quantization**
- **[EuroSys '19] GrandSLAm: Guaranteeing SLAs for Jobs in Microservices Execution Frameworks**
- **[OSDI '18] Pretzel: Opening the Black Box of Machine Learning Prediction Serving Systems**
- **[NSDI '17] Clipper: A Low-Latency Online Prediction Serving System**

## **Deep Learning Compiler**

- **[PLDI '21] DeepCuts: A Deep Learning Optimization Framework for Versatile GPU Workloads**
- **[OSDI '18] TVM: An Automated End-to-End Optimizing Compiler for Deep Learning**

## **Very Large Models**

- **[ASPLOS '23] Optimus-CC: Efficient Large NLP Model Training with 3D Parallelism Aware Communication Compression**
- **[arxiv '21] ZeRO-Infinity: Breaking the GPU Memory Wall for Extreme Scale Deep Learning**
- **[ATC '21] ZeRO-Offload: Democratizing Billion-Scale Model Training**
- **[FAST '21] Behemoth: A Flash-centric Training Accelerator for Extreme-scale DNNs**

## **Deep Learning Recommendation Models**

- **[OSDI '22] FAERY: An FPGA-accelerated Embedding-based Retrieval System**
- **[OSDI '22] Ekko: A Large-Scale Deep Learning Recommender System with Low-Latency Model Update**
- **[EuroSys '22] Fleche: An Efficient GPU Embedding Cache for Personalized Recommendations**
- **[ASPLOS '22] RecShard: statistical feature-based memory optimization for industry-scale neural recommendation**
- **[HPCA '22] Hercules: Heterogeneity-Aware Inference Serving for At-Scale Personalized Recommendation**
- **[MLSys '21] TT-Rec: Tensor Train Compression for Deep Learning Recommendation Model Embeddings**
- **[HPCA '21] Tensor Casting: Co-Designing Algorithm-Architecture for Personalized Recommendation Training**
- **[HPCA '21] Understanding Training Efficiency of Deep Learning Recommendation Models at Scale**
- **[ISCA '20] DeepRecSys: A System for Optimizing End-To-End At-scale Neural Recommendation Inference**
- **[HPCA '20] The Architectural Implications of Facebook’s DNN-based Personalized Recommendation**
- **[MICRO '19] TensorDIMM: A Practical Near-Memory Processing Architecture for Embeddings and Tensor Operations in Deep Learning**

## **Hardware Support for ML**

- **[ISCA '18] A Configurable Cloud-Scale DNN Processor for Real-Time AI**
- **[ISCA '17] In-Datacenter Performance Analysis of a Tensor Processing Unit**

## **ML at Mobile & Embedded Systems**

- **[MobiCom '20] SPINN: Synergistic Progressive Inference of Neural Networks over Device and Cloud**
- **[RTSS '19] Pipelined Data-Parallel CPU/GPU Scheduling for Multi-DNN Real-Time Inference**
- **[ASPLOS '17] Neurosurgeon: Collaborative Intelligence Between the Cloud and Mobile Edge**

## **ML Techniques for Systems**

- **[ICML '20] An Imitation Learning Approach for Cache Replacement**
- **[ICML '18] Learning Memory Access Patterns**