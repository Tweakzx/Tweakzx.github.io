---
title: "【论文笔记】Gandiva论文阅读笔记"
description: 
date: 2022-11-08T09:59:52+08:00
image: https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202211081933118.png
math: 
license: 
hidden: false
comments: true
draft: false
categories: 
    - 论文
tags: 
    - Gandiva
    - 论文
---

# Gandiva 论文阅读笔记

## Abstract

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

## Introduction

- DLT  job的特征

  - 反馈驱动

    - 超参数搜索：

      - 用户通常会尝试一个作业的几个配置（a multi-job），并利用这些作业的早期反馈来决定是否优先处理或杀死其中的一些子集。
    > 传统的调度器运行一个工作子集， 完成前其他工作排队
    >
    > - 这种模式不适合multi-jobs，因为a multi-job中的所有工作需要**同时**得到早期反馈。
    > - 另外，伴随着multi-jobs，其他已经确定了正确的超参数的DLT作业，运行了几个小时到几天，导致了行头阻塞，因为长期运行的作业对GPU拥有独家访问权，直到完成，而取决于早期反馈的多作业则在队列中等待。
    > - 长的排队时间迫使用户要么使用预留的GPU，要么要求集群超额配置，从而降低集群效率。

  - 异构

    - Jobs之间的固有差异
      - 显存使用
      - 核使用
      - 带宽敏感性
      - job之间的干扰

    > 传统调度器将job视作黑箱，无法取得最优的集群效率

  - 作业内可预测

- Gandiva

  - Gandiva利用可预测性来执行剖析驱动的自省。
    - 它使用小批量的进度率来不断反省其决策，以提高集群效率。
    - 例如，只有在内存和GPU利用率较低时，它才会将多个作业装箱在同一个GPU上
    - 它动态地将通信密集型作业迁移到更多的亲和力强的GPU上
    - 它还会适时地 "增长 "作业的并行程度，以利用空闲资源，并在空闲资源消失后缩小作业。
    - 我们目前实施的自省策略是一个有状态的试错策略，它是可行的，因为它的预测能力很强。
  - Gandiva除了特定的内省核调度策略，还提供了一些API
  - API：（a）高效的挂起恢复或时间切片，（b）低延迟迁移，（c）细粒度监控，（d）弹性，以及（e）动态优先级。
    - 这些原语高效的关键是Gandiva的协同设计：跨越调度器层与DLT工具包层（如pytorch， tensorflow)
  - 通过利用GPU集群的专用性，Gandiva为深度学习的特定工作负载定制了调度器，从而为调度器提供了对工作的更多可见性和控制，同时仍然实现了对任意DLT工作的通用性。
  - Gandiva的实现
    - 修改两个流行的框架 PyTorch 和 Tensorflow ，为调度程序提供必要的新原语
    - 并在 Kubernetes 和 Docker 容器之上实现了一个初始调度策略管理器

- 本文贡献
  - 我们说明了深度学习工作流程的各种独特特征，并将其映射到集群调度所需的具体要求。
  - 我们确定了DLT作业调度策略可以使用的通用原语，并提供了应用感知技术，通过利用DL特有的作业内周期性知识，使时间切割和迁移等原语的效率提高了一个数量级，从而变得实用。
  - 我们提出并评估了一个新的自省式调度框架，该框架利用DLT工作的特定领域知识来不断完善其调度决策，从而显著改善早期反馈时间并提供高集群效率。

## Backgroud

- 反馈驱动的探索。

  - 实现高精确度的一个前提条件是模型的选择。新模型的发现，如ResNet或Inception，如今大多是一个试错的过程，尽管使之自动化的方法是一个活跃的研究领域。
  - 除了模型结构外，还有一些参数，称为超参数，也需要作为DLT工作的一部分被指定。超参数包括模型中的层数/权重、最小批量大小、学习率等。这些参数通常由用户根据领域知识和试错来选择，有时甚至会导致早期训练失败。
  - 因此，DLT工作的早期反馈是至关重要的，特别是在训练的初始阶段。

