整个MapReduce库将计算看作两个函数：Map、Reduce。

Map函数由用户自定义，接受输入键值对并产生中间键值对的集合。然后将相同中间键的所有值发送给Reduce函数。

Reduce函数也由用户自定义，接受一个中间键和一系列对应的值。它将这些值融合以形成更小的值集合。
计算文档中每个词出现的次数，用户可以照着下面的伪代码构造map、reduce函数

```c++
 map(String key, String value):
     // key: document name
     // value: document contents
     for each word w in value:
         EmitIntermediate(w, "1");
 ​
 reduce(String key, Iterator values):
     // key: a word
     // values: a list of counts
     int result = 0;
     for each v in values:
         result += ParseInt(v);
     Emit(AsString(result));
```

# 实现原理:

当在多台机器上调用Map时，输入数据会被自动分成M份，因此数据能够被不同的机器并行处理。然后Map可以依据hash函数将不同key的数据映射到R份中间输出中，以供Reduce使用。

![img](https://img-blog.csdnimg.cn/20210830075325310.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBASm9pbkFwcGVy,size_20,color_FFFFFF,t_70,g_se,x_16)

上图展示了一个MapReduce操作的任务流。当用户程序调用MapReduce函数时，会发生下面一系列的动作：

1:首先程序中的MapReduce 库通常会将输入的文件分割成大小为16MB ~ 64MB 的多份切片，然后在集群上启动该程序的多份副本；

2:其中一份程序副本作为 master，其余的作为 workers , 并由 master 分配任务。总共有 M 个 map tasks 以及R个 reduce tasks 需要被分配。master 会选取空闲的 worker 并分配给它们一个 map task 或者 reduce task；

3:被分配map task 的 worker 会读取相应的输入内容。然后从输入数据中解析出 key/value 键值对，并将其作为用户定义的 map 函数输入。最后通过 map 函数产生的键值对会被缓存在内存中；

4:这些缓存的键值对会周期性地写入本地磁盘， 并通过分区函数被划分到R个区域。 **这些缓存的键值对在本地磁盘上的位置会被传递回 master ，而 master 会将这些位置信息转发给 reduce workers；**

4:当 reduce worker 接收到 master 的位置信息通知时，它会使用 rpc 从 map workers的本地磁盘 上读取缓存的数据。当一个 reduce worker 读取了所有的中间数据后，它会按照中间键进行排序，以便相同 key 的键值对可以在同个分组。由于很多不同的键值对会被传递到相同的reduce task，因此需要进行排序。如果中间数据太大的话，可以使用外部排序；

5:reduce worker 会对已经排序的中间数据进行迭代，当遇到到每个唯一的中间键时，它会传递 key 和 对应的中间值集合到用户的reduce函数中。reduce函数的输出结果会附加到这个 reduce 分区的最终输出文件中；

6:当所有 map tasks 和 reduce tasks 都完成后，master 会唤醒用户程序。这时，用户程序中的 MapReduce 调用完成并返回到用户代码；

# master的数据结构

对于每个map task和reduce task而言，它存储了任务状态（空闲，正在进行中或者完成），以及任务对应的worker节点信息（针对非空闲任务）。

master将中间文件的位置信息从 map tasks 传递到 reduce tasks 。因此对于每个完成的 map task，master 都会存储R 个由 map task 产生的中间文件位置和大小。当map task 完成时，master 会收到位置和大小的更新信息，而这些消息会增量式地被推送给正在执行 reduce tasks的workers。


# 错误处理

## **3.3.1 Worker Failure**

master 会周期性地 ping worker 节点。如果在一定时间内没有收到来自worker 节点的响应，master 会标记这个 worker 节点已经失败。任何执行完成 的map tasks 都会被重置到初始的 idle 状态，此时这些 task 可以在其他 worker 上被再次调度。同样地，任何执行失败的 map tasks 或者 reduce tasks 也会被重置到 idle 状态，并且可以被重新调度。

当 map tasks 的输出存储的硬盘损坏时，已经完成的 map tasks 会被重新执行。已经完成的 reduce tasks 不需要重新执行，因为 reduce tasks 的输出会存储在全局的文件系统中。

当一个 map task 先在 worker A 上执行，然后由于A失败又在worker B 执行时，所有执行 reduce task 的 worker 都会被通知重新执行。任何一个还没有从 worker A 上读取数据的 reduce task 都会从 worker B上读取数据。

MapReduce 对于大规模的 worker 故障有恢复能力。例如，在一次MapReduce操作期间，由于网络维护，导致正在运行的集群中有80个机器在几分钟内无法访问。MapReduce master 服务器只是简单地重新执行了无法访问的机器所完成的工作，最终完成 MapReduce 操作。


## **3.3.2 Master Failure**

我们可以让 master 服务器定期将上述描述的数据结构写入检查点。如果 master 宕机，则可以从最后一个检查点状态进行恢复。但由于只有一个 master 服务器，因此发生故障的可能性不大。因此，如果 master 宕机，我们当前的实现将中止 MapReduce 计算。client 可以检查这种情况，并根据需要重试MapReduce 操作。

## **3.3.3 Semantics in the Presence of Failures**

当用户提供的 map 和 reduce算子是其输入值的确定性函数时，我们的分布式实现的输出与整个程序的无故障顺序执行的输出相同。

### 3.4 Locality

在我们的计算环境中，网络带宽是相对稀缺的资源。我们将输入数据存储在组成集群的机器本地磁盘上，从而节省了网络带宽。

### 3.5 Task Granularity

我们将 map 阶段划分为 M 个子阶段，将 reduce 阶段划分为R个子阶段。理想情况下，**M和R应该比worker机器的数量大得多**。让每个 worker 执行许多不同的任务以改善动态负载均衡，并且还可以在 worker 发生故障时加快恢复速度：许多完成的 map 任务可以传播到所有其他 worker 机器上。

此外，R 通常受用户约束，因为每个reduce任务的输出最终都存储在单独的输出文件中。在实践中，我们倾向于选择M，以便每个单独的map任务大约是16 MB到64 MB的输入数据（这样上述的局部优化是最有效的），并且我们将R设置为我们期望使用的worker机器数量乘以一个小小的放大因子。我们经常使用2000个 worker 机器执行M = 200000和 R = 5000 的MapReduce计算。

### 3.6 Backup Tasks

MapReduce 操作总时间拖延的常见原因之一是“**拖尾**”现象：一台机器花费异常长的时间来完成最后几个 map 或者 reduce 任务。拖尾之所以出现，可能是由于多种原因。例如，磁盘损坏的机器可能会经常遇到可纠正的错误，从而将其读取性能从30 MB/s降低到1 MB/s。集群调度系统可能在机器上调度了其他任务，由于竞争CPU、内存、本地磁盘或网络带宽，导致其执行MapReduce代码的速度较慢。我们最近遇到的一个问题是机器初始化代码中的一个错误，该错误导致了处理器缓存被禁用：受影响机器上的计算速度降低了一百倍。

我们有一个通用的机制来减轻拖尾的影响。**当MapReduce操作接近完成时，主服务器会调度其余正在进行的任务进行备份执行。每当主执行或备份执行完成时，该任务就会标记为已完成。**我们已经对该机制进行了调整，以使该操作使用的计算资源增加不超过百分之几。我们发现这大大减少了完成大型MapReduce操作的时间。

## 4. Refinements

### 4.1 Partitioning Function

MapReduce 的用户指定他们想要的 reduce tasks /输出文件的数量 (R)。通过对中间键按照分区函数分区，可以将数据分配给这些任务。MapReduce 提供了默认的分区函数 hash(key) mod R ，这往往会使得分区相当均衡。但是，在某些情况下，通过其他方式对数据进行分区也很有用。例如，有时输出键是URL，而我们希望单个主机的所有条目都以同一输出文件结尾。为了支持这种情况，MapReduce库的用户可以提供特殊的分区功能。例如，使用 hash( Hostname(urlkey) ) mod R 作为分区函数可以使来自同一主机的所有URL最终都位于同一输出文件中。
