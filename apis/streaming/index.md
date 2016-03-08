---
title: "Flink DataStream API 编程指南"

# Top-level navigation
top-nav-group: apis
top-nav-pos: 2
top-nav-title: <strong>Streaming 指南</strong> (DataStream API)

# Sub-level navigation
sub-nav-group: streaming
sub-nav-group-title: Streaming 指南
sub-nav-pos: 1
sub-nav-title: DataStream API
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


Flink 中的 DataStream API 是对数据流进行转换操作（例如，过滤、更新状态、定义窗口、聚合）常用的方式。数据流可以从各种源（例如，消息队列、socket流、文件）创建而来。结果通过 sinks 操作返回，例如可能是将数据写入到文件，或者到标准输出（如命令行窗口）。Flink 程序可以运行在多样的环境下，standalone集群，或者嵌入其他程序中。执行过程可以发生在本地，也可以是由许多机器构成的集群上。

Please see [basic concepts]({{ site.baseurl }}/apis/common/index.html) for an introduction
to the basic concepts of the Flink API.

In order to create your own Flink DataStream program, we encourage you to start with
[anatomy of a Flink Program]({{ site.baseurl }}/apis/common/index.html#anatomy-of-a-flink-program)
and gradually add your own
[transformations](#datastream-transformations). The remaining sections act as references for additional
operations and advanced features.

请先阅读[基本概念]({{ site.baseurl }}/apis/common/index.html)了解下 Flink API 的基本概念。

为了创建你的第一个 Flink DataStream 程序，我们鼓励你从 [剖析 Flink 程序]({{ site.baseurl }}/apis/common/index.html#anatomy-of-a-flink-program) 开始，然后逐步地增加你的 [转换操作](#datastream-transformations)。而剩余的章节主要作为额外操作(operations)和高级特性的一个参考。


* This will be replaced by the TOC
{:toc}


样例程序
---------------

下面这段程序是一个完整的，可运行的，基于流和窗口的 word count 应用样例。从一个网络socket中以5秒的窗口统计单词数量。你可以复制 &amp; 粘贴这段代码，然后在本地跑一跑。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(0)
                .timeWindow(Time.seconds(5))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time

object WindowWordCount {
  def main(args: Array[String]) {

    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val text = env.socketTextStream("localhost", 9999)

    val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
      .map { (_, 1) }
      .keyBy(0)
      .timeWindow(Time.seconds(5))
      .sum(1)

    counts.print

    env.execute("Window Stream WordCount")
  }
}
{% endhighlight %}
</div>

</div>

要在本地运行本样例程序，需要先从终端启动 netcat 作为输入流：

~~~bash
nc -lk 9999
~~~


然后在终端中敲一些单词进去，这些会作为 word count 程序的输入。如果你想要看到统计值大于1的，在5秒内不断地敲入一样的单词（如果你手速没这么快，可以将5秒的窗口调大&#9786;）。

{% top %}

DataStream 转换（Transformations）
--------------------------

数据的转换操作可以讲一个或多个 DataStream 转换成一个新的 DataStream。程序可以合并多个转换操作为复杂的拓扑结构。

本节对所有可用的转换操作做了个简单描述。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
          <td><strong>Map</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>输入一个元素，生成另一个元素，元素类型不变。一个将输入流中的值双倍返回的 map 函数：</p>
{% highlight java %}
DataStream<Integer> dataStream = //...
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value;
    }
});
{% endhighlight %}
          </td>
        </tr>

        <tr>
          <td><strong>FlatMap</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>输入一个元素，生成零个、一个或者多个元素。一个将句子切分成多个单词的 flatmap 函数：</p>
{% highlight java %}
dataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out)
        throws Exception {
        for(String word: value.split(" ")){
            out.collect(word);
        }
    }
});
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Filter</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>对每个元素执行一个布尔函数，只保留返回 true 的元素。一个过滤掉零值的 filter 函数：</p>
{% highlight java %}
dataStream.filter(new FilterFunction<Integer>() {
    @Override
    public boolean filter(Integer value) throws Exception {
        return value != 0;
    }
});
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>KeyBy</strong><br>DataStream &rarr; KeyedStream</td>
          <td>
            <p>将流逻辑分区成不相交的分区，每个分区包含相同 key 的元素。内部是用 hash 分区来实现的。查阅 <a href="#specifying-keys">keys</a> 了解如何指定 keys。这个转换返回了一个 KeyedDataStream。</p>
{% highlight java %}
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Reduce</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>在一个 KeyedStream 上“滚动” reduce 。合并当前元素与上一个被 reduce 的值，然后输出新的值。注意三者的类型是一致的。
              <br/><br/>
              一个构造局部求和流的 reduce 函数：</p>
{% highlight java %}
keyedStream.reduce(new ReduceFunction<Integer>() {
    @Override
    public Integer reduce(Integer value1, Integer value2)
    throws Exception {
        return value1 + value2;
    }
});
{% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Fold</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
          <p>在一个 KeyedStream 上基于一个初始值“滚动”折叠。合并当前元素和上一个被折叠的值，然后输出新值。注意 Fold 的输入值与返回值类型可以不一致。
          <br/>
          <br/>
          <p>需要将序列 (1,2,3,4,5) 转换成 "start-1", "start-1-2", "start-1-2-3", ... 的一个 fold 函数长这个样子：</p>
{% highlight java %}
DataStream<String> result =
  keyedStream.fold("start", new FoldFunction<Integer, String>() {
    @Override
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
  });
{% endhighlight %}
          </p>
          </td>
        </tr>
        <tr>
          <td><strong>Aggregations</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>在一个 KeyedStream 上滚动聚合。min 与 minBy 的区别是 min 返回了最小值，而 minBy 返回了在这个字段上是最小值的所有元素（max 和 maxBy 也是同样的）。</p>
{% highlight java %}
keyedStream.sum(0);
keyedStream.sum("key");
keyedStream.min(0);
keyedStream.min("key");
keyedStream.max(0);
keyedStream.max("key");
keyedStream.minBy(0);
keyedStream.minBy("key");
keyedStream.maxBy(0);
keyedStream.maxBy("key");
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window</strong><br>KeyedStream &rarr; WindowedStream</td>
          <td>
            <p>窗口可以被定义在已经被分区的 KeyedStreams 上。窗口会对数据的每一个 key 根据一些特征（例如，在最近 5 秒中内到达的数据）进行分组。查阅<a href="windows.html">窗口</a>了解关于窗口的完整描述。
{% highlight java %}
dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
{% endhighlight %}
        </p>
          </td>
        </tr>
        <tr>
          <td><strong>WindowAll</strong><br>DataStream &rarr; AllWindowedStream</td>
          <td>
              <p>窗口可以被定义在 DataStream 上。窗口会对所有数据流事件根据一些特征（例如，在最近 5 秒中内到达的数据）进行分组。查阅<a href="windows.html">窗口</a>了解关于窗口的完整描述。</p>
              <p><strong>警告:</strong> 这在许多案例中这是一种<strong>非并行</strong>的转换。所有的记录都会被聚集到一个执行 WindowAll 操作的 task 中，这是非常影响性能的。</p>
{% highlight java %}
dataStream.windowAll(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Apply</strong><br>WindowedStream &rarr; DataStream<br>AllWindowedStream &rarr; DataStream</td>
          <td>
            <p>应用一个一般的函数到窗口上，窗口中的数据会作为一个整体被计算。下面的函数手工地计算了一个窗口中的元素总和。
            </p>
            <p><strong>注意:</strong> 如果你正在使用一个 WindowAll 的转换，你需要用 AllWindowFunction 来替换。</p>
{% highlight java %}
windowedStream.apply (new WindowFunction<Tuple2<String,Integer>, Integer, Tuple, Window>() {
    public void apply (Tuple tuple,
            Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});

// applying an AllWindowFunction on non-keyed window stream
allWindowedStream.apply (new AllWindowFunction<Tuple2<String,Integer>, Integer, Window>() {
    public void apply (Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Reduce</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>应用一个 reduce 函数到窗口上，返回 reduce 后的值。</p>
{% highlight java %}
windowedStream.reduce (new ReduceFunction<Tuple2<String,Integer>() {
    public Tuple2<String, Integer> reduce(Tuple2<String, Integer> value1, Tuple2<String, Integer> value2) throws Exception {
        return new Tuple2<String,Integer>(value1.f0, value1.f1 + value2.f1);
    }
};
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Fold</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>应用一个 fold 函数到窗口上，然后返回折叠后的值。
            在窗口上将序列 (1,2,3,4,5) 转换成 "start-1", "start-1-2", "start-1-2-3", ... 的一个 fold 函数长这个样子：
            </p>
{% highlight java %}
windowedStream.fold("start-", new FoldFunction<Integer, String>() {
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
};
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Aggregations on windows</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>聚合一个窗口中的内容。min 与 minBy 的区别是 min 返回了最小值，而 minBy 返回了在这个字段上是最小值的所有元素（max 和 maxBy 也是同样的）。</p>
{% highlight java %}
windowedStream.sum(0);
windowedStream.sum("key");
windowedStream.min(0);
windowedStream.min("key");
windowedStream.max(0);
windowedStream.max("key");
windowedStream.minBy(0);
windowedStream.minBy("key");
windowedStream.maxBy(0);
windowedStream.maxBy("key");
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Union</strong><br>DataStream* &rarr; DataStream</td>
          <td>
            <p>Union 两个或多个数据流，生成一个新的包含了来自所有流的所有数据的数据流。注意：如果你将一个数据流与其自身进行了合并，在结果流中对于每个元素你都会拿到两份。</p>
{% highlight java %}
dataStream.union(otherStream1, otherStream2, ...);
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Join</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>在一个给定的 key 和窗口上 join 两个数据流。</p>
{% highlight java %}
dataStream.join(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new JoinFunction () {...});
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window CoGroup</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>在一个给定的 key 和窗口上 co-group 两个数据流。</p>
{% highlight java %}
dataStream.coGroup(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new CoGroupFunction () {...});
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Connect</strong><br>DataStream,DataStream &rarr; ConnectedStreams</td>
          <td>
            <p>“连接”两个数据流并保持原先的类型。Connect 可以让两条流之间共享状态。</p>
{% highlight java %}
DataStream<Integer> someStream = //...
DataStream<String> otherStream = //...

ConnectedStreams<Integer, String> connectedStreams = someStream.connect(otherStream);
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>CoMap, CoFlatMap</strong><br>ConnectedStreams &rarr; DataStream</td>
          <td>
            <p>在一个 ConnectedStreams 上做类似 map 和 flatMap 的操作。</p>
{% highlight java %}
connectedStreams.map(new CoMapFunction<Integer, String, Boolean>() {
    @Override
    public Boolean map1(Integer value) {
        return true;
    }

    @Override
    public Boolean map2(String value) {
        return false;
    }
});
connectedStreams.flatMap(new CoFlatMapFunction<Integer, String, String>() {

   @Override
   public void flatMap1(Integer value, Collector<String> out) {
       out.collect(value.toString());
   }

   @Override
   public void flatMap2(String value, Collector<String> out) {
       for (String word: value.split(" ")) {
         out.collect(word);
       }
   }
});
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Split</strong><br>DataStream &rarr; SplitStream</td>
          <td>
            <p>
            根据具体的标准切分数据流成两个或多个流。
{% highlight java %}
SplitStream<Integer> split = someDataStream.split(new OutputSelector<Integer>() {
    @Override
    public Iterable<String> select(Integer value) {
        List<String> output = new ArrayList<String>();
        if (value % 2 == 0) {
            output.add("even");
        }
        else {
            output.add("odd");
        }
        return output;
    }
});
{% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Select</strong><br>SplitStream &rarr; DataStream</td>
          <td>
            <p>
                从一个 SplitStream 中选出一个或多个流。
{% highlight java %}
SplitStream<Integer> split;
DataStream<Integer> even = split.select("even");
DataStream<Integer> odd = split.select("odd");
DataStream<Integer> all = split.select("even","odd");
{% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Iterate</strong><br>DataStream &rarr; IterativeStream &rarr; DataStream</td>
          <td>
            <p>
                在流(flow)中创建一个带反馈的循环，通过重定向一个 operator 的输出到之前的 operator。这对于定义一些需要不断更新模型的算法是非常有帮助的。下面这段代码对一个流不断地应用迭代体。大于 0 的元素会被发送到反馈通道，剩余的元素会继续发往下游。查阅 <a href="#iterations">迭代</a> 了解完整的描述。

{% highlight java %}
IterativeStream<Long> iteration = initialStream.iterate();
DataStream<Long> iterationBody = iteration.map (/*do something*/);
DataStream<Long> feedback = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Integer value) throws Exception {
        return value > 0;
    }
});
iteration.closeWith(feedback);
DataStream<Long> output = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Integer value) throws Exception {
        return value <= 0;
    }
});
{% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>提取时间戳</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>
                为了能够工作于使用 event time 语义的窗口，需要从记录中提取时间戳。查阅 <a href="{{ site.baseurl }}/apis/streaming/time.html">working with time</a> 了解更多。 
                
{% highlight java %}
stream.assignTimestamps (new TimeStampExtractor() {...});
{% endhighlight %}
            </p>
          </td>
        </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">


<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 25%">转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
    <tr>
          <td><strong>Map</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>输入一个元素，生成另一个元素，元素类型不变。一个将输入流中的值双倍返回的 map 函数：</p>
{% highlight scala %}
dataStream.map { x => x * 2 }
{% endhighlight %}
          </td>
        </tr>

        <tr>
          <td><strong>FlatMap</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>输入一个元素，生成零个、一个或者多个元素。一个将句子切分成多个单词的 flatmap 函数：</p>
{% highlight scala %}
dataStream.flatMap { str => str.split(" ") }
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Filter</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>对每个元素执行一个布尔函数，只保留返回 true 的元素。一个过滤掉零值的 filter 函数：
            </p>
{% highlight scala %}
dataStream.filter { _ != 0 }
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>KeyBy</strong><br>DataStream &rarr; KeyedStream</td>
          <td>
            <p>将流逻辑分区成不相交的分区，每个分区包含相同 key 的元素。内部是用 hash 分区来实现的。查阅 <a href="#specifying-keys">keys</a> 了解如何指定 keys。这个转换返回了一个 KeyedDataStream。</p>
{% highlight scala %}
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Reduce</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>在一个 KeyedStream 上“滚动” reduce 。合并当前元素与上一个被 reduce 的值，然后输出新的值。注意三者的类型是一致的。 
                    <br/>
              <br/>
            一个构造局部求和流的 reduce 函数：</p>
{% highlight scala %}
keyedStream.reduce { _ + _ }
{% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Fold</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
          <p>在一个 KeyedStream 上基于一个初始值“滚动”折叠。合并当前元素和上一个被折叠的值，然后输出新值。注意 Fold 的输入值与返回值类型可以不一致。 
          <br/>
          <br/>
          <p>需要将序列 (1,2,3,4,5) 转换成 "start-1", "start-1-2", "start-1-2-3", ... 的一个 fold 函数长这个样子：</p>
{% highlight scala %}
val result: DataStream[String] =
    keyedStream.fold("start", (str, i) => { str + "-" + i })
{% endhighlight %}
          </p>
          </td>
        </tr>
        <tr>
          <td><strong>Aggregations</strong><br>KeyedStream &rarr; DataStream</td>
          <td>
            <p>在一个 KeyedStream 上滚动聚合。min 与 minBy 的区别是 min 返回了最小值，而 minBy 返回了在这个字段上是最小值的所有元素（max 和 maxBy 也是同样的）。</p>
{% highlight scala %}
keyedStream.sum(0)
keyedStream.sum("key")
keyedStream.min(0)
keyedStream.min("key")
keyedStream.max(0)
keyedStream.max("key")
keyedStream.minBy(0)
keyedStream.minBy("key")
keyedStream.maxBy(0)
keyedStream.maxBy("key")
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window</strong><br>KeyedStream &rarr; WindowedStream</td>
          <td>
            <p>窗口可以被定义在已经被分区的 KeyedStreams 上。窗口会对数据的每一个 key 根据一些特征（例如，在最近 5 秒中内到达的数据）进行分组。查阅<a href="windows.html">窗口</a>了解关于窗口的完整描述。
            
{% highlight scala %}
dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))) // Last 5 seconds of data
{% endhighlight %}
        </p>
          </td>
        </tr>
        <tr>
          <td><strong>WindowAll</strong><br>DataStream &rarr; AllWindowedStream</td>
          <td>
              <p>窗口可以被定义在 DataStream 上。窗口会对所有数据流事件根据一些特征（例如，在最近 5 秒中内到达的数据）进行分组。查阅<a href="windows.html">窗口</a>了解关于窗口的完整描述。
              </p>
              <p><strong>警告:</strong> 这在许多案例中这是一种<strong>非并行</strong>的转换。所有的记录都会被聚集到一个执行 WindowAll 操作的 task 中，这是非常影响性能的。
              </p>
{% highlight scala %}
dataStream.windowAll(TumblingEventTimeWindows.of(Time.seconds(5))) // Last 5 seconds of data
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Apply</strong><br>WindowedStream &rarr; DataStream<br>AllWindowedStream &rarr; DataStream</td>
          <td>
            <p>应用一个一般的函数到窗口上，窗口中的数据会作为一个整体被计算。下面的函数手工地计算了一个窗口中的元素总和。</p>
            <p><strong>注意:</strong> 如果你正在使用一个 WindowAll 的转换，你需要用 AllWindowFunction 来替换。</p>
{% highlight scala %}
windowedStream.apply { WindowFunction }

// applying an AllWindowFunction on non-keyed window stream
allWindowedStream.apply { AllWindowFunction }

{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Reduce</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>应用一个 reduce 函数到窗口上，返回 reduce 后的值。</p>
{% highlight scala %}
windowedStream.reduce { _ + _ }
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Fold</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>应用一个 fold 函数到窗口上，然后返回折叠后的值。 在窗口上将序列 (1,2,3,4,5) 转换成 "start-1", "start-1-2", "start-1-2-3", ... 的一个 fold 函数长这个样子：</p>
{% highlight scala %}
val result: DataStream[String] =
    windowedStream.fold("start", (str, i) => { str + "-" + i })
{% endhighlight %}
          </td>
  </tr>
        <tr>
          <td><strong>Aggregations on windows</strong><br>WindowedStream &rarr; DataStream</td>
          <td>
            <p>聚合一个窗口中的内容。min 与 minBy 的区别是 min 返回了最小值，而 minBy 返回了在这个字段上是最小值的所有元素（max 和 maxBy 也是同样的）。</p>
{% highlight scala %}
windowedStream.sum(0)
windowedStream.sum("key")
windowedStream.min(0)
windowedStream.min("key")
windowedStream.max(0)
windowedStream.max("key")
windowedStream.minBy(0)
windowedStream.minBy("key")
windowedStream.maxBy(0)
windowedStream.maxBy("key")
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Union</strong><br>DataStream* &rarr; DataStream</td>
          <td>
            <p>Union 两个或多个数据流，生成一个新的包含了来自所有流的所有数据的数据流。注意：如果你将一个数据流与其自身进行了合并，在结果流中对于每个元素你都会拿到两份。</p>
{% highlight scala %}
dataStream.union(otherStream1, otherStream2, ...)
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window Join</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>在一个给定的 key 和窗口上 join 两个数据流。</p>
{% highlight scala %}
dataStream.join(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply { ... }
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Window CoGroup</strong><br>DataStream,DataStream &rarr; DataStream</td>
          <td>
            <p>在一个给定的 key 和窗口上 co-group 两个数据流。</p>
{% highlight scala %}
dataStream.coGroup(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply {}
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Connect</strong><br>DataStream,DataStream &rarr; ConnectedStreams</td>
          <td>
            <p>“连接”两个数据流并保持原先的类型。Connect 可以让两条流之间共享状态。</p>
{% highlight scala %}
someStream : DataStream[Int] = ...
otherStream : DataStream[String] = ...

val connectedStreams = someStream.connect(otherStream)
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>CoMap, CoFlatMap</strong><br>ConnectedStreams &rarr; DataStream</td>
          <td>
            <p>在一个 ConnectedStreams 上做类似 map 和 flatMap 的操作。</p>
{% highlight scala %}
connectedStreams.map(
    (_ : Int) => true,
    (_ : String) => false
)
connectedStreams.flatMap(
    (_ : Int) => true,
    (_ : String) => false
)
{% endhighlight %}
          </td>
        </tr>
        <tr>
          <td><strong>Split</strong><br>DataStream &rarr; SplitStream</td>
          <td>
            <p>
                根据具体的标准切分数据流成两个或多个流。

{% highlight scala %}
val split = someDataStream.split(
  (num: Int) =>
    (num % 2) match {
      case 0 => List("even")
      case 1 => List("odd")
    }
)
{% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Select</strong><br>SplitStream &rarr; DataStream</td>
          <td>
            <p>
                从一个 SplitStream 中选出一个或多个流。

{% highlight scala %}

val even = split select "even"
val odd = split select "odd"
val all = split.select("even","odd")
{% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>Iterate</strong><br>DataStream &rarr; IterativeStream  &rarr; DataStream</td>
          <td>
            <p>
              在流(flow)中创建一个带反馈的循环，通过重定向一个 operator 的输出到之前的 operator。这对于定义一些需要不断更新模型的算法是非常有帮助的。下面这段代码对一个流不断地应用迭代体。大于 0 的元素会被发送到反馈通道，剩余的元素会继续发往下游。查阅 <a href="#iterations">迭代</a> 了解完整的描述。

{% highlight java %}
initialStream.iterate {
  iteration => {
    val iterationBody = iteration.map {/*do something*/}
    (iterationBody.filter(_ > 0), iterationBody.filter(_ <= 0))
  }
}
{% endhighlight %}
            </p>
          </td>
        </tr>
        <tr>
          <td><strong>提取时间戳</strong><br>DataStream &rarr; DataStream</td>
          <td>
            <p>
                为了能够工作于使用 event time 语义的窗口，需要从记录中提取时间戳。查阅 <a href="{{ site.baseurl }}/apis/streaming/time.html">working with time</a> 了解更多。
{% highlight scala %}
stream.assignTimestamps { timestampExtractor }
{% endhighlight %}
            </p>
          </td>
        </tr>
  </tbody>
</table>

</div>
</div>

下面的转换只适用于基于 Tuple 类型的数据流：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">


<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>投影</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>从元组中选择了一部分字段子集。
{% highlight java %}
DataStream<Tuple3<Integer, Double, String>> in = // [...]
DataStream<Tuple2<String, Integer>> out = in.project(2,0);
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>投影</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>从元组中选择了一部分字段子集。
{% highlight scala %}
val in : DataStream[(Int,Double,String)] = // [...]
val out = in.project(2,0)
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>


### 物理分区

在流转换后，Flink 在精确控制流分区上也提供了底层的控制（如果需要），通过下面的函数可以实现。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>自定义分区</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            使用一个用户自定义的 Partitioner 对每一个元素选择目标 task。
          
{% highlight java %}
dataStream.partitionCustom(partitioner, "someKey");
dataStream.partitionCustom(partitioner, 0);
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
     <td><strong>随机分区</strong><br>DataStream &rarr; DataStream</td>
     <td>
       <p>
            以均匀分布的形式将元素随机地进行分区。
          
{% highlight java %}
dataStream.shuffle();
{% endhighlight %}
       </p>
     </td>
   </tr>
   <tr>
      <td><strong>Rebalancing (Round-robin partitioning)</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            基于 round-robin 对元素进行分区，使得每个分区负责均衡。对于存在数据倾斜的性能优化是很有用的。
            
{% highlight java %}
dataStream.rebalance();
{% endhighlight %}
        </p>
      </td>
    </tr>
    <tr>
      <td><strong>Rescaling</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            以 round-robin 的形式将元素分区到下游操作的子集中。如果你想要将数据从一个源的每个并行实例中散发到一些 mappers 的子集中，用来分散负载，但是又不想要完全的 rebalance 介入（引入 `rebalance()`），这会非常有用。根据一些如TaskManager 的 slots 个数的配置，这将会只需要本地数据传输，而不是通过网络。
        </p>
        <p>
            上游操作所发送的元素被分区到下游操作的哪些子集，依赖于上游和下游操作的并发度。例如，如果上游操作的并发为 2 ，而下游操作的并发为 4 ，那么一个上游操作会分发元素给两个下游操作，同时另一个上游操作会分发给另两个下游操作。相反的，如果下游操作的并发为 2 ，而下游操作的并发为4，那么两个上游操作会分发数据给一个下游操作，同时另两个上游操作会分发数据给另一个下游操作。
        </p>
        <p>
            在上下游的并发度不是呈倍数关系的情况下，下游操作会有数量不同的来自上游操作的输入。
        </p>
        </p>
            下图是对上述例子的一个可视化：
        </p>

        <div style="text-align: center">
            <img src="{{ site.baseurl }}/apis/streaming/fig/rescale.svg" alt="Checkpoint barriers in data streams" />
            </div>


        <p>
{% highlight java %}
dataStream.rescale();
{% endhighlight %}

        </p>
      </td>
    </tr>
   <tr>
      <td><strong>广播</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            广播每个元素到所有分区。
{% highlight java %}
dataStream.broadcast();
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>自定义分区</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            使用一个用户自定义的 Partitioner 对每一个元素选择目标 task。

{% highlight scala %}
dataStream.partitionCustom(partitioner, "someKey")
dataStream.partitionCustom(partitioner, 0)
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
     <td><strong>随机分区</strong><br>DataStream &rarr; DataStream</td>
     <td>
       <p>
            以均匀分布的形式将元素随机地进行分区。

{% highlight scala %}
dataStream.shuffle()
{% endhighlight %}
       </p>
     </td>
   </tr>
   <tr>
      <td><strong>Rebalancing (Round-robin partitioning)</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            基于 round-robin 对元素进行分区，使得每个分区负责均衡。对于存在数据倾斜的性能优化是很有用的。
{% highlight scala %}
dataStream.rebalance()
{% endhighlight %}
        </p>
      </td>
    </tr>
    <tr>
      <td><strong>Rescaling</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            以 round-robin 的形式将元素分区到下游操作的子集中。如果你想要将数据从一个源的每个并行实例中散发到一些 mappers 的子集中，用来分散负载，但是又不想要完全的 rebalance 介入（引入 `rebalance()`），这会非常有用。根据一些如TaskManager 的 slots 个数的配置，这将会只需要本地数据传输，而不是通过网络。
        </p>
        <p>
            上游操作所发送的元素被分区到下游操作的哪些子集，依赖于上游和下游操作的并发度。例如，如果上游操作的并发为 2 ，而下游操作的并发为 4 ，那么一个上游操作会分发元素给两个下游操作，同时另一个上游操作会分发给另两个下游操作。相反的，如果下游操作的并发为 2 ，而下游操作的并发为4，那么两个上游操作会分发数据给一个下游操作，同时另两个上游操作会分发数据给另一个下游操作。
        </p>
        <p>
            在上下游的并发度不是呈倍数关系的情况下，下游操作会有数量不同的来自上游操作的输入。
        </p>
        </p>
            下图是对上述例子的一个可视化：
        </p>

        <div style="text-align: center">
            <img src="{{ site.baseurl }}/apis/streaming/fig/rescale.svg" alt="Checkpoint barriers in data streams" />
            </div>


        <p>
{% highlight java %}
dataStream.rescale()
{% endhighlight %}

        </p>
      </td>
    </tr>
   <tr>
      <td><strong>广播</strong><br>DataStream &rarr; DataStream</td>
      <td>
        <p>
            广播每个元素到所有分区。

{% highlight scala %}
dataStream.broadcast()
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>

### 任务链和资源组

为了获得更好的性能，你可以链接（chaining）两个连续的转换，这意味着将它们置于同一个线程中。Flink 会尽可能地链接 operators （例如，两个 map 转换）。如果需要的话，该 API 提供了对链接（chaining）细粒度的控制。

如果你想要在整个任务中禁用 chaining ，使用 `StreamExecutionEnvironment.disableOperatorChaining()`。想了解更细粒度的控制，下面的函数是很有用的。注意这些函数只能被用在一个 DataStream 的转换之后，因为它们要指向之前的转换。例如，你可以 `someStream.map(...).startNewChain()`，但不能 `someStream.startNewChain()`。

在 Flink 中一个资源组就是一个 slot 。查阅 [slots]({{site.baseurl}}/setup/config.html#configuring-taskmanager-processing-slots) 了解更多。如果需要的话，你可以手动地隔离 slot 。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td>Start new chain</td>
      <td>
        <p>从这个操作符开始一个新的链条（chain）。这两个 mapper 会被链接，而 filter 不会被与第一个 mapper 链上。
        
{% highlight java %}
someStream.filter(...).map(...).startNewChain().map(...);
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
      <td>Disable chaining</td>
      <td>
        <p>不要与这个 map operator 进行链接。
{% highlight java %}
someStream.map(...).disableChaining();
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
      <td>Set slot sharing group</td>
      <td>
        <p>设置一个操作的 slot 共享组。Flink 会将相同 slot 共享组的操作都放到同一个 slot 中，而把没有 slot 共享组的操作放到其他 slots 上。这可以用来做 slots 隔离。如果所有的输入操作都在相同的 slot 共享组中，那么 slot 共享组会从输入操作中继承来的。默认的 slot 共享组名称是 "default"，操作（operations）可以通过调用 slotSharingGroup("default") 显示地将其置于该组中。

{% highlight java %}
someStream.filter(...).slotSharingGroup("name");
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>

<div data-lang="scala" markdown="1">


<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td>Start new chain</td>
      <td>
        <p>从这个操作符开始一个新的链条（chain）。这两个 mapper 会被链接，而 filter 不会被与第一个 mapper 链上。

{% highlight scala %}
someStream.filter(...).map(...).startNewChain().map(...)
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
      <td>Disable chaining</td>
      <td>
        <p>不要与这个 map operator 进行链接。

{% highlight scala %}
someStream.map(...).disableChaining()
{% endhighlight %}
        </p>
      </td>
    </tr>
   <tr>
      <td>Set slot sharing group</td>
      <td>
        <p>设置一个操作的 slot 共享组。Flink 会将相同 slot 共享组的操作都放到同一个 slot 中，而把没有 slot 共享组的操作放到其他 slots 上。这可以用来做 slots 隔离。如果所有的输入操作都在相同的 slot 共享组中，那么 slot 共享组会从输入操作中继承来的。默认的 slot 共享组名称是 "default"，操作（operations）可以通过调用 slotSharingGroup("default") 显示地将其置于该组中。

{% highlight scala %}
someStream.filter(...).slotSharingGroup("name");
{% endhighlight %}
        </p>
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>


{% top %}

数据源
------------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

源可以通过 `StreamExecutionEnvironment.addSource(sourceFunction)` 来创建。你可以使用 Flink 自带的数据源函数，也可以通过实现 `SourceFunction` 接口写一个自定义的非并行数据源，或者通过实现 `ParallelSourceFunction` 接口或者继承 `RichParallelSourceFunction` 来写一个并行的数据源。

有一些预定义的流数据源，可以通过 `StreamExecutionEnvironment` 访问到。

基于文件：

- `readTextFile(path)` / `TextInputFormat` - 读取文件行并将它们以 String 类型返回。

- `readFile(path)` / 任何输入格式 - 以指定的输入格式读取文件。

- `readFileStream` - 创建一个流，当文件有修改的时候，会将元素附加到流中。

基于 Socket：

- `socketTextStream` - 从一个 socket 中读取数据，可以指定分隔符来切分元素。

基于集合：

- `fromCollection(Collection)` - 从 Java java.util.Collection 集合中创建一个数据流。集合中的所有元素的类型必须一致。

- `fromCollection(Iterator, Class)` - 从一个迭代器中创建一个数据流。Class 指定了迭代器返回的数据类型。

- `fromElements(T ...)` - 从一个对象序列中创建一个数据流。所有的对象的类型必须一致。

- `fromParallelCollection(SplittableIterator, Class)` - 从一个迭代器中创建一个并行数据流。Class 指定了迭代器返回的数据类型。

- `generateSequence(from, to)` - 创建一个并行数据流，生成区间范围内的数字序列。

自定义：

- `addSource` - 添加一个新的源函数。例如，从 Apache Kafka 读取的话你可以：`addSource(new FlinkKafkaConsumer08<>(...))`. 查阅 [连接器(connectors)]({{ site.baseurl }}/apis/streaming/connectors/) 了解更多。

</div>

<div data-lang="scala" markdown="1">

源可以通过 `StreamExecutionEnvironment.addSource(sourceFunction)` 来创建。你可以使用 Flink 自带的数据源函数，也可以通过实现 `SourceFunction` 接口写一个自定义的非并行数据源，或者通过实现 `ParallelSourceFunction` 接口或者继承 `RichParallelSourceFunction` 来写一个并行的数据源。

有一些预定义的流数据源，可以通过 `StreamExecutionEnvironment` 访问到。

基于文件：

- `readTextFile(path)` / `TextInputFormat` - 读取文件行并将它们以 String 类型返回。

- `readFile(path)` / 任何输入格式 - 以指定的输入格式读取文件。

- `readFileStream` - 创建一个流，当文件有修改的时候，会将元素附加到流中。

基于 Socket：

- `socketTextStream` - 从一个 socket 中读取数据，可以指定分隔符来切分元素。

基于集合：

- `fromCollection(Seq)` - 从 Java java.util.Collection 集合中创建一个数据流。集合中的所有元素的类型必须一致。

- `fromCollection(Iterator)` - 从一个迭代器中创建一个数据流。Class 指定了迭代器返回的数据类型。

- `fromElements(elements: _*)` - 从一个对象序列中创建一个数据流。所有的对象的类型必须一致。

- `fromParallelCollection(SplittableIterator)` - 从一个迭代器中创建一个并行数据流。Class 指定了迭代器返回的数据类型。

- `generateSequence(from, to)` - 创建一个并行数据流，生成区间范围内的数字序列。

自定义：

- `addSource` - 添加一个新的源函数。例如，从 Apache Kafka 读取的话你可以：`addSource(new FlinkKafkaConsumer08<>(...))`. 查阅 [连接器(connectors)]({{ site.baseurl }}/apis/streaming/connectors/) 了解更多。

</div>
</div>

{% top %}

数据下沉（Sinks）
----------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

数据下沉（sinks）消费了 DataStream 并将它们发往文件、socket、外部系统、或打印出来。Flink 拥有很多内建的输出格式，这些都被封装在了 DataStream 的操作背后：

- `writeAsText()` / `TextOuputFormat` - 以字符串的形式成行地输出元素。元素的字符串可以通过调用 *toString()* 获得。

- `writeAsCsv(...)` / `CsvOutputFormat` - 将元组写入到 CSV 文件。 Writes tuples as comma-separated value files. 行和字段的分隔符是可以配置的。每个字段的值可以通过对象的 *toString()* 方法获得。

- `print()` / `printToErr()`  - 打印每个元素的 *toString()* 值到标准输出流 / 标准错误流。可选的，可以提供一个前缀（msg）作为前置输出。这可以帮助区分不同次的调用 *print* 。如果并发度大于 1 ，task id 也会被前置到输出中。

- `writeUsingOutputFormat()` / `FileOutputFormat` - 自定义文件输出的方法和基类。支持自定义的对象到字节的转换。 

- `writeToSocket` - 根据 `SerializationSchema` 将元素写入到 socket 中。

- `addSink` - 调用自定义的 sink 方法。Flink 自带了很多连接器（connectors），用来连接其他系统（如 Apache Kafka），这些连接器都实现了 sink 方法。

</div>
<div data-lang="scala" markdown="1">


数据下沉（sinks）消费了 DataStream 并将它们发往文件、socket、外部系统、或打印出来。Flink 拥有很多内建的输出格式，这些都被封装在了 DataStream 的操作背后：

- `writeAsText()` / `TextOuputFormat` - 以字符串的形式成行地输出元素。元素的字符串可以通过调用 *toString()* 获得。

- `writeAsCsv(...)` / `CsvOutputFormat` - 将元组写入到 CSV 文件。 Writes tuples as comma-separated value files. 行和字段的分隔符是可以配置的。每个字段的值可以通过对象的 *toString()* 方法获得。

- `print()` / `printToErr()`  - 打印每个元素的 *toString()* 值到标准输出流 / 标准错误流。可选的，可以提供一个前缀（msg）作为前置输出。这可以帮助区分不同次的调用 *print* 。如果并发度大于 1 ，task id 也会被前置到输出中。

- `writeUsingOutputFormat()` / `FileOutputFormat` - 自定义文件输出的方法和基类。支持自定义的对象到字节的转换。 

- `writeToSocket` - 根据 `SerializationSchema` 将元素写入到 socket 中。

- `addSink` - 调用自定义的 sink 方法。Flink 自带了很多连接器（connectors），用来连接其他系统（如 Apache Kafka），这些连接器都实现了 sink 方法。

</div>
</div>

注意 `DataStream` 的 `write*()` 方法主要是用来 debug 的。它们不会参与 Flink 的 checkpoint 机制，这意味着这些函数一般只有最少一次（at-lease-once）语义。数据刷到目标系统的动作依赖于 OutputFormat 的实现。这也就是说不是所有发送给 OutputFormat 的元素会立即在目标系统上可见。另外，在失败的情况下，这些记录可能会丢失。

为了可靠性，可以用 `flink-connector-filesystem` 实现流到文件系统的恰好一次（exactly-once）。同样的，可以通过 `.addSink(...)` 方法自己实现 SinkFunction，这也能参与 Flink 的 checkpoint 机制，达到 exactly-once 语义。

{% top %}

迭代（Iterations）
----------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

迭代的流程序实现了一个分步的函数并嵌入到了 `IterativeStream` 中。因为一个 DataStream 程序可能永远不会结束的，所以迭代次数没有上限。你需要指出流的哪部分是要反馈到迭代中的，哪部分是要继续往下游发送的，这可以用 `split` 或 `filter` 转换来实现。这里，我们给出一个使用 filters 的样例。首先，我们定义了一个 `IterativeStream`。

{% highlight java %}
IterativeStream<Integer> iteration = input.iterate();
{% endhighlight %}

然后，我们使用一系列的转换来说明了迭代中被执行的逻辑（这里就是一个简单 `map` 转换）。

{% highlight java %}
DataStream<Integer> iterationBody = iteration.map(/* this is executed many times */);
{% endhighlight %}

要关闭一个迭代并定义迭代的尾部，请调用 `IterativeStream` 的 `closeWith(feedbackStream)` 方法。传给 `closeWith` 方法的 DataStream 会被反馈给迭代的头部。一种常见的形式就是使用一个 filter 来分离流中需要被反馈的部分和需要被继续发往下游的部分。在 filter 中可以定义“结束”逻辑，来决定了一个元素是被发往下游还是被反馈的。

{% highlight java %}
iteration.closeWith(iterationBody.filter(/* one part of the stream */));
DataStream<Integer> output = iterationBody.filter(/* some other part of the stream */);
{% endhighlight %}

默认情况下，反馈流的分区将被自动设定为与迭代头部的输入相同的分区。用户可以在 `closeWith` 方法中设置一个可选的布尔 flag 来覆盖这种默认行为。

例如，下面这段程序就是对一串数字不断地做减 1 操作，知道它们都为 0 了为止。

{% highlight java %}
DataStream<Long> someIntegers = env.generateSequence(0, 1000);

IterativeStream<Long> iteration = someIntegers.iterate();

DataStream<Long> minusOne = iteration.map(new MapFunction<Long, Long>() {
  @Override
  public Long map(Long value) throws Exception {
    return value - 1 ;
  }
});

DataStream<Long> stillGreaterThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value > 0);
  }
});

iteration.closeWith(stillGreaterThanZero);

DataStream<Long> lessThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value <= 0);
  }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

迭代的流程序实现了一个分步的函数并嵌入到了 `IterativeStream` 中。因为一个 DataStream 程序可能永远不会结束的，所以迭代次数没有上限。你需要指出流的哪部分是要反馈到迭代中的，哪部分是要继续往下游发送的，这可以用 `split` 或 `filter` 转换来实现。这里，我们给出一个迭代样例，其中主体部分（不断重复计算的部分）是一个简单的 map 转换，而每个元素到底是被反馈还是被发往下游主要是使用 filter 来区分的。


{% highlight scala %}
val iteratedStream = someDataStream.iterate(
  iteration => {
    val iterationBody = iteration.map(/* this is executed many times */)
    (tail.filter(/* one part of the stream */), tail.filter(/* some other part of the stream */))
})
{% endhighlight %}

默认情况下，反馈流的分区将被自动设定为与迭代头部的输入相同的分区。用户可以在 closeWith 方法中设置一个可选的布尔 flag 来覆盖这种默认行为。

例如，下面这段程序就是对一串数字不断地做减 1 操作，知道它们都为 0 了为止。

{% highlight scala %}
val someIntegers: DataStream[Long] = env.generateSequence(0, 1000)

val iteratedStream = someIntegers.iterate(
  iteration => {
    val minusOne = iteration.map( v => v - 1)
    val stillGreaterThanZero = minusOne.filter (_ > 0)
    val lessThanZero = minusOne.filter(_ <= 0)
    (stillGreaterThanZero, lessThanZero)
  }
)
{% endhighlight %}

</div>
</div>

{% top %}

<a id="execution-parameters"></a>

执行参数
--------------------

`StreamExecutionEnvironment` 包含了 `ExecutionConfig`，`ExecutionConfig` 用来设置任务运行时的具体配置值。

请参考 [execution configuration]({{ site.baseurl }}/apis/common/index.html#execution-configuration) 了解更多参数的说明。下面这些参数是 DataStream API 特有的：

- `enableTimestamps()` / **`disableTimestamps()`**: 启用的话，从源发出的每一条消息（event）都会附加上一个时间戳。`areTimestampsEnabled()` 返回了当前是否启用的值。

- `setAutoWatermarkInterval(long milliseconds)`: 设置自动水位排放的间隔时间。你可以通过 `long getAutoWatermarkInterval()` 获得当前值。

{% top %}

### 容错

[容错章节](fault_tolerance.html) 描述了启用和配置 Flink checkpoint 机制的选项和配置项。

### 延迟控制

默认情况下，数据并不是一个接着一个在网络上传输的（这会导致不必要的网络流量），而是被缓冲的（buffered）。缓冲（实际上是机器之间的传输） 的大小可以在 Flink 配置文件中设置。虽然这种方法有利于优化吞吐量，但当输入的数据流不够快时，它可能会导致延迟问题时。要控制吞吐量和延迟，你可以在 `StreamExecutionEnvironment` 上使用`env.setBufferTimeout(timeoutMillis)`（或者单独的 operator 上）设置等待缓冲区被填满的最长等待时间。超过了这个时间，即时缓冲区还没有满也会被自动发送出去。默认的超时时间是 100 ms。

用法:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.genereateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment
env.setBufferTimeout(timeoutMillis)

env.genereateSequence(1,10).map(myMap).setBufferTimeout(timeoutMillis)
{% endhighlight %}
</div>
</div>

为了最大化吞吐量，可以设置 `setBufferTimeout(-1)`，这会移除超时等待时间而缓冲区只有被填满后才会被发送出去。为了最小化延时，可以设置一个接近 0 的超时时间（如 5 或 10 毫秒）。建议避免缓冲超时时间为 0 ，因为这会降低服务性能。

{% top %}

调试
---------

先确保实现的算法按照预期正常工作了，再将这个 streaming 程序跑到分布式集群上，是一个好想法。因此，实现数据分析程序通常是一个渐进的过程：检查结果、调试和改进。

Flink 提供了许多特性来极大地简化了数据分析程序的开发过程。比如支持了在 IDE 中进行本地调试，测试数据注入，和结果数据的收集。本节主要就如何简化开发 Flink 程序提供一些提示。

### 本地执行环境

`LocalStreamEnvironment` 会在同一个 JVM 进程中启动一个 Flink 引擎。如果你从 IDE 中启动了`LocalStreamEnvironment`，你可以在你的代码中设置断点，然后轻松地调试你的程序。

一个 LocalEnvironment 可以像下面这样被创建和使用；

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

DataStream<String> lines = env.addSource(/* some source */);
// build your program

env.execute();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = StreamExecutionEnvironment.createLocalEnvironment()

val lines = env.addSource(/* some source */)
// build your program

env.execute()
{% endhighlight %}
</div>
</div>

### 集合数据源

Flink 提供了基于 Java 集合实现的特殊数据源，用来简化测试。一旦一个程序测试通过了，数据源和 sinks 可以被方便地替换成从外部系统读写的数据源和 sinks。

集合数据源可以以如下的方式使用：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// Create a DataStream from a list of elements
DataStream<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataStream from any Java collection
List<Tuple2<String, Integer>> data = ...
DataStream<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataStream from an Iterator
Iterator<Long> longIt = ...
DataStream<Long> myLongs = env.fromCollection(longIt, Long.class);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.createLocalEnvironment()

// Create a DataStream from a list of elements
val myInts = env.fromElements(1, 2, 3, 4, 5)

// Create a DataStream from any Collection
val data: Seq[(String, Int)] = ...
val myTuples = env.fromCollection(data)

// Create a DataStream from an Iterator
val longIt: Iterator[Long] = ...
val myLongs = env.fromCollection(longIt)
{% endhighlight %}
</div>
</div>

**注意：** 现在，集合数据源需要数据类型和迭代器都实现 `Serializable`。此外，集合数据源不能被并行执行（parallelism = 1）。


### 数据接收迭代器（Iterator Data Sink）

Flink 为了测试和调试的目的还提供了一个 sink 来收集 DataStream 的结果。可以像下面这样使用：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
import org.apache.flink.contrib.streaming.DataStreamUtils

DataStream<Tuple2<String, Integer>> myResult = ...
Iterator<Tuple2<String, Integer>> myOutput = DataStreamUtils.collect(myResult)
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
import org.apache.flink.contrib.streaming.DataStreamUtils
import scala.collection.JavaConverters.asScalaIteratorConverter

val myResult: DataStream[(String, Int)] = ...
val myOutput: Iterator[(String, Int)] = DataStreamUtils.collect(myResult.getJavaStream).asScala
{% endhighlight %}
</div>
</div>

{% top %}