- multi-job

  - 一旦用户确定了要进一步探索的特定模型，用户通常会进行超参数搜索以提高任务的准确性。

  - 这可以在超参数空间上使用各种搜索技术来完成；也就是说，用户生成多个DLT任务或多任务，每个任务使用**一组超参数或配置**进行全面训练。由于用户通常会探索数百个这样的配置，这个过程在计算上是很昂贵的。

  - 因此，文献中出现了复杂版本的超参数搜索，如HyperOpt和Hyperband。

    >  例如，Hyperband最初可能会产生128个DLT作业，并在每一轮（例如100个小批量迭代）中，杀死一半精度最低的作业。

  - 同样，对于这些算法来说，对整个作业集的早期反馈是至关重要的，因为否则他们将无法做出有效的训练决定。

## DLT job的特征

### 对位置（locality)敏感

- 多GPU DLT工作的性能取决于分配的GPU的亲和力。
  - 不同的DLT工作对GPU间的亲和力表现出不同程度的敏感性。
  - 即使是同一台机器上的GPU，由于不对称的架构，我们观察到不同程度的GPU之间的亲和力
    - 两个GPU可能位于不同的CPU插槽（表示为DiffSocket）
    - 在同一个CPU插槽，但在不同的PCIe Switch（表示为SameSocket）
    - 在同一个PCIe Switch（表示为SamePCIeSw）

![image-20221108193230029](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202211081932589.png)

- 图1显示了两个模型VGG16[44]和ResNet-50[24]对服务器内定位的不同敏感性。
  - 当使用Tensorflow的两个P100 GPU进行训练时，VGG16在不良定位下受到很大影响。
  - 在最差的定位下，当两个GPU位于不同的CPU插座上时，VGG16只实现了最佳定位配置的60%，即两个GPU被放置在同一个PCIe开关下。
  - 另一方面，在这种设置下，ResNet-50不受GPU定位的影响。这是因为VGG16是一个比ResNet-50更大的神经模型，因此在每个小批次中的模型同步会在底层PCIe总线上产生更高的通信负荷。
- 我们在分布式环境中观察到类似的趋势。图2显示了一个4GPU Tensorflow作业的性能，它以不同的服务器间定位运行，训练ResNet-50和InceptionV3[46]模型。
  - 即使是用40G InfiniBand网络互连，当作业被分配到4个GPU时，性能差异明显，
    - 其中它们均匀地分散在4台服务器（表示为4*1-GPU）
    - 2台服务器（表示为2*2-GPU）
    - 以及全部在一台服务器（表示为本地4GPU）
    - 尽管两个模型对位置性的敏感性不同。
- 因此，DLT调度器在分配GPU时必须考虑到作业对位置的敏感性。

### 对干扰敏感

- 当在一个共享的执行环境中运行时，DLT工作可能会因为资源争夺而相互干扰。我们再次观察到，不同的DLT工作表现出不同程度的干扰。

![image-20221108193107517](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202211081932419.png)

- 即使对于单GPU作业，也存在干扰。
  - 图3显示了: 当把一个语言模型作业（标记为LM）与另一个作业放在同一个PCI-e交换机下时，由于服务器内的干扰而导致的性能下降情况。
    - 当两个LM一起运行时，两个作业都会遭受19%的减速。
    - 然而，ResNet-50并没有受到GPU与LM共处的影响。
    - 神经机器翻译（GNMT）[51]对LM的干扰程度不大。
    - 同样地，我们也观察到不同类型的训练模型对多GPU训练的不同程度的干扰。

- 图4显示了用40G InfiniBand网络连接的两个4GPU服务器上的服务器间干扰。
  - 当运行多个2-GPU作业时，每个GPU被放在不同的服务器上，
    - ResNet-50显示出高达47%的减速，
    - InceptionV3显示出30%的减速，
    - 而DeepSpeech[23]仅显示出5%的减速。

- 总之，不同应用领域的流行深度学习模型，如视觉、语言和语音，表现出对位置性和干扰的不同程度的敏感性。为了迎合这些挑战，Gandiva利用了DLT工作的一个关键特征，我们接下来会详细说明。

### job内可预测性

![image-20221108193308480](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202211081933633.png)

