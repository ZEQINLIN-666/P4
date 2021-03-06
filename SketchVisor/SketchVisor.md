# 摘要

sketch检测数据包时：

- 固定大小的内存
- 误差有限

缺点，高流量负载下性能下降。



**提出网络测量框架：sketchVisor**

在高负载下激活快速路径，提供高性能的局部测量，而精度略有下降。

通过压缩感知，恢复全网测量结果。



# 1 Introduction

软件数据包处理是现代数据中心网络的重要支柱。它强调可编程性和可扩展性，从而支持新的网络管理功能。

网络测量对于管理软件数据包处理平台至关重要。其目标是收集基本的网络流量统计信息（例如，heavy hitters，流量异常，流量分配和流量熵），以帮助网络运营商在流量工程，性能诊断和攻击防范方面做出更好的网络管理决策

现有的软件交换机在数据包采样上测量精度较低，如果要通过提高采样率甚至是记录所有流量来提高测量精度，资源消耗过多，高速网络中可扩展性差。

---

sketch，细粒度测量，是紧凑的数据结构，可以以固定的存储空间去汇总所有数据包的流量统计信息，而仅产生有限的错误

sketch实际上消耗了大量的CPU资源。基于sketch的测量需要过多的CPU资源来满足线速要求

>根本原因是草图只是图元。尽管它们在设计上既简单高效，但将其应用到实际网络测量中却需要额外的扩展或组件，这些扩展或组件通常会导致大量的计算

基于以上原因，我们不仅要求测量解决方案在高流量负载下具有资源效率，而且要准确地推断出高流量负载的行为。

---

sketchvisor高健壮性的网络测量框架。

>健壮性：即使在高流量负载下，SketchVisor仍可以保持高（甚至是线速）性能和高精度，以进行全网范围的测量。 

SketchVisor并未提出新的sketch设计，而是使用单独的数据路径（称为快速路径）扩展了现有的基于草图的解决方案，来测量高流量负载，精度稍有下降。

此后，它可以从基于草图的测量和快速路径测量中准确地恢复整个网络的测量结果。

---

SketchVisor在网络中的软件交换机之间部署了分布式数据平面，每个交换机部署独立的sketch来处理数据包，并在任务超载且无法处理这些任务时将过多的数据包重定向到快速路径高速分组。

快速路径中设计新的top-k算法，跟踪大流量

维护一个全局计数器，统计进入快速路径的流量，也可以捕获小流量的总体特征。

>Notes:快速路径通常可以支持为不同类型的流量统计信息设计的各种测量任务。



SketchVisor部署了一个集中式控制平面，合并来自所有软件交换机的本地测量结果（基于sketch和快速路径的测量）

快速路径可能会丢失信息，所以利用矩阵插值，使得控制平面能够通过压缩感知来恢复丢失的信息。

---

SketchVisor都具有单个CPU内核，并且在快速路径中只有几KB的内存，可以达到17Gbps以上的吞吐量，并且具有接近最佳的精度。



# 2 BACKGROUND AND MOTIVATION  

## 2.1流量监测的常用统计信息  

流量统计信息可以

- 基于流（由5元组标识）
- 基于主机（由IP地址标识）
- 基于卷（通过字节数衡量）
- 基于连接（通过不同的流/主机数衡量）

**Heavy hitter**：一个字节周期中的字节数超过阈值的流
**Heavy changer**：在两个连续的时期内，其字节数变化超过阈值的流。
**DDoS**：在某个时期内，从数量超过阈值数量的源主机接收数据的目标主机
**Superspreade**r：在一个时期内将数据发送到超过阈值数量的目标主机的源主机（即，超级扩展程序与DDoS相反）
**Cardinality**：一个时期中不同流的数量
**Flow size distribution**：一个时期中不同字节计数范围的流分数。
**Entropy**：一个时期的流量分布的熵

## 2.2 性能分析

1.Sketch是一种紧凑的数据结构，包括一组存储桶，每个存储桶都与一个或多个计数器关联。它将每个数据包映射到具有独立哈希功能的存储桶的子集，并更新这些存储桶的计数器。网络运营商可以查询计数器值，以恢复流量统计信息。

2.sketch必须用其他组件和操作来完成一个完整的测量任务。

3.为了收集有意义的流量统计信息，我们必须在sketch上添加扩展名以使其可逆，这意味着草图不仅可以存储流量统计信息，还可以高效地查询流量统计信息。

但是：

- 查询成本高

- Deltoid和FlowRadar扩展了sketch算法，但是会导致大量的计算开销。

4.大多数基于草图的解决方案都是为特定的测量任务和流量统计而设计的

5.UnivMon [30]允许一个草图同时收集不同类型的流量统计信息。但是，它需要更新各种组件（包括CountSketch [8]和top-k流键），并且计算量仍然很大

6.性能瓶颈在基于草图的解决方案中有所不同.

- 在哈希计算（包括随机化流头以解决哈希冲突）上，FlowRadar和Reversible Sketch分别导致超过67％和95％的CPU周期。

- Deltoid的主要瓶颈在于更新其额外的计数器以对流头进行编码，这占CPU周期的86％以上。 
  - UnivMon分别在散列计算和堆维护上花费了53％和47％的CPU周期。

7.散列表的计算量少于草图[1]，但它们消耗大量的内存使用量

8.一些系统[21、29、38、62]试图通过预定义的规则来过滤流量，以减少存储器的使用。但是，它需要人工来配置适当的规则，以同时实现高精度和存储效率	

9.草图为内存使用情况和错误范围提供了理论上的保证，但会产生较高的计算开销  

# 3 SKETCHVISOR OVERVIEW  

SketchVisor是用于软件数据包处理的强大网络测量框架，其设计目标如下：

- 性能：它高速处理数据包，旨在满足基础数据包处理管道的线速要求。
- 资源效率：它有效地利用CPU进行数据包处理，并利用内存存储数据结构。
- 精度：保留了Sketch的高测量精度。
- 通用性：它支持各种基于sketch的测量任务。
- 简单性：它可以自动减轻高流量负载下基于sketch的测量任务的处理负担，而无需手动的每台主机配置和网络运营商的结果汇总。

## 3.1数据平面

![SketchVisor architecture.PNG](https://github.com/ZEQINLIN-666/P4/blob/master/SketchVisor/SketchVisor%20architecture.PNG?raw=true)

![数据平面逻辑图.PNG](https://github.com/ZEQINLIN-666/P4/blob/master/SketchVisor/%E6%95%B0%E6%8D%AE%E5%B9%B3%E9%9D%A2%E9%80%BB%E8%BE%91%E5%9B%BE.PNG?raw=true)

​																								流量重定向到快速路径的方法



## 3.2 控制平面

它从多个主机收集本地测量结果并将其合并以提供网络范围内的测量结果测量结果，其目标是实现整个网络范围内的精确测量，就好像所有流量仅由每个主机的正常路径进行处理一样。

**挑战：**消除由于快速路径测量而引起的额外错误至关重要。换句话说，所有测量误差应仅来自sketch本身。然而，这种误差的消除在很大程度上取决于快速路径设计，而快速路径设计必须通用才能适应各种测量任务。

同样，控制平面中的错误消除必须适用于任何测量任务。