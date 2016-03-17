---
title: 状态管理

sub-nav-parent: fault_tolerance
sub-nav-group: streaming
sub-nav-pos: 1
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

Flink 中所有的转换操作看起来都像是函数调用（函数式处理术语），但它们实际上都是有状态的操作。
通过 Flink 的状态接口（state interface）或代码中的 checkpoint 实例变量，可以将 *每一个* 转换操作，如`map`、`filter`等，
变成有状态的操作。只要实现状态相关的接口，就可以在代码中将任意的实例变量注册为 ***托管的*** 状态变量。
通过这种方式，或者使用 Flink 原生的状态接口，Flink 能够定期对状态做一致性快照，并且在失败时从快照中恢复状态数据。

最终效果是，无论处理过程中有无失败，对任意状态的更新都是一致的。


首先，我们看一下如何保证状态变量在有处理失败情况下的状态一致性，然后来看一下 Flink 的状态接口。

默认情况下，状态的 checkpoint 会被保存在 JobManager 的内存中。对于较大的状态，Flink 也支持将 checkpoint 保存在文件系统中
（如HDFS、S3或任意挂载的 POSIX 文件系统），可以通过 `flink-conf.yaml` 配置文件或者 `StreamExecutionEnvironment.setStateBackend(…)`
代码来配置如何存储checkpoint。
[state backends]({{ site.baseurl }}/apis/streaming/state_backends.html) 描述了可用的状态存储后台以及如何配置它们。

* ToC
{:toc}

## 使用 基于 Key/Value 的状态接口 

基于 Key/Value 的状态接口提供了访问当前输入元素中所有 key 值状态的方法，这也意味着，这些状态只能用于`KeyedStream`之上，
可以通过 `stream.keyBy(...)` 来创建 `KeyedStream` 。


我们首先看一下可用的状态接口列表，以及如何在代码中使用它们。
Flink 中可用的状态接口有：

* `ValueState<T>`：这个状态保存一个可被更新和获取的值，该值与输入数据的 key 对应，因此在多个 key 的情况下，对于每一个 key，
都可能会有一个不同的状态值。可以通过`update(T)`来更新状态值，并通过`T value()`来获取状态值。

* `ListState<T>`：这个状态保存了一系列输入数据。可以通过`add(T)`往状态中追加数据，并通过`Iterable<T> get`返回一个`Iterable`来遍历所有数据。

* `ReducingState<T>`：这个状态保存了一个对一系列数据进行聚合后的值。这个接口与`ListState`有点类似，但是通过`add(T)`添加的元素，会通过
指定的聚合方法`ReduceFunction`计算出一个聚合值。


所有的状态接口都有一个`clear()`方法，用于清除当前 key 所对应的状态。


需要注意的是，上述状态接口应仅用于状态交互。状态不一定保存在运行时的内存中，也可能保存在磁盘或者其他地方。
还需注意，获取到的状态值依赖于输入元素的 key 值，因此如果输入元素的 key 值不同，可能会导致对相同方法调用返回的状态值也不同。

通过创建一个`StateDescriptor`，可以得到一个包含特定名称的状态句柄（后面会提到，可以创建多个互不重名的状态句柄来引用多个不同的状态）。
状态所包含的值可能是一个用户自定义的函数，如`ReduceFunction`。根据想要获取的状态的不同，可以分别创建`ValueStateDescriptor`、
`ListStateDescriptor`或`ReducingStateDescriptor`状态句柄。

状态是通过`RuntimeContext`来访问的，因此只能在 *rich functions* 中访问状态。
[这里]({{ site.baseurl }}/apis/common/#specifying-transformation-functions) 有更详细的说明，稍后我们也将看到一个简短的示例。
`RichFunction` 中的 `RuntimeContext` 有以下获取状态的方法：

* `ValueState<T> getState(ValueStateDescriptor<T>)`
* `ReducingState<T> getReducingState(ReducingStateDescriptor<T>)`
* `ListState<T> getListState(ListStateDescriptor<T>)`

下面通过一个`FlatMapFunction`示例来演示如何使用状态相关接口：

{% highlight java %}
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * ValueState 状态，第一个字段为 count 值，第二个字段为 sum 值
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // 访问状态值
        Tuple2<Long, Long> currentSum = sum.value();

        // 更新状态的count值
        currentSum.f0 += 1;

        // 将输入值累加到sum中
        currentSum.f1 += input.f1;

        // 更新状态值
        sum.update(currentSum);

        // 如果 count 值 >=2，则输出count 及平均值，并且清空状态
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // type information
                        Tuple2.of(0L, 0L)); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}

// 然后在一个流式程序中使用上面的代码（假设已经创建了名为env的StreamExecutionEnvironment）
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
        .flatMap(new CountWindowAverage())
        .print();