- 一个DLT作业包括许多小批量的迭代， 呈现对应的周期性。
  - 图5(a)显示了在四个K80 GPU上使用ResNet-50模型对ImageNet数据进行训练的20s快照期间使用的GPU总内存。
    - 所使用的GPU内存明显遵循一个周期性模式。
    - 每个周期都对应着一个小批次的处理（大约1.5s），内存在前向传递中增加，在后向传递中减少。
    - 使用的最大和最小的GPU内存分别为23GB和0.3GB，或77倍的系数。
    - 这个比例随着迷你批处理量的增加而扩大（通常在16到256之间；本例中为128）。
  - 图5(b)显示了在一个K80 GPU上使用GNMT模型时，对WMT'14英语德语数据集进行训练的20s快照所使用的GPU总内存。
    - 虽然小批量迭代与ImageNet的例子不完全相同（由于不同的句子长度和PyTorch中使用的动态图），但该图具有类似的循环性质。
    - 最大值和最小值之间的差异较小（3倍），主要是由于较大的模型（0.4GB）和较小的迷你批次大小（本例中为16）。
  - 除了这里显示的图像和语言模型外，其他训练领域，如语音、生成式逆向网络（GAN）和变异自动编码器都遵循类似的循环模式（由于空间限制没有显示），因为训练的核心是梯度下降算法，执行许多小批量迭代。
- 充分利用可预测性。
  - 首先，一个DLT作业可以被自动分割成小批量的迭代，这些迭代在60秒内的集合，例如一个微任务，形成一个调度间隔。
  - 第二，通过在内存周期的最小值上执行暂停操作，可以大大减少从GPU复制到CPU中的内存量，从而使暂停/恢复和迁移的效率比naive的实现要高一个数量级。
  - 第三，可以对小批量的进度进行分析，并将其作为评估装箱或迁移等机制的有效性的代理。

## 设计

![image-20221108193438681](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202211081934805.png)

- 如今的集群中出现高延迟和低利用率的问题
  - 是因为DLT作业被专门分配了一组**固定**的GPU。
  - **独占**访问 GPU 的原因行头阻塞，阻塞了早期反馈，导致作业的排队时间过长。
  - 当作业无法完全利用其分配的GPU时，对一组固定的GPU的独家访问也会导致GPU的低利用率。

### 机制

- 三种方式消除GPU对DLT作业的排他性和固定分配来解决低效率问题
  - 首先，在过载期间，Gandiva允许后来的工作与现有的工作**共享**GPU
    - 而不是等待当前工作的离开。
    - 这是为DLT作业定制的**挂起-重启机制**和**选择性装箱**而实现的。
  - 第二，Gandiva支持DLT作业从一组GPU到另一组的**高效迁移**
    - 迁移允许时间碎片化的作业迁移到其他（最近空出的）GPU上
    - 或者对集群进行去碎片化处理，从而使后来的作业被分配到具有良好位置性的GPU上。
  - 第三，Gandiva支持GPU增长-缩减机制
    - 这样空闲的GPU就可以适时地被使用。
    - 为了有效地支持这些机制并实现有效的资源管理，Gandiva通过不断地**剖析**DLT作业的资源使用情况并估计其性能，对DLT作业进行内省。

#### 挂起-重启与装箱

- Suspend-resume
  - 挂起-重启是Gandiva用来消除一组GPU对DLT作业的独占性的一种机制。
    - Gandiva利用这种机制，增加了对GPU时间分割的自定义支持。
  - Gandiva的关键思想是利用这种周期性行为，在DLT作业的**GPU显存使用量最低**时暂停-恢复。
    - 1）发出暂停调用
    - 2）DLT工具包会等到内存使用周期的最小值，将存储在GPU中的对象复制到CPU，释放其所有GPU内存分配（包括缓存）
    - 3）然后调用经典的CPU暂停机制
    - 4）之后，当CPU恢复工作时，DLT框架首先分配适当的GPU内存，将存储的对象复制回GPU，然后恢复工作。
  - Suspend-resume也可以在同一台服务器中启动GPU的更换
    - 虽然更换GPU的成本很高，但我们可以把这个延迟从关键路径中隐藏
      - 典型的图像分类工作，暂停-恢复一起可以在100ms内完成
      - 而对于大型语言翻译工作，暂停-恢复可能需要1s
    - 考虑到1分钟的时间切分间隔，这相当于2%或更少的开销
  - 延迟
    - Gandiva中的suspend可能最多**延迟**DLT作业的一个小批次间隔
    - 值得
      - 由于减少了GPU-CPU的复制成本和更少的CPU使用的内存，它的开销明显减少
      - 在这个延迟期间还完成了有用的工作。
    - 调度器跟踪这一延迟，并相应地调整时间分割的间隔，以实现公平性

