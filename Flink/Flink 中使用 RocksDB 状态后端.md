# Flink 中使用 RocksDB 状态后端

原文链接：[何时以及如何在 Apache Flink 中使用 RocksDB 状态后端](https://blog.csdn.net/u010942041/article/details/114944767)

**流处理应用程序通常是有状态的，“记住”已处理事件中的信息**，并使用它来影响进一步的事件处理。**在Flink中，记住的信息，即状态，被本地存储在配置的状态后端中**。为了防止发生故障时丢失数据，状态后端会**定期将其内容的快照持久化到预先配置的持久存储中**。[RocksDB](https://rocksdb.org/)状态后端（即RocksDBStateBackend）是Flink中三个内置状态后端之一。这篇博文将引导您了解使用RocksDB管理应用程序状态的好处，解释何时以及如何使用它，并澄清一些常见的误解。尽管如此，这并**不是**一篇解释RocksDB如何深入工作或如何进行高级故障排除和性能调整的博客文章；如果您需要这些主题中的任何一个的帮助，可以访问[Flink用户邮件列表](https://flink.apache.org/community.html#mailing-lists)。

## 1. Flink中的状态

为了更好地理解Flink中的状态和状态后端，区分**飞行状态(in-flight state)**和**状态快照(state snapshots)**是很重要的。飞行状态，也称为**工作状态**，是Flink作业正在处理的状态。它总是本地存储在内存中（有可能溢出到磁盘），并且在作业失败时可能会丢失，而不会影响作业的可恢复性。**状态快照**，即检[查点](https://ci.apache.org/projects/flink/flink-docs-stable/ops/state/checkpoints.html)和[保存点](https://ci.apache.org/projects/flink/flink-docs-stable/ops/state/savepoints.html#what-is-a-savepoint-how-is-a-savepoint-different-from-a-checkpoint)，存储在远程持久存储器中，用于在作业失败时恢复本地状态。生产部署的适当状态后端取决于可伸缩性、吞吐量和延迟要求。

## 2. 为什么要管理状态

有状态的计算是流处理框架要实现的重要功能，因为稍复杂的流处理场景都需要记录状态，然后在新流入数据的基础上不断更新状态。下面的几个场景都需要使用流处理的状态功能：

- 数据流中的数据有重复，我们想**对重复数据去重**，需要记录哪些数据已经流入过应用，当新数据流入时，根据已流入过的数据来判断去重。
- **检查输入流是否符合某个特定的模式**，需要将之前流入的元素以状态的形式缓存下来。比如，判断一个温度传感器数据流中的温度是否在持续上升。
- **对一个时间窗口内的数据进行聚合分析**，分析一个小时内某项指标的75分位或99分位的数值。
- 在线机器学习场景下，需要根据新流入数据不断更新机器学习的模型参数。

我们知道，Flink的一个算子有多个子任务，每个子任务分布在不同实例上，我们可以把状态理解为某个算子子任务在其当前实例上的一个变量，变量记录了数据流的历史信息。当新数据流入时，我们可以结合历史信息来进行计算。**实际上，Flink的状态是由算子的子任务来创建和管理的。**一个状态更新和获取的流程如下图所示，**一个算子子任务接收输入流，获取对应的状态，根据新的计算结果更新状态。**一个简单的例子是对一个时间窗口内输入流的某个整数字段求和，那么当算子子任务接收到新元素时，会获取已经存储在状态中的数值，然后将当前输入加到状态上，并将状态数据更新。

![2021-09-06-te4Uqk](https://image.ldbmcs.com/2021-09-06-te4Uqk.jpg)

获取和更新状态的逻辑其实并不复杂，但流处理框架还需要解决以下几类问题：

- 数据的产出要保证**实时性**，延迟不能太高。
- 需要保证数据不丢不重，**恰好计算一次**（exactly one），尤其是当状态数据非常大或者应用出现故障需要恢复时，要保证状态的计算不出任何错误。
- 一般流处理任务都是7*24小时运行的，程序的**可靠性**非常高。

基于上述要求，我们不能将状态直接交由内存管理，因为内存的容量是有限制的，当状态数据稍微大一些时，就会出现内存不够的问题，这里就排除了`MemoryStateBackend`。假如我们使用一个持久化的备份系统，不断将内存中的状态备份起来，当流处理作业出现故障时，需要考虑**如何从备份中恢复**。而且，大数据应用一般是横向分布在多个节点上，流处理框架需要保证横向的**伸缩扩展性**。可见，状态的管理并不那么容易。

作为一个计算框架，Flink提供了有状态的计算，封装了一些底层的实现，比如状态的高效存储、Checkpoint和Savepoint持久化备份机制、计算资源扩缩容等问题。因为Flink接管了这些问题，开发者只需调用Flink API，这样可以更加专注于业务逻辑。

## 2. 什么是RocksDB?

认为RocksDB是一个分布式数据库，需要在集群上运行并由专门的管理员管理，这是一个常见的误解。**[RocksDB](http://rocksdb.org/)是一个用于快速存储的可嵌入持久键值存储。**它通过Java本机接口（JNI）与Flink交互。下图显示了RocksDB在Flink集群节点中的位置。以下各节将详细说明。

![2021-09-06-PqqXco](https://image.ldbmcs.com/2021-09-06-PqqXco.jpg)

## 3. Flink中的RocksDB

将RocksDB用作状态后端所需的一切都捆绑在Apache Flink发行版中，包括本机共享库：

```bash
$ jar -tvf lib/flink-dist_2.12-1.12.0.jar| grep librocksdbjni-linux64
8695334 Wed Nov 27 02:27:06 CET 2019 librocksdbjni-linux64.so
```

在运行时，RocksDB嵌入到TaskManager进程中。它在本机线程中运行并处理本地文件。例如，如果你的Flink集群中有一个配置了`RocksDBStateBendback`的作业，您将看到类似于下面的内容，其中32513是TaskManager进程ID。

```bash
$ ps -T -p 32513 | grep -i rocksdb
32513 32633 ?        00:00:00 rocksdb:low0
32513 32634 ?        00:00:00 rocksdb:high0
```

> 注意：该命令仅适用于Linux。对于其他操作系统，请参阅其文档

## 4. 什么时候使用RocksDBStateBackend

除了RocksDBStateBackend，Flink还有另外两个内置的状态后端：`MemoryStateBackend`和`FsStateBackend`。它们都是基于**堆**的，因为运行中的状态存储在JVM堆中。目前，让我们忽略`MemoryStateBackend`，因为它只用于**本地开发**和**调试**，不用于生产。

使用RocksDBStateBackend，**运行中的状态首先写入堆外/本机内存，然后在达到配置的阈值时刷新到本地磁盘**。这意味着RocksDBStateBackend可以支持大于总配置堆容量的状态。可以存储在RocksDBStateBackend中的状态量仅受整个集群中可用**磁盘空间**的限制。此外，由于RocksDBStateBackend不使用JVM堆来存储运行中的状态，因此它不受JVM垃圾收集的影响，因此具有可预测的延迟。

除了完整的、自包含的状态快照之外，RocksDBStateBackend还支持作为性能调优选项的[增量检查点](https://ci.apache.org/projects/flink/flink-docs-stable/ops/state/large_state_tuning.html#incremental-checkpoints)。**增量检查点仅存储自上次完成的检查点以来发生的更改**。与执行完整快照相比，这大大减少了检查点时间。RocksDBStateBendback是当前唯一支持增量检查点的状态后端。

在以下情况下，RocksDB是一个不错的选择：

- 作业的状态超出了本地内存的容量（例如，长时间的窗口、大的KeyedState）；
- 你正在研究增量检查点作为一种减少检查点时间的方法；
- 希望有更可预测的延迟，而不受JVM垃圾回收的影响

否则，如果应用程序的**状态很小**或需要**很低的延迟**，则应该考虑FsStateBackend。根据经验，RocksDBStateBackend比基于堆的状态后端慢几倍，因为它将键值对存储为序列化的字节。这意味着任何状态访问（读/写）都需要经过一个跨JNI边界的反序列化/序列化过程，这比直接使用堆上的状态表示更昂贵。好处是，对于相同数量的状态，与相应的堆上表示法相比，它的**内存占用率较低**。

## 5. 如何使用RocksDBStateBackend

RocksDB完全嵌入到TaskManager进程中，并完全由TaskManager进程管理。RocksDBStateBackend可以在集群级别配置为整个集群的默认值，也可以在作业级别配置为单个作业的默认值。**作业级配置优先于集群级配置**。

### 5.1 集群级别

在[`conf/flink-conf.yaml`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html)中添加以下配置:

```yaml
state.backend: rocksdb
state.backend.incremental: true
state.checkpoints.dir: hdfs:///flink-checkpoints # location to store checkpoints
```

### 5.2 作业级别

创建`StreamExecutionEnvironment`后，将以下内容添加到作业的代码中：

```java
# 'env' is the created StreamExecutionEnvironment
# 'true' is to enable incremental checkpointing
env.setStateBackend(new RocksDBStateBackend("hdfs:///fink-checkpoints", true));   
```

> 注意：除了HDFS之外，如果在[FLINK_HOME/plugins](https://blog.csdn.net/u010942041/article/details/FLINK_HOME/plugins)下添加了相应的依赖项，那么还可以使用其他本地或基于云的对象存储。

## 6. 最佳实践和高级配置

我们希望这个概述能帮助您更好地理解RocksDB在Flink中的作用，以及如何使用RocksDBStateBackend成功地运行作业。最后，我们将探讨一些最佳实践和一些参考点，以便进一步进行故障诊断和性能调优。

### 6.1 状态在RocksDB中的位置

如前所述，RocksDBStateBackend 中的运行中状态会溢出到磁盘上的文件。这些文件位于Flink配置指定的[`state.backend.rocksdb.localdir`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#state-backend-rocksdb-localdir)目录下。因为磁盘性能直接影响RocksDB的性能，所以建议将此目录放在**本地**磁盘上。不鼓励将其配置到基于网络的远程位置，如NFS或HDFS，因为写入远程磁盘通常比较慢。高可用性也不是飞行状态(in-flight state)的要求。如果需要高磁盘吞吐量，则首选本地SSD磁盘。

**状态快照将持久化到远程持久存储。**在状态快照期间，TaskManager会对飞行中的状态(in-flight state)进行快照并远程存储。将状态快照传输到远程存储完全由TaskManager本身处理，而不需要状态后端的参与。所以，[`state.checkpoints.dir`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#state-checkpoints-dir) 目录或者您在代码中为特定作业设置的参数可以是不同的位置，如本地[HDFS](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html)集群或基于云的对象存储，如[Amazon S3](https://aws.amazon.com/s3/)、[Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/)、[Google cloud Storage](https://cloud.google.com/storage)、[Alibaba OSS](https://www.alibabacloud.com/product/oss)等。

### 6.2 RocksDB故障诊断

要检查RocksDB在生产中的行为，应该查找名为LOG的RocksDB日志文件。默认情况下，此日志文件与数据文件位于同一目录中，即Flink配置指定的目录[`state.backend.rocksdb.localdir`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#state-backend-rocksdb-localdir)。启用时，[RocksDB统计信息](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide#rocksdb-statistics)也会记录在那里，以帮助诊断潜在的问题。有关更多信息，请查看[RocksDB Wiki](https://github.com/facebook/rocksdb/wiki)中的[Troubleshooting Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Troubleshooting-Guide)。如果你对RocksDB行为趋势感兴趣，可以考虑为你的Flink作业启用[RocksDB本机指标](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#rocksdb-native-metrics)。

> 注意：从Flink1.10开始，通过将日志级别设置为HEADER，RocksDB日志记录被有效地禁用。要启用它，请查看[How to get RocksDB’s LOG file back for advanced troubleshooting](https://ververica.zendesk.com/hc/en-us/articles/360015933320-How-to-get-RocksDB-s-LOG-file-back-for-advanced-troubleshooting)。

> 警告：在Flink中启用RocksDB的原生指标可能会对您的工作产生负面影响。

从Flink 1.10开始，Flink默认将RocksDB的内存分配配置为每个任务槽的托管内存(managed memory)量。改善内存相关性能问题的主要机制是通过Flink配置[`taskmanager.memory.managed.size`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#taskmanager-memory-managed-size)或[`taskmanager.memory.managed.fraction`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#taskmanager-memory-managed-fraction)增加Flink的[托管内存](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/memory/mem_setup_tm.html#managed-memory)。对于更细粒度的控制，应该首先通过设置[`state.backend.rocksdb.memory.managed`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#state-backend-rocksdb-memory-managed)为false，然后从以下Flink配置开始：[`state.backend.rocksdb.block.cache-size`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#state-backend-rocksdb-block-cache-size)（与RocksDB中的块大小相对应），[`state.backend.rocksdb.writebuffer.size`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#state-backend-rocksdb-writebuffer-size)（与RocksDB中的write_buffer_size相对应），以及[`state.backend.rocksdb.writebuffer.count`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#state-backend-rocksdb-writebuffer-count)（对应于RocksDB中的最大写入缓冲区数）。有关更多详细信息，请查看[这篇](https://www.ververica.com/blog/manage-rocksdb-memory-size-apache-flink)关于如何在Flink中管理RocksDB内存大小的文章和[RocksDB内存使用](https://github.com/facebook/rocksdb/wiki/Memory-usage-in-RocksDB)Wiki页面。

在RocksDB中写入或覆盖数据时，RocksDB线程在后台管理从内存到本地磁盘的刷新和数据压缩。在多核CPU的机器上，应该通过设置Flink配置[`state.backend.rocksdb.thread.num`](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#state-backend-rocksdb-thread-num)（对应于RocksDB中的max_background_jobs）来增加后台刷新和压缩的并行性。对于生产设置来说，默认配置通常太小。如果你的工作经常从RocksDB读取内容，那么应该考虑启用[布隆过滤器](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide#bloom-filters)。

对于其他RocksDBStateBackend配置，请查看Flink文档[Advanced RocksDB State Backends Options](https://ci.apache.org/projects/flink/flink-docs-stable/deployment/config.html#advanced-rocksdb-state-backends-options)。有关进一步的调优，请查看[RocksDB Wiki](https://github.com/facebook/rocksdb/wiki)中的[RocksDB Tuning Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide)。

## 7. 总结

[RocksDB](https://rocksdb.org/)状态后端（即RocksDBStateBackend）是Flink中捆绑的三种状态后端之一，在配置流应用程序时是一个很好的选择。它使可扩展的应用程序能够保持高达数TB的状态，并保证`exactly-once`。如果Flink作业的状态太大，无法放入JVM堆中，或者你**对增量检查点很感兴趣，或者希望有可预测的延迟，那么应该使用RocksDBStateBackend**。由于RocksDB作为本机线程嵌入到TaskManager进程中，并且可以处理本地磁盘上的文件，RocksDBStateBackend支持开箱即用，无需更多设置和管理任何外部系统或进程。

