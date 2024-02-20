# 并行方式

nndeploy当前支持任务级并行和流水线并行两种并行方式。二者面向的场景不同：

+   任务级并行：在多模型以及多硬件设备的的复杂场景下，基于有向无环图的模型部署方式，可充分挖掘模型部署中的并行性，**缩短单次算法全流程运行耗时。**
+   流水线并行：在处理多帧的场景下，基于有向无环图的模型部署方式，可将前处理 `Node`、推理 `Node`、后处理 `Node`绑定三个不同的线程，每个线程又可绑定不同的硬件设备下，从而三个`Node`可流水线并行处理。在多模型以及多硬件设备的的复杂场景下，更加可以发挥流水线并行的优势，从而可显著**提高整体吞吐量**。

## 任务级并行

代码位于`nndeploy/include/nndeploy/dag/graph/parallel_task_executor.h`

任务级并行利用模型内部节点的并行性，将多个节点调度在多个线程中同时执行。假设有一个9节点的有向无环图，其拓扑架构如下，边表示数据的流向，用边连接的两个节点具有生产者-消费者的依赖关系，当生产者节点运行完毕后，消费者节点才能运行。例如E节点需要等C节点和D节点运行完毕后再运行。

![{nodes}](../../image/architecture_guide/nodes.png)

从最初的输入开始。input数据准备好后，A、B节点就可以并行运行，A节点运行完后C节点可以运行，同理B节点运行完后D节点可以运行。从图上看，似乎C、D节点也是并行的。然而在实际运行时，由于A、B节点的运行时间未知，因此C、D节点不一定是并行的，有可能A、C节点都运行结束后，B节点仍在运行。这种运行时间未知带来的问题是无法在编译时就确定哪些节点之间是并行的，因此静态的建立图节点并行计算方式是非常困难的。

nndeploy采用的方式是运行时动态解图，在每个节点计算完毕后再判断其消费者节点是否能执行。主要流程如下：

1.初始化

初始化线程池；

对图进行拓扑排序，返回剔除死节点的排序后图，并记录总任务节点数量；

2.运行

全图运行流程如下：

![{task_parallel}](../../image/architecture_guide/task_parallel_process.png)

从开始节点（入度为0，即没有依赖的节点）出发。更改节点状态为运行中，然后将节点的运行函数和运行后处理函数提交线程池，以异步的方式开始执行。节点的运行函数为用户自己实现的运行函数，运行后处理函数为nndeploy添加。

运行后处理函数包含节点状态更新、提交后续节点、唤醒主线程三部分。

节点状态更新将该节点状态更改为运行结束。

提交后续节点将遍历该节点的每一个后续节点，判断该后续节点的所有前驱节点是否都已运行结束，若结束再将该后续节点提交线程池。例如对于上图E节点，其后继节点为G、H。检查G节点的所有前驱即E节点是否运行完毕，运行完毕则加入线程池，检查H节点的所有前驱节点E、F是否执行完毕，运行完毕则加入线程池，若F节点尚未执行完毕，则H节点会在F节点执行后再检查一次是否可以提交执行。这样可以保证所有的节点都能被提交执行。

若该节点是尾节点（出度为0的节点，即没有后继节点），则检查是否完成节点数量达到所有节点数量，若达到，则唤醒主线程，所有节点均执行完毕。

3.运行后处理

将所有节点执行状态恢复为未运行，以便下次的全图运行。



## 流水线并行

 