- 装箱
  - 暂停-恢复的另一种方法是装箱
    - 在一个GPU上同时运行多个DLT作业，让GPU分担作业的时间。
  - 有效性
    - 只有当装箱的作业不超过GPU的资源，并且不会对彼此产生不利影响时，GPU中的装箱才是有效的。
    - 如果作业相互干扰，装箱就会比暂停-恢复差很多。
  - 装箱的使用
    - 当DLT作业有排他性的访问时，我们使用Profiling来监测它们的资源和进度
    - 如果两个作业被确定为装箱的候选者，我们就把它们装箱在一起，并继续监控它们
    - 如果给定的装箱结果对作业的性能产生了不利影响，我们就解除这些作业的装箱并恢复到暂停-恢复状态

#### 迁移

- 迁移是Gandiva用来改变分配给DLT作业的GPU集的机制。
  - 迁移在几种情况下是有用的
    - i）将时间分割的作业移到集群中任何地方的空闲GPU上
    - ii）将相互干扰的作业移开
    - iii）对集群进行去碎片化处理，使进入的作业获得具有良好位置性的GPU
  - 我们评估了两种解决DLT进程状态迁移的方法。
    - 在第一种方法中，我们利用一个通用的进程迁移机制，如**CRIU**。
      - CRIU本身不支持使用GPU设备的进程迁移
        - 我们首先对GPU对象进行检查
        - 调用CRIU之前从进程中删除所有GPU状态
      - 因为CRIU检查点和恢复整个进程的内存
        - 对于这些使用PyTorch的DLT作业来说，检查点的大小是GB级别的。
        - 对于单GPU作业来说，所产生的迁移开销约为810s，对于多GPU作业来说则更高。
    - 我们考虑的第二种方法是使用具有检查点意识的DLT作业。
      - Tensorflow等DLT框架已经支持允许自动检查点和恢复模型的API
      - 通过在迁移前对目的地进行**预热**，并且只迁移必要的训练状态。我们可以降低迁移开销小到一两秒
  - 无论采用哪种方法，我们发现，与它在提高GPU总体利用率方面提供的好处相比，服务器间迁移的开销是值得的

#### 增长与收缩

- Gandiva用来消除GPU对DLT作业的排他性的第三个机制是增长-收缩。
  - 这种机制主要针对集群可能没有被完全利用的情况
  - 基本思想
    - 在空闲时间内，适时地增加可用于作业的GPU数量
    - 在负载增加时相应地减少可用的GPU数量
  - 许多DLT工作，特别是在图像领域，随着GPU数量的增加，会看到线性的性能扩展。
    - Gandiva只对那些特别声明他们有足够的适应性来利用这些增长机会的DLT工作应用这一机制。
    - 当多个DLT工作符合这一标准时，Gandiva使用Profiling来估计每个工作的进度，然后相应地分配GPU。

#### Profiling

- Gandiva监控资源使用情况
  - 如CPU和GPU利用率，CPU/GPU内存等。
  - Gandiva的独特之处在于，它还以一种应用感知的方式内省
    - 利用了DLT作业表现出的规律内省
    - 利用周期性以估计DLT进度
  - Gandiva估计DLT作业的mini_batch时间
    - 即对一批输入数据做一次前向/后向传递的时间，作为GPU内存使用周期的两个最小值之间的时间
    - 由于DLT作业在其生命周期中通常会执行数百万次这样的小批量操作，调度器会在调度决策之前和之后比较DLT的mini_batch时间以确定其有效性。
  - Gandiva可以决定装箱是否有效
    - 通过比较装箱前后两个DLT作业的小批量时间
    - 如果没有这样的剖析，为了做出装箱的决定
      - 人们不仅要对两个DLT作业在不同GPU上的性能进行建模，
      - 还要对它们可能相互干扰的各种方式进行建模（例如，缓存、内存带宽等）
      - 这不是一项简单的任务

