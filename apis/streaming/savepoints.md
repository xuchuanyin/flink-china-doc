---
title: "Savepoints"
is_beta: false
sub-nav-group: streaming
sub-nav-pos: 5
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

-Programs written in the [Data Stream API](index.html) can resume execution from a **savepoint**. Savepoints allow both updating your programs and your Flink cluster without losing any state. This page covers all steps to trigger, restore, and dispose savepoints. For more details on how Flink handles state and failures, check out the [State in Streaming Programs](state_backends.html) and [Fault Tolerance](fault_tolerance.html) pages.
+程序写在 [Data Stream API](index.html) 可以从一个 **savepoint** 恢复执行。Savepoints 允许你更新程序，并且 Flink 集群不会丢失任何的运行状态数据。当前页面包含了所有到触发、还原、处理保存点的步骤。如果想了解更多关于 Flink 处理中间状态和错误的细节，请检出 [State in Streaming Programs](state_backends.html) 和 [Fault Tolerance](fault_tolerance.html) 页面。

* toc
{:toc}

## Overview

-Savepoints are **manually triggered checkpoints**, which take a snapshot of the program and write it out to a state backend. They rely on the regular checkpointing mechanism for this. During execution programs are periodically snapshotted on the worker nodes and produce checkpoints. For recovery only the last completed checkpoint is needed and older checkpoints can be safely discarded as soon as a new one is completed.
+Savepoints 称为 **手动触发检查点**，它会对程序做一个镜像，并将它写出到带状态的后端。为此他们依靠定期检查点机制。在运行中，程序会定期在工作节点上做快照，并产生一些检查点。恢复时，只需要一个最近完成的检查点，在新检查点完成时，可以安全的删除旧的检查点。

-Savepoints are similar to these periodic checkpoints except that they are **triggered by the user** and **don't automatically expire** when newer checkpoints are completed.
+Savepoints 和定期检查点是十分相似的，不同点是当新检查点完成之后，他们是 **用户触发** 和 **不自动过期** 。

<img src="fig/savepoints-overview.png" class="center" />

-In the above example the workers produce checkpoints **c<sub>1</sub>**, **c<sub>2</sub>**, **c<sub>3</sub>**, and **c<sub>4</sub>** for job *0xA312Bc*. Periodic checkpoints **c<sub>1</sub>** and **c<sub>3</sub>** have already been *discarded* and **c<sub>4</sub>** is the *latest checkpoint*. **c<sub>2</sub> is special**. It is the state associated with the savepoint **s<sub>1</sub>** and has been triggered by the user and it doesn't expire automatically (as c<sub>1</sub> and c<sub>3</sub> did after the completion of newer checkpoints).
+在以下的例子中，工作节点为  *0xA312Bc* 创建了以下检查点：**c<sub>1</sub>**, **c<sub>2</sub>**, **c<sub>3</sub>**, 和 **c<sub>4</sub>** 。定期的检查点 **c<sub>1</sub>** 和 **c<sub>3</sub>** 已经被 *丢弃* ，检查点 **c<sub>4</sub>** 是 *最新的检查点*。 **c<sub>2</sub> 是特殊的检查点**. 和保存点相关的检查点 **s<sub>1</sub>** 已经被用户触发，然而并没有自动过期 (当 c<sub>1</sub> 和 c<sub>3</sub> 这两个检查点完成后新建检查点后才去执行过期操作)。

-Note that **s<sub>1</sub>** is only a **pointer to the actual checkpoint data c<sub>2</sub>**. This means that the actual state is *not copied* for the savepoint and periodic checkpoint data is kept around.
+需要注意的是 **s<sub>1</sub>** 只是一个 **真实检查点 c<sub>2</sub> 的指针**。也就意味着 savepoints 的位置是不可复制的，定期的数据检查点一直存在于它的周围。

## Configuration

-Savepoints point to regular checkpoints and store their state in a configured [state backend](state_backends.html). Currently, the supported state backends are **jobmanager** and **filesystem**. The state backend configuration for the regular periodic checkpoints is **independent** of the savepoint state backend configuration. Checkpoint data is **not copied** for savepoints, but points to the configured checkpoint state backend.
+Savepoints 指向周期性的检查点，并按照 [state backend](state_backends.html) 的配置保存位置信息。当前，**jobmanager** 和 **filesystem** 用以支撑后台。savepoint 后台配置对于定期检查后台配置状态是 **independent**  的。检查点数据并不是 savepoint 的 **副本** ，但却指向检查点的后台配置状态。

