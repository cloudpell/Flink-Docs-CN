# 数据流容错

本文档描述了Flink的流数据流容错机制。

## 介绍

_注意：_默认情况下，禁用检查点。有关如何启用和配置检查[点](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/checkpointing.html)的详细信息，请参阅[检查](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/checkpointing.html)点。

_注意：_要使此机制实现其完全保证，数据流源（例如消息队列或代理）需要能够将流回滚到定义的最近点。[Apache Kafka](http://kafka.apache.org/)具有这种能力，Flink与Kafka的连接器利用了这种能力。有关Flink连接器提供的保证的更多信息，请参阅[数据源和接收器的容错保证](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/connectors/guarantees.html)。

_注意：_由于Flink的检查点是通过分布式快照实现的，因此我们可以互换使用_快照_和_检查点_。

## 检查点

Flink的容错机制的核心部分是绘制分布式数据流和运营商状态的一致快照。这些快照充当一致的检查点，系统可以在发生故障时将其回滚。Flink用于绘制这些快照的机制在“ [分布式数据流的轻量级异步快照](http://arxiv.org/abs/1506.08603) ”中进行了描述。它受到分布式快照的标准[Chandy-Lamport算法的](http://research.microsoft.com/en-us/um/people/lamport/pubs/chandy.pdf)启发，专门针对Flink的执行模型而定制。

### 障碍（Barriers）

Flink分布式快照的核心元素是_流障碍_。这些障碍被注入数据流并与记录一起作为数据流的一部分流动。障碍永远不会超过记录，流量严格符合要求。障碍将数据流中的记录分为进入当前快照的记录集和进入下一个快照的记录。每个障碍都携带快照的ID，该快照的记录在其前面推送。障碍不会中断流的流动，因此非常轻量级。来自不同快照的多个障碍可以同时在流中，这意味着可以同时生成各种快照。

![](../.gitbook/assets/image%20%2829%29.png)

流障碍被注入流源的并行数据流中。注入快照_n_的障碍（我们称之为S n）的点是源流中快照覆盖数据的位置。例如，在Apache Kafka中，此位置将是分区中最后一条记录的偏移量。该位置S n被报告给_检查点协调员_（Flink的JobManager）。

然后障碍物向下游流动。当中间操作符从其所有输入流中收到快照_n_的障碍时，它会将快照_n_的障碍发送到其所有输出流中。一旦接收器操作符（流式DAG的末端）从其所有输入流接收到障碍_n_，它就向快照_n_确认检查点协调器。在所有接收器确认快照后，它被视为已完成。

一旦完成了快照_n_，作业将永远不再向源请求来自S n之前的记录，因为此时这些记录（及其后代记录）将通过整个数据流拓扑。

![](../.gitbook/assets/image%20%2810%29.png)

接收多个输入流的操作符需要在快照屏障上_对齐_输入流。上图说明了这一点：

* 一旦操作符从输入流接收到快照障碍_n_，它就不能处理来自该流的任何其他记录，直到它从其他输入接收到障碍_n_为止。否则，它会混合属于快照_n的_记录和属于快照_n + 1的记录_。
* 报告障碍_n的_流暂时被搁置。从这些流接收的记录不会被处理，而是放入输入缓冲区。
* 一旦最后一个流接收到障碍_n_，操作符就会发出所有挂起的传出记录，然后自己发出快照_n个障碍_。
* 之后，它将继续处理来自所有输入流的记录，在处理来自流的记录之前处理来自输入缓冲区的记录。

### 状态

当操作符包含任何形式的_状态时_，此状态也必须是快照的一部分。操作符状态有不同的形式：

* _用户定义的状态_：这是由转换函数（如`map()`或`filter()`）直接创建和修改的状态。有关详细信息，请参阅[流应用程序](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/index.html)中的状
* _系统状态_：此状态是指作为操作符计算一部分的数据缓冲区。此状态的典型示例是_窗口缓冲区_，系统在其中收集（和聚合）窗口记录，直到窗口被评估和逐出。

当操作符从输入流中接收到所有快照障碍时，以及在将障碍发送到输出流之前，对其状态进行快照。此时，所有来自障碍之前的记录的状态更新都将完成，并且在障碍应用之后，没有依赖于记录的更新。因为快照的状态可能很大，所以它存储在可配置的[状态后端](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/state/state_backends.html)。默认情况下，这是JobManager的内存，但是对于生产使用，应该配置分布式可靠存储\(如HDFS\)。在存储状态之后，操作符确认检查点，将快照障碍发送到输出流中，然后继续。

生成的快照现在包含：

* 对于每个并行流数据源，启动快照时流中的偏移/位置
* 对于每个操作符，指向作为快照的一部分存储的状态的指针

![](../.gitbook/assets/image%20%2823%29.png)

### 完全一次与至少一次

对齐步骤可能会增加流程序的延迟。通常，这个额外的延迟大约是几毫秒，但是我们已经看到一些异常值的延迟显著增加的情况。对于所有记录都需要持续超低延迟\(几毫秒\)的应用程序，Flink有一个开关，可以跳过检查点期间的流对齐。当操作符从每个输入中看到检查点屏障时，仍然会绘制检查点快照。

当跳过对齐时，操作符将继续处理所有输入，即使在检查点n的一些检查点屏障到达之后也是如此。这样，操作符还可以在检查点n的状态快照被获取之前处理属于检查点n+1的元素。在恢复时，这些记录将以重复的形式出现，因为它们都包含在检查点n的状态快照中，并且将在检查点n之后作为数据的一部分重新播放。

_注意_：对齐仅适用于具有多个前驱（连接）的操作符以及具有多个发送方的操作符（在流重新分区/随机播放之后）。正因为如此，数据流只有在易并行的流操作（`map()`、`flatMap()`、`filter()`、…）实际上提供了完全一次保证，即使在至少一次的模式下也是如此。

### 异步状态快照

注意，上述机制意味着操作符在将_状态_的快照存储在_状态后端时_停止处理输入记录。每次拍摄快照时，此_同步_状态快照都会引入延迟。

注意，上述机制意味着操作符在将_状态_的快照存储在_状态后端时_停止处理输入记录。每次拍摄快照时，此_同步_状态快照都会引入延迟。

可以让操作符在存储其状态快照时继续处理，从而有效地让状态快照在后台_异步_发生。为此，操作符必须能够生成一个状态对象，该状态对象应以某种方式存储，以便对操作符状态的进一步修改不会影响该状态对象。例如，诸如RocksDB中使用_的写时复制_数据结构具有这种行为。

在接收到输入的检查点障碍后，操作符启动其状态的异步快照复制。它会立即释放其输出的障碍，并继续进行常规流处理。后台复制过程完成后，它会向检查点协调员（JobManager）确认检查点。检查点现在仅在所有接收器都已收到障碍并且所有有状态操作符已确认其完成备份（可能在障碍物到达接收器之后）之后才完成。

有关状态快照的详细信息，请参阅[状态后端](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/state/state_backends.html)。

## 恢复

在这种机制下的恢复是直截了当的：当失败时，Flink选择最新完成的检查点_k_。然后，系统重新部署整个分布式数据流，并为每个操作符提供作为检查点_k的_一部分进行快照的状态。设置源以开始从位置S k读取流。例如，在Apache Kafka中，这意味着告诉消费者从偏移量S k开始提取。

如果状态以递增方式进行快照，则操作符将从最新完整快照的状态开始，然后对该状态应用一系列增量快照更新。

有关更多信息，请参阅[重启策略](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/restart_strategies.html)

## 操作符快照实施

在获取操作符快照时，有两部分:**同步部分**和**异步部分**。

操作符和状态后端以Java形式提供其快照`FutureTask`。该任务包含完成_同步_部分且_异步_部分处于挂起状态的状态。然后，异步部分由该检查点的后台线程执行。

检查点纯粹同步返回已经完成的操作符`FutureTask`。如果需要执行异步操作，则以该`run()`方法执行`FutureTask`。

任务是可取消的，因此可以释放流和其他资源消耗句柄。