### 调度原则

- 定义：在我们描述调度器的细节之前，我们定义一些术语。
  - DLT作业
    - 被封装在容器中, 包括
      - 所需的GPU数量
      - 优先级（可以是动态的 ）
      - 一个指示作业是否能够增长-收缩的标志
    - 我们假设一个作业所要求的GPU数量是2的幂。
  - 集群
    - 一个集群由一个或多个服务器组成
    - 每个服务器有一个或多个GPU
    - 我们假设一个专门的GPU集群用于DLT工作
  - 服务器的高度
    - 定义为⌈M/N]， 其中M是分配的GPU数量，N是总GPU的数量。
    - 只有当服务器的高度超过1时，才会使用暂停/恢复机制。
  - 集群的高度
    - 被定义为其所有服务器的最大高度。
    - 当集群的高度大于1时，就会出现过载；即所有作业的请求/分配的GPU之和大于GPU的总数。
  - 服务器的亲和力
    - 被定义为分配给该服务器的作业类型。
    - 例如，最初服务器的亲和力为零，如果一个需要两个GPU的作业被分配到一个服务器上，那么该服务器的亲和力就会变成两个。
    - 这个参数被调度器用来将具有类似GPU需求的作业分配到同一台服务器上。

- 目标
  - Gandiva调度器的主要设计目标是**为作业提供早期反馈**。
    - 在流行的调度器中，作业在过载期间会在队列中等待。
    - Gandiva通过立即为新作业分配GPU并使用suspend-resume机制提供早期结果来支持超额订阅。
  - 第二个设计目标是**集群效率**。
    - 通过一个持续的优化过程来实现，该过程使用了剖析和贪婪的启发式，利用了诸如装箱、迁移和增长-收缩等机制。
  - 集群级的公平性**不是**Gandiva的设计目标。
    - 只关注使用暂停-恢复机制在每个服务器上提供作业之间的公平性
    - 集群级的公平性留给未来工作
  - 为了实现这些目标，Gandiva调度器以两种模式运行：
    - 响应模式：调度器对诸如工作到达、离开、机器故障等事件做出反应
    - 内省模式： 一个持续的过程，过程中，调度器的目标是提高集群的利用率和作业完成时间
    - 请注意，调度器可以同时在两种模式下运行。

#### 响应模式

响应模式被设计用来处理诸如工作到达、离开和机器故障的事件

![image-20221108220646915](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202211082206024.png)

当一个新的作业到达时，调度器为该作业分配服务器/GPU。

- Gandiva使用的节点分配策略如算法1所示。 

  - findNodes是一个函数，用于返回满足作业请求的节点候选者，并有一个亲和力约束的可选参数。
  - 最初，Gandiva试图找到与新工作具有相同亲和力的节点，并在这些节点中找到具有最小负载的节点。如果存在这样的节点，并且它们的高度小于1（第5-6行），该节点将被分配。
  - 否则，Gandiva试图找到并分配未亲和的节点（第7-8行）。
  - 如果没有这样的空闲服务器，第三个选择是寻找有空闲GPU的节点，同时忽略亲和力（第9-10行）。
  - 这可能会导致多个节点之间的碎片化分配，但正如我们将在后面看到的，迁移可以用于碎片化。如果上述方法都不奏效，这意味着集群中没有可用的GPU。在这种情况下，如果存在具有相同亲和力的节点，它们将被用于暂停-恢复（第11-12行）；
  - 如果没有，作业将被排队（第13-14行）。

- 放置job

  - > 传统的调度器将使用**作业离开**来触发从等待队列中挑选下一个作业。

  - Gandiva检查集群的高度是否可以减少

    - 将被暂停的作业迁移到新的空闲GPU上
    - 这个作业可能来自同一台服务器或集群中的任何其他服务器。
    - 作业的离开也可以触发迁移，以改善位置性