### JobManager

-This is the **default backend** for savepoints.
+JobManager 是 savepoints 的 **默认后台**。

-Savepoints are stored on the heap of the job manager. They are *lost* after the job manager is shut down. This mode is only useful if you want to *stop* and *resume* your program while the **same cluster** keeps running. It is *not recommended* for production use. Savepoints are *not* part of the [job manager's highly available]({{ site.baseurl }}/setup/jobmanager_high_availability.html) state.
+Savepoints 保存在 job manager 的堆内存中。当 job manager 停止后， Savepoints 也会 *丢失*。这种模式下，在运行的 **same cluster** 上执行 *stop* 和 *resume* 才会生效。这种方式 *不推荐* 在生产环境使用。 Savepoints 并 *不* 是 [job manager's highly available]({{ site.baseurl }}/setup/jobmanager_high_availability.html) 状态的一部分。

<pre>
savepoints.state.backend: jobmanager
</pre>

-**Note**: If you don't configure a specific state backend for the savepoints, the jobmanager backend will be used.
+**Note**: 不配置 savepoints 的后台状态， jobmanager 就会启用。

### File system

-Savepoints are stored in the configured **file system directory**. They are available between cluster instances and allow to move your program to another cluster.
+Savepoints 存储在配置项 **file system directory** 中。他们在集群实例中是可用的，并且允许移动程序到其他的集群。

<pre>
savepoints.state.backend: filesystem
savepoints.state.backend.fs.dir: hdfs:///flink/savepoints
</pre>

-**Note**: If you don't configure a specific directory, the job manager backend will be used.
+**Note**: 如果不单独指定目录， job manager 后台程序会默认启用。

-**Important**: A savepoint is a pointer to a completed checkpoint. That means that the state of a savepoint is not only found in the savepoint file itself, but also needs the actual checkpoint data (e.g. in a set of further files). Therefore, using the *filesystem* backend for savepoints and the *jobmanager* backend for checkpoints does not work, because the required checkpoint data won't be available after a job manager restart.
+**重要项**: 一个 savepoint 是完成检查点的一个指针。意味着， savepoint 的状态并不只会在他自己的文件中找到，但是童谣需要真实的检查点数据 (e.g. in a set of further files)。毕竟，savepoints 配置了 *filesystem* 后台后， 检查点的 *jobmanager* 并不会工作。因为必需的检查点数据在 job manager 重启后不可用。

## Changes to your program

-Savepoints **work out of the box**, but it is **highly recommended** that you slightly adjust your programs in order to be able to work with savepoints in future versions of your program.
+Savepoints **work out of the box**， 但是强烈推荐你细微的调整程序并按照保存点的方式运行，这样可以试用未来的版本。

<img src="fig/savepoints-program_ids.png" class="center" />

-For savepoints **only stateful tasks matter**. In the above example, the source and map tasks are stateful whereas the sink is not stateful. Therefore, only the state of the source and map tasks are part of the savepoint.
+关于 savepoints **only stateful tasks matter**. 在下面的例子中，source 和 map tasks 是有状态的，但 sink是无状态的。毕竟，只有source 和 map tasks 的状态属于 savepoint 的一部分。

-Each task is identified by its **generated task IDs** and **subtask index**. In the above example the state of the source (**s<sub>1</sub>**, **s<sub>2</sub>**) and map tasks (**m<sub>1</sub>**, **m<sub>2</sub>**) is identified by their respective task ID (*0xC322EC* for the source tasks and *0x27B3EF* for the map tasks) and subtask index. There is no state for the sinks (**t<sub>1</sub>**, **t<sub>2</sub>**). Their IDs therefore do not matter.
+每个任务都有自己的 **自增id** 和 **子任务索引**。在下面的例子中，source (**s<sub>1</sub>**, **s<sub>2</sub>**) 的状态和map tasks (**m<sub>1</sub>**, **m<sub>2</sub>**) 都以他们的任务id(*0xC322EC* for the source tasks and *0x27B3EF* for the map tasks)进行标识，sinks (**t<sub>1</sub>**, **t<sub>2</sub>**)并没有状态，他们的id没有意义。

-<span class="label label-danger">Important</span> The IDs are generated **deterministically** from your program structure. This means that as long as your program does not change, the IDs do not change. **The only allowed changes are within the user function, e.g. you can change the implemented `MapFunction` without changing the topology**. In this case, it is straight forward to restore the state from a savepoint by mapping it back to the same task IDs and subtask indexes. This allows you to work with savepoints out of the box, but gets problematic as soon as you make changes to the topology, because they result in changed IDs and the savepoint state cannot be mapped to your program any more.
+<span class="label label-danger">重点</span> 这些 ID 是从你的程序结构中以确定性的方式自动生成的。意味着不管程序在长时间内有没有改变， ID 都不会改变。 **唯一的改变是包含在用户方法中的，例如：你可以修改实现 `MapFunction` 而没有修改拓扑结构**。 在这个例子中，它直接从 savepoint 保存状态，通过映射他们返回相同的 ID 和 子任务。这允许你在程序之外工作，但是可以通过改变拓扑结构尽快找到问题，因为他们会导致更改 ID 而且 savepoint 状态将不在映射到你的程序上。

-<span class="label label-info">Recommended</span> In order to be able to change your program and **have fixed IDs**, the *DataStream* API provides a method to manually specify the task IDs. Each operator provides a **`uid(String)`** method to override the generated ID. The ID is a String, which will be deterministically hashed to a 16-byte hash value. It is **important** that the specified IDs are **unique per transformation and job**. If this is not the case, job submission will fail.
+<span class="label label-info">推荐配置</span> 为了修改程序和 **修复 IDs**， *DataStream* 的api提供了借口用以手动指定任务ID。每个操作都提供了 **`uid(String)`** 方法覆盖掉自动生成的 ID。ID 是 String 类型，它被哈希为一个16位哈希值。**重要** 的是：指定的 IDs 在**每个transformation和job都是唯一的**。如果不是这种情况，任务提交会失败。
{% highlight scala %}
DataStream<String> stream = env.
  // Stateful source (e.g. Kafka) with ID
  .addSource(new StatefulSource())
  .uid("source-id")
  .shuffle()
  // The stateful mapper with ID
  .map(new StatefulMapper())
  .uid("mapper-id")

// Stateless sink (no specific ID required)
stream.print()
{% endhighlight %}

## Command-line client 

-You control the savepoints via the [command line client]({{site.baseurl}}/apis/cli.html#savepoints).
+你可以控制 savepoints 通过 [command line client]({{site.baseurl}}/apis/cli.html#savepoints).

## Current limitations

- **Parallelism**: When restoring a savepoint, the parallelism of the program has to match the parallelism of the original program from which the savepoint was drawn. There is no mechanism to re-partition the savepoint's state yet.
+ **Parallelism**: 恢复 savepoint 时，程序的并发必须和原来的 savepoint 已经完成的程序一致。savepoint 没有重分区的机制。

- **Chaining**: Chained operators are identified by the ID of the first task. It's not possible to manually assign an ID to an intermediate chained task, e.g. in the chain `[  a -> b -> c ]` only **a** can have its ID assigned manually, but not **b** or **c**. To work around this, you can [manually define the task chains](index.html#task-chaining-and-resource-groups). If you rely on the automatic ID assignment, a change in the chaining behaviour will also change the IDs.
+ **Chaining**: 连续的操作按照第一个任务的 ID 进行确认。并不存在为中间的连续任务手工指定 ID 的可能性，例如：在操作 `[  a -> b -> c ]` 中，只有 **a** 可以手动指定它的 ID, **b** 和 **c** 则不行。要解决这种情况可以参考 [manually define the task chains](index.html#task-chaining-and-resource-groups)。如果依赖自动的 ID 分配，连续操作的行为会改变 IDs。

- **Disposing custom state handles**: Disposing an old savepoint does not work with custom state handles (if you are using a custom state backend), because the user code class loader is not available during disposal.
- **Disposing custom state handles**: 处理旧的 savepoint 不能自定义状态句柄（如果启用了自定义状态后端），因为在处置过程中，用户的代码加载器不可用。