// 最终执行结果将会输出：(1,4) and (1,5)
{% endhighlight %}

上面的例子实现了一个很简单的计数窗口，其中 key 值为元组的第一个元素（上面的例子中值都为1）。我们的`RichFunction`会将 count 值
以及滑动的计数值保存在`ValueState`中，一旦 count 总数达到2，就会输出平均值并且清空状态，后续状态从0开始重新累计。
注意，如果输入元组的第一个字段值不同，我们将会保存一个不同的状态值。

### Scala DataStream API 中的状态

除了上面提到的java的状态接口，Scala API 中也提供了在`keyedStream`之上针对`map()`或`flatMap()`进行`ValueState`更新的快捷操作。
用户代码首先获得`ValueState`的当前状态值（返回值为 `Option`类型），然后返回一个更新后的状态值，用于状态的更新。

{% highlight scala %}
val stream: DataStream[(String, Int)] = ...

val counts: DataStream[(String, Int)] = stream
  .keyBy(_._1)
  .mapWithState((in: (String, Int), count: Option[Int]) =>
    count match {
      case Some(c) => ( (in._1, c), Some(c + in._2) )
      case None => ( (in._1, 0), Some(in._2) )
    })
{% endhighlight %}

## 对实例变量做 checkpoint

通过`Checkpointed`接口，也可以对实例变量做 checkpoint 。

只要用户实现了`Checkpointed`接口，`snapshotState(…)` 和 `restoreState(…)`方法就会被调用，分别用于保存和恢复方法的状态。

除此之外，用户也可以实现`CheckpointNotifier`接口，这样在完成 checkpoint 的时候，会得到`notifyCheckpointComplete(long checkpointId)`
方法的回调通知。

需要注意的是，如果在 checkpoint 完成和发送通知之间发生了错误，则不能够保证用户一定会收到通知。
因此这种机制仅应被视为一种不可靠的通知，当你收到后续的 checkpoint 的通知时，它有可能是之前丢失的通知。


上面示例中的 `ValueState` 也可以用实例变量来实现：

{% highlight java %}

public class CountWindowAverage
        extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>>
        implements Checkpointed<Tuple2<Long, Long>> {

    private Tuple2<Long, Long> sum = null;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // 更新count值
        sum.f0 += 1;

        // 将输入的count值累加到sum中
        sum.f1 += input.f1;


        // 如果 count 值 >=2，则输出count 及平均值，并且清空状态
        if (sum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, sum.f1 / sum.f0));
            sum = Tuple2.of(0L, 0L);
        }
    }

    @Override
    public void open(Configuration config) {
        if (sum == null) {
            // 仅当 sum 为null时创建状态变量，因为restoreState可能先于open()方法被调用
            // 此时 sum 可能已经被设置为被恢复的状态值
            sum = Tuple2.of(0L, 0L);
        }
    }

    // 持续地持久化保存状态
    @Override
    public Serializable snapshotState(long checkpointId, long checkpointTimestamp) {
        return sum;
    }

    // 出错时恢复状态
    @Override
    public void restoreState(Tuple2<Long, Long> state) {
        sum = state;
    }
}
{% endhighlight %}

## 有状态的 Source 函数

处理有状态的 source 需要比处理其他状态更加小心一点。为了保证状态更新和输出的原子性（当需要exactly-once的处理语义时），
在做 checkpoint 时需要获取从 source context 中获取锁。 

{% highlight java %}
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements Checkpointed<Long> {

    /**  exactly-once语义中的当前 offset */
    private long offset;

    /** job 是否运行的标识 */
    private volatile boolean isRunning = true;

    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();

        while (isRunning) {
            // 由于获得了checkpoint 锁，这里的输出和状态更新是原子的
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }

    @Override
    public Long snapshotState(long checkpointId, long checkpointTimestamp) {
        return offset;

    }

    @Override
    public void restoreState(Long state) {
        offset = state;
    }
}
{% endhighlight %}

在有的操作中 可能需要知道该操作的 checkpoint 何时被 Flink 框架所确认，以便与外部进行交互。
具体可见：`flink.streaming.api.checkpoint.CheckpointNotifier` 接口。

## 在迭代Job中的状态checkpoint

Flink 目前只保证非迭代 Job 的处理语义，在迭代的Job中开启 checkpoint 将会得到一个异常。
如果需要强制在迭代 Job 中开启 checkpoint 机制，则用户在开启 checkpoint 的同时还需要设置：`env.enableCheckpointing(interval, force = true)`。

注意，在迭代的循环边界中，如果出现失败，则处理的数据以及相关状态的变化都可能丢失。

{% top %}