- Gandiva的工作安排Policy时考虑到了两个因素。
  - Gandiva**允许超额认购**。
    - 当一个服务器被超额认购时，我们会进行**加权轮流调度**，给每个作业公平的时间份额。
  - GPU分配不是作业到达时的一次性事件，Gandiva使用自省模式来**持续改善**集群的利用率。
    - 因此，Gandiva依靠一个简单的作业安置策略来快速分配GPU资源给新作业，从而实现早期反馈。

#### 内省模式

在自省模式下，Gandiva持续监控并优化作业在集群中的GPU上的位置，以提高DLT作业的整体利用率和完成时间。

- 装箱

  - 只有在过载时才会考虑装箱。
    - 基本思想：在GPU上同时运行两个或多个作业以提高效率。
    - 可能无效
      - 如果装箱工作的内存需求加起来高于GPU的内存，那么从CPU内存中 "分页 "的开销就会很高，装箱就没有效果。
      - 当两个或更多作业的内存需求小于GPU内存时，装箱仍然可能不比暂停-恢复更有效。
  - 鉴于DLT作业的异质性，对装箱的性能进行分析建模是一个具有挑战性的问题。
    - 相反，Gandiva依靠一种贪婪的启发式方法来装箱job。
      - 当job到达时，我们总是使用暂停-恢复的独占模式运行它们，并收集剖析信息（GPU利用率、内存和作业进展率）。
      - 基于剖析数据，调度器维护一个按GPU利用率排序的作业列表。
      - 调度器贪婪地挑选出GPU利用率最低的作业，并试图将其装箱到具有最低GPU利用率的GPU上。我们只在装箱作业的总内存利用率不超过GPU的总内存时才这样做。
      - 当装箱作业的总吞吐量大于时间切割时，装箱被认为是成功的。
      - 如果装箱不成功，我们就撤消装箱，并尝试下一个利用率最低的GPU。
      - 如果装箱成功，我们找到下一个利用率较低的作业并重复这个过程。
    - 根据我们的评估，我们发现这个简单的贪婪的启发式方法实现了26%的效率提升。

- 迁移

  - GPU的位置性在一些作业的性能中起着重要作用。
  - 在Gandiva中
    - 每当作业离开时，我们都会使用迁移来改善位置性，
    - 同时也作为一个后台进程来 "整理 "集群的内容。
    - 为了改善位置性，我们挑选那些不在同一地点的作业，并试图找到一个新的同一地点的位置。

  > ![image-20221109111201868](https://cdn.jsdelivr.net/gh/Tweakzx/ImageHost@main/img/202211091112968.png)
  >
  > - 图8展示了一个集群实验的例子（第6.4节）。
  >   - 当一个有4个作业、每个作业需要2个GPU的多作业被安排时，它的GPU亲和性很差；
  >     - 只有J0的两个GPU被安排在一起，而多作业中的其他3个作业（J1、J2和J3）被分配到不同的GPU。
  >     - 三分钟后，一个背景训练作业DeepSpeech完成了，并释放了它的8个GPU。
  >     - 8个GPU中的3个，在图8中标记为D，位于三个不同的服务器（服务器1、3和4），可以提高多任务的训练效率。
  >     - 因此，Gandiva启动了迁移过程，将J1、J2和J3重新安置到同地的GPU。

  - 去碎片化
    - 我们在所有非空闲的服务器中挑选出拥有最多空闲GPU的服务器。
    - 然后我们尝试将运行在该服务器上的作业迁移到其他服务器上。
    - 只要性能损失可以忽略不计，作业就会被迁移到另一个空闲GPU较少的服务器上。
    - 我们重复这个过程，直到每台非空闲服务器上的空闲GPU数量少于阈值（在我们的实验中是4个中的3个），或者没有作业会从迁移中受益。

- 扩缩

  - 增长-收缩的条件
    - 集群未被充分利用
    - DLT作业明确指出自己可以进行增长-收缩
  - 限制
    - 在我们目前的系统中，我们只让作业增长到**单台服务器**中可用的最大GPU数量。
    - 此外，我们只在**空闲一段时间后**触发增长，以避免惊扰，
    - 在新作业可能需要GPU时**立即收缩**。

- 时分

  - 我们在每个服务器中支持轮回调度，以公平地分享GPU。
  - 当作业有多个优先级时，较高优先级的作业将不会被暂停以适应较低优先级的作业。
  - 如果一台服务器被较高优先级的作业完全利用，如果可行的话，较低优先级的作业将被迁移到另一台服务器。

## 实现

DLT作业被封装为Docker容器，其中包含我们定制的DL工具箱和Gandiva客户端的版本。这些工作被提交给Kubernetes系统。Gandiva还实现了一个定制的调度器，然后对这些作业进行调度。

### 调度器

- Gandiva由一个定制的中央调度器和一个客户端组件组成，客户端是每个DLT工作容器的一部分。
- 调度器只是另一个由Kubernetes管理的容器。
  - Kubernetes负责整体集群管理，
  - Gandiva调度器管理DLT作业的调度。
  - Gandiva调度器使用Kubernetes API来获取集群节点和容器信息，每当提交一个新的容器时，调度器会根据调度策略将其分配给集群中的一个或多个GPU。

- 当一个容器被安排在一个节点上时
  - 最初只有Gandiva客户端开始执行。
  - 然后，它轮询Gandiva调度器，以确定哪些GPU可用于DLT工作，
  - 并使用暂停/恢复和迁移命令控制DLT工作的执行。
- 虽然我们集群中所有GPU的调度完全由中央调度器控制，但如果弹性成为我们关心的问题，可能需要一个分层的方法。

### DL工具的修改

- PyTorch的时分。

  - 发出SIGTSTP信信号
    - Gandiva client会发出一个SIGTSTP信号，表示工具包必须暂停进程。
    - 它还指示恢复是否应该通过内存文件在新的GPU中发生。
    - 收到信号后，工具包会设置一个暂停标志，并且只在一个小批处理边界的末端执行暂停。

  - 识别mini-batch边界
    - 在Tensorflow这个定义并运行的工具包中，mini-batch的边界很容易识别（session.run()的结束）。
    - 在PyTorch这个逐一定义的工具包中，
      - 我们通过跟踪GPU内存使用周期来确定mini-batch的边界，
      - 这是PyTorch的GPU内存管理器（THCCachingAllocator）的一部分，
      - 每当GPU内存被释放时，我们就会寻找一个**周期最小值**。
  - 执行暂停
    - 一旦检测到周期最小值，工具包 i）将所有存储的对象从GPU复制到CPU，ii）释放GPU分配，以及iii）暂停进程。
  - 恢复进程
    - 当Gandiva客户端发出SIGCONT信号时，工具包会分配GPU内存，将存储的对象从CPU复制到GPU，并恢复进程。
    - 为了处理重新启动时的**地址改变**，我们跟踪工具包中的GPU对象，并用新的地址对其进行修补。
    - 改变GPU需要调用cudaDeviceReset和CudaInit，这可能需要5-10秒。我们通过在 "暂停 "时在后台执行这些操作来**隐藏这个延时**

- Tensorflow迁移

  - 我们在每个服务器上部署了一个迁移助手，以支持按需checkpoint和迁移。
    - 当收到来自调度器的迁移命令时，目的地助手首先**预热TF会话**并等待检查点的到来。
    - 然后，源帮助者要求TF保存检查点，在跨服务器迁移的情况下将检查点移到目的地，最后恢复训练会话。
    - 为了加快迁移过程，我们采用Ramdisk将检查点保存在内存中。在跨服务器的情况下，修改后的TF通过网络文件系统（NFS）协议将检查点直接保存到远程Ramdisk。
  - 当Migration Helper要求作业执行checkpoint时
    - 修改后的TF会在一个小批处理结束时调用tf.Saver。
    - 对于数据的并行性，检查点只包括一个GPU中的模型，而不考虑训练中使用的GPU数量。
    - 为了进一步加快TF的迁移，我们在检查点中不包括元图结构，因为它可以根据用户代码进行重建。
  - 在**预热阶段**，修改后的TF检查GPU配置并重建元图。
    - 它进一步创建Executor来运行预热操作，以确保初始化不会被懒惰地推迟。
    - 当恢复训练过程时，修改后的TF加载检查点，由多个GPU并行加载，并继续进行训练。

