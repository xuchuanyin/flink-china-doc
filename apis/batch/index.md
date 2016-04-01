---
title: "Flink DataSet API 编程指南"

# Top-level navigation
top-nav-group: apis
top-nav-pos: 3
top-nav-title: <strong>Batch 指南</strong> (DataSet API)

# Sub-level navigation
sub-nav-group: batch
sub-nav-group-title: Batch 指南
sub-nav-id: dataset_api
sub-nav-pos: 1
sub-nav-title: DataSet API
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


DataSet 程序是 Flink 实现了数据集上的转换操作（如 filtering, mapping, joining, grouping 等）的普通程序。初始数据集是从特定的数据源（例如文件或集合）中创建出来的。
通过sink来返回结果，它有可能把结果写到（分布式）文件或标准输出（比如命令行终端）。
Flink 可以运行在各种环境中， 比如 standalone，或嵌入到其他程序中。可以在本地 JVM 或集群上运行 Flink 程序。

如果想要学习DataSet,建议从[基本概念]({{ site.baseurl }}/apis/common/index.html)和[剖析 Flink 程序]({{ site.baseurl }}/apis/common/index.html#anatomy-of-a-flink-program)开始入手，并逐步增加自己的[转换操作](#dataset-transformations)。其他章节将介绍额外的操作或高级特性。

* This will be replaced by the TOC
{:toc}

<a id="example-program"></a>

示例程序
---------------

下面的程序是一个完整的、可运行的 WordCount 示例。你可以复制 &amp; 粘贴下方代码并在本地运行。你只需要引入正确的 Flink 依赖到项目中（参见 [关联 Flink]({{ site.baseurl }}/apis/common/#linking-with-flink)）并指定具体的 imports。之后你就可以出发了！


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

{% highlight java %}
public class WordCountExample {
    public static void main(String[] args) throws Exception {
        final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

        DataSet<String> text = env.fromElements(
            "Who's there?",
            "I think I hear them. Stand, ho! Who's there?");

        DataSet<Tuple2<String, Integer>> wordCounts = text
            .flatMap(new LineSplitter())
            .groupBy(0)
            .sum(1);

        wordCounts.print();
    }

    public static class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String line, Collector<Tuple2<String, Integer>> out) {
            for (String word : line.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }
}
{% endhighlight %}

</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._

object WordCount {
  def main(args: Array[String]) {

    val env = ExecutionEnvironment.getExecutionEnvironment
    val text = env.fromElements(
      "Who's there?",
      "I think I hear them. Stand, ho! Who's there?")

    val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
      .map { (_, 1) }
      .groupBy(0)
      .sum(1)

    counts.print()
  }
}
{% endhighlight %}
</div>

</div>

{% top %}

<a id="dataset-transformations"></a>

DataSet 转换（Transformations）
-----------------------

转换（Transformations) 是将一个或多个 DataSet 转化为一个新的 DataSet。程序可以结合多个转换构建出一个复杂的拓扑。

本节将简要介绍可用的转换。[转换文档](dataset_transformations.html)有对所有转换的完整介绍，并附带了例子。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>Map</strong></td>
      <td>
        <p>输入一个元素，生成一个元素。</p>
{% highlight java %}
data.map(new MapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>FlatMap</strong></td>
      <td>
        <p>输入一个元素，生成零个、一个、或多个元素。</p>
{% highlight java %}
data.flatMap(new FlatMapFunction<String, String>() {
  public void flatMap(String value, Collector<String> out) {
    for (String s : value.split(" ")) {
      out.collect(s);
    }
  }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>MapPartition</strong></td>
      <td>
      <p>在一个函数中对一个并行的分区进行转换。该函数将分区以 <code>Iterable</code> 流的形式传入，然后生成任意数量的结果值。每个分区中的元素数量取决于上一个操作的并行度。
      </p>
{% highlight java %}
data.mapPartition(new MapPartitionFunction<String, Long>() {
  public void mapPartition(Iterable<String> values, Collector<Long> out) {
    long c = 0;
    for (String s : values) {
      c++;
    }
    out.collect(c);
  }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Filter</strong></td>
      <td>
        <p>执行过滤操作，对每个元素进行判断，只保留函数返回为 true 的元素。<br/>

        <strong>重要:</strong> 系统假定该方法不会修改元素。违反该假定的话可能会导致错误的结果。
        </p>
{% highlight java %}
data.filter(new FilterFunction<Integer>() {
  public boolean filter(Integer value) { return value > 1000; }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Reduce</strong></td>
      <td>
        <p>通过不断重复合并两个元素到一个元素的操作，将一组元素合并为一个元素。Reduce 可以应用在一个完整的数据集上，或者已分组的数据集上。
        </p>
{% highlight java %}
data.reduce(new ReduceFunction<Integer> {
  public Integer reduce(Integer a, Integer b) { return a + b; }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>ReduceGroup</strong></td>
      <td>
        <p>将一组元素合并为一个或多个元素。ReduceGroup 可以应用在一个完整的数据集上，或者已分组的数据集上。
        </p>
{% highlight java %}
data.reduceGroup(new GroupReduceFunction<Integer, Integer> {
  public void reduce(Iterable<Integer> values, Collector<Integer> out) {
    int prefixSum = 0;
    for (Integer i : values) {
      prefixSum += i;
      out.collect(prefixSum);
    }
  }
});
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Aggregate</strong></td>
      <td>
        <p>将一组值聚合到一个值中。Aggregation 函数可以认为是内建的 reduce 函数。Aggregate 可以应用在一个完整的数据集上，或者已分组的数据集上。
        </p>
{% highlight java %}
Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.aggregate(SUM, 0).and(MIN, 2);
{% endhighlight %}
	<p>同时，对于 minimum, maximum, 和 sum 聚合，你也可以使用简写语法。</p>
	{% highlight java %}
	Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.sum(0).andMin(2);
	{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Distinct</strong></td>
      <td>
        <p>返回数据集中不重复的元素。去除了输入数据集中在全字段（或字段子集）上重复的元素。
        </p>
    {% highlight java %}
        data.distinct();
    {% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Join</strong></td>
      <td>
        Join 两个数据集，生成所有 key 相同的元素对。
        可选地，可以使用 JoinFunction 转换元素对为单个元素，或者使用 FlatJoinFunction 转换元素对为任意多（包括无）的元素。参见 <a href="{{ site.baseurl}}/apis/common/#specifying-keys">key 章节</a> 了解如何定义 join 的 key。
        
{% highlight java %}
result = input1.join(input2)
               .where(0)       // key of the first input (tuple field 0)
               .equalTo(1);    // key of the second input (tuple field 1)
{% endhighlight %}
        你可以通过 <i>Join Hints</i> 指定 join 执行的方式。Hint 描述了 join 是否会有分区或广播的发生，以及是否使用了基于排序或基于哈希的算法。请参考 <a href="dataset_transformations.html#join-algorithm-hints">转换指南</a> 了解可用的 hint 列表以及示例。</br>
        如果没有指定 hint，系统会尝试对输入数据进行估算并根据估算结果选择一个最佳策略。
{% highlight java %}
// This executes a join by broadcasting the first data set
// using a hash table for the broadcasted data
result = input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
               .where(0).equalTo(1);
{% endhighlight %}
        注意 join 转换仅能处理 equi-joins。其他 join 类型需要使用 OuterJoin 或 CoGroup。
      </td>
    </tr>

    <tr>
      <td><strong>OuterJoin</strong></td>
      <td>
      在两个数据集上执行 left/right/full-outer join。Outer join 类似普通的（inner）join，会生成相同 key 的所有元素对。额外的，对于 "outer" （left 或 right 或 full）那一面的记录而言，如果在另一面没有找到匹配的 key，则这些记录都会被保留。匹配的元素对（或一个元素和另一边的一个 <code>null</code> 值）传入到 JoinFunction 中被转成了单个的元素，或传入到 FlatJoinFunction 中被转成了任意多（包括无）的元素。参见 <a href="{{ site.baseurl}}/apis/common/#specifying-keys">key 章节</a> 了解如何定义 join 的 key。
{% highlight java %}
input1.leftOuterJoin(input2) // rightOuterJoin or fullOuterJoin for right or full outer joins
      .where(0)              // key of the first input (tuple field 0)
      .equalTo(1)            // key of the second input (tuple field 1)
      .with(new JoinFunction<String, String, String>() {
          public String join(String v1, String v2) {
             // NOTE:
             // - v2 might be null for leftOuterJoin
             // - v1 might be null for rightOuterJoin
             // - v1 OR v2 might be null for fullOuterJoin
          }
      });
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>CoGroup</strong></td>
      <td>
        <p>对两维数据进行reduce操作， 对一个或多个字段进行分组操作，然后进行join这些分组。每一对分组都会调用该转换函数。参见 <a href="{{ site.baseurl}}/apis/common/#specifying-keys">key 章节</a> 了解如何定义 join 的 key。
        </p>
{% highlight java %}
data1.coGroup(data2)
     .where(0)
     .equalTo(1)
     .with(new CoGroupFunction<String, String, String>() {
         public void coGroup(Iterable<String> in1, Iterable<String> in2, Collector<String> out) {
           out.collect(...);
         }
      });
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Cross</strong></td>
      <td>
        <p>对两个输入进行笛卡尔积（cross product），生成所有的元素对。可选地，可以使用 CrossFunction 将元素对转换成单个元素。
        </p>
{% highlight java %}
DataSet<Integer> data1 = // [...]
DataSet<String> data2 = // [...]
DataSet<Tuple2<Integer, String>> result = data1.cross(data2);
{% endhighlight %}
      <p>注意：Cross 是一个潜在的计算量非常大的操作。建议通过使用 <i>crossWithTiny()</i> 和 <i>crossWithHuge()</i> 告诉系统该数据集的大小。
      </p>
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>生成两个数据集的并集。如果多于一个数据集被用作函数的输入，该操作会被隐式地调用。
        </p>
{% highlight java %}
DataSet<String> data1 = // [...]
DataSet<String> data2 = // [...]
DataSet<String> result = data1.union(data2);
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Rebalance</strong></td>
      <td>
        <p>
        主要是解决在多分区情况下，数据倾斜问题。将一个数据集均匀地分布到多个并行分区中。只有类似 map 的转换操作会跟在 rebalance 转换之后。</p>
{% highlight java %}
DataSet<String> in = // [...]
DataSet<String> result = in.rebalance()
                           .map(new Mapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Hash-Partition</strong></td>
      <td>
        <p>在某个key上执行哈希分区。Key 可以是 position key，也可以是 expression key，或是 selector 函数。
        </p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionByHash(0)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Range-Partition</strong></td>
      <td>
        <p>在某个key上 Range-Partition。Key 可以是 position key，也可以是 expression key，或是 selector 函数。
        </p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionByRange(0)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Custom Partitioning</strong></td>
      <td>
        <p>自定义分区操作。在数据上手工指定一个分区函数。
          <br/>
          <i>注意</i>: 该方法仅在单字段 key 上有效。</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionCustom(Partitioner<K> partitioner, key)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Sort Partition</strong></td>
      <td>
        <p>在某个字段上本地排序所有分区的数据。字段可以指定为 tuple 下标，或字段表达式。
        在多字段上排序可以通过 chaining 上 sortPartition()。
         </p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.sortPartition(1, Order.ASCENDING)
                            .mapPartition(new PartitionMapper());
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>First-n</strong></td>
      <td>
        <p>返回一个数据集中的的前 n（任意）个元素。First-n 可以应用在普通的数据集上，或是分组的数据集上，或是分组且排序的数据集上。分组的 key 可以指定为 key-selector 函数，或是 field position。
         </p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
// regular data set
DataSet<Tuple2<String,Integer>> result1 = in.first(3);
// grouped data set
DataSet<Tuple2<String,Integer>> result2 = in.groupBy(0)
                                            .first(3);
// grouped-sorted data set
DataSet<Tuple2<String,Integer>> result3 = in.groupBy(0)
                                            .sortGroup(1, Order.ASCENDING)
                                            .first(3);
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

----------

以下的转换操作仅可以用在 Tuple 的数据集上：

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">转换</th>
      <th class="text-center">描述</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Project</strong></td>
      <td>
        <p>从 tuples 中选择一部分字段子集。
        </p>
{% highlight java %}
DataSet<Tuple3<Integer, Double, String>> in = // [...]
DataSet<Tuple2<String, Integer>> out = in.project(2,0);
{% endhighlight %}
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
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>Map</strong></td>
      <td>
        <p>输入一个元素，生成一个元素。</p>
{% highlight scala %}
data.map { x => x.toInt }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>FlatMap</strong></td>
      <td>
        <p>输入一个元素，生成零个、一个、或多个元素。</p>
{% highlight scala %}
data.flatMap { str => str.split(" ") }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>MapPartition</strong></td>
      <td>
        <p>在一个函数中对一个并行的分区进行转换。该函数将分区以 <code>Iterable</code> 流的形式传入，然后生成任意数量的结果值。每个分区中的元素数量取决于上一个操作的并行度。</p>
{% highlight scala %}
data.mapPartition { in => in map { (_, 1) } }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Filter</strong></td>
      <td>
        <p>执行过滤操作，对每个元素进行判断，只保留函数返回为 true 的元素。
          <strong>重要:</strong> 系统假定该方法不会修改元素。违反该假定的话可能会导致错误的结果。
        </p>
{% highlight scala %}
data.filter { _ > 1000 }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Reduce</strong></td>
      <td>
        <p>通过不断重复合并两个元素到一个元素的操作，将一组元素合并为一个元素。Reduce 可以应用在一个完整的数据集上，或者已分组的数据集上。</p>
{% highlight scala %}
data.reduce { _ + _ }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>ReduceGroup</strong></td>
      <td>
        <p>将一组元素合并为一个或多个元素。ReduceGroup 可以应用在一个完整的数据集上，或者已分组的数据集上。</p>
{% highlight scala %}
data.reduceGroup { elements => elements.sum }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Aggregate</strong></td>
      <td>
        <p>将一组值聚合到一个值中。Aggregation 函数可以认为是内建的 reduce 函数。Aggregate 可以应用在一个完整的数据集上，或者已分组的数据集上。</p>
{% highlight scala %}
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Doublr)] = input.aggregate(SUM, 0).aggregate(MIN, 2);
{% endhighlight %}
  <p>同时，对于 minimum, maximum, 和 sum 聚合，你也可以使用简写语法。</p>
{% highlight scala %}
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Doublr)] = input.sum(0).min(2)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Distinct</strong></td>
      <td>
        <p>返回数据集中不重复的元素。去除了输入数据集中在全字段（或字段子集）上重复的元素。</p>
{% highlight scala %}
   data.distinct()
{% endhighlight %}
      </td>
    </tr>

    </tr>
      <td><strong>Join</strong></td>
      <td>
      Join 两个数据集，生成所有 key 相同的元素对。 可选地，可以使用 JoinFunction 转换元素对为单个元素，或者使用 FlatJoinFunction 转换元素对为任意多（包括无）的元素。参见 <a href="{{ site.baseurl}}/apis/common/#specifying-keys">key 章节</a> 了解如何定义 join 的 key。
{% highlight scala %}
// In this case tuple fields are used as keys. "0" is the join field on the first tuple
// "1" is the join field on the second tuple.
val result = input1.join(input2).where(0).equalTo(1)
{% endhighlight %}
  你可以通过 <i>Join Hints</i> 指定 join 执行的方式。Hint 描述了 join 是否会有分区或广播的发生，以及是否使用了基于排序或基于哈希的算法。请参考 <a href="dataset_transformations.html#join-algorithm-hints">转换指南</a> 了解可用的 hint 列表以及示例。 <br>
  如果没有指定 hint，系统会尝试对输入数据进行估算并根据估算结果选择一个最佳策略。
{% highlight scala %}
// This executes a join by broadcasting the first data set
// using a hash table for the broadcasted data
val result = input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
                   .where(0).equalTo(1)
{% endhighlight %}
          Note that the join transformation works only for equi-joins. Other join types need to be expressed using OuterJoin or CoGroup.
      </td>
    </tr>

    <tr>
      <td><strong>OuterJoin</strong></td>
      <td>
      在两个数据集上执行 left/right/full-outer join。Outer join 类似普通的（inner）join，会生成相同 key 的所有元素对。额外的，对于 "outer" （left 或 right 或 full）那一面的记录而言，如果在另一面没有找到匹配的 key，则这些记录都会被保留。匹配的元素对（或一个元素和另一边的一个 null 值）传入到 JoinFunction 中被转成了单个的元素，或传入到 FlatJoinFunction 中被转成了任意多（包括无）的元素。参见 <a href="{{ site.baseurl}}/apis/common/#specifying-keys">key 章节</a> 了解如何定义 join 的 key。
{% highlight scala %}
val joined = left.leftOuterJoin(right).where(0).equalTo(1) {
   (left, right) =>
     val a = if (left == null) "none" else left._1
     (a, right)
  }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>CoGroup</strong></td>
      <td>
        <p>对两维数据进行reduce操作， 对一个或多个字段进行分组操作，然后进行join这些分组。每一对分组都会调用该转换函数。参见 <a href="{{ site.baseurl}}/apis/common/#specifying-keys">key 章节</a> 了解如何定义 join 的 key。</p>
{% highlight scala %}
data1.coGroup(data2).where(0).equalTo(1)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Cross</strong></td>
      <td>
        <p>对两个输入进行笛卡尔积（cross product），生成所有的元素对。可选地，可以使用 CrossFunction 将元素对转换成单个元素。</p>
{% highlight scala %}
val data1: DataSet[Int] = // [...]
val data2: DataSet[String] = // [...]
val result: DataSet[(Int, String)] = data1.cross(data2)
{% endhighlight %}
  <p>注意：Cross 是一个潜在的计算量非常大的操作。建议通过使用 <i>crossWithTiny()</i> 和 <i>crossWithHuge()</i> 告诉系统该数据集的大小。</p>
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>生成两个数据集的并集。如果多于一个数据集被用作函数的输入，该操作会被隐式地调用。</p>
{% highlight scala %}
data.union(data2)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Rebalance</strong></td>
      <td>
        <p>主要是解决在多分区情况下，数据倾斜问题。将一个数据集均匀地分布到多个并行分区中。只有类似 map 的转换操作会跟在 rebalance 转换之后。</p>
{% highlight scala %}
val data1: DataSet[Int] = // [...]
val result: DataSet[(Int, String)] = data1.rebalance().map(...)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Hash-Partition</strong></td>
      <td>
        <p>在某个key上执行哈希分区。Key 可以是 position key，也可以是 expression key，或是 selector 函数。</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.partitionByHash(0).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Range-Partition</strong></td>
      <td>
        <p>在某个key上 Range-Partition。Key 可以是 position key，也可以是 expression key，或是 selector 函数。</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.partitionByRange(0).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    </tr>
    <tr>
      <td><strong>Custom Partitioning</strong></td>
      <td>
        <p>自定义分区操作。在数据上手工指定一个分区函数。 
          <br/>
          <i>注意</i>: 该方法仅在单字段 key 上有效。</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in
  .partitionCustom(partitioner: Partitioner[K], key)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Sort Partition</strong></td>
      <td>
        <p>在某个字段上本地排序所有分区的数据。字段可以指定为 tuple 下标，或字段表达式。 在多字段上排序可以通过 chaining 上 sortPartition()。</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.sortPartition(1, Order.ASCENDING).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>First-n</strong></td>
      <td>
        <p>返回一个数据集中的的前 n（任意）个元素。First-n 可以应用在普通的数据集上，或是分组的数据集上，或是分组且排序的数据集上。分组的 key 可以指定为 key-selector 函数，或是 field position。</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
// regular data set
val result1 = in.first(3)
// grouped data set
val result2 = in.groupBy(0).first(3)
// grouped-sorted data set
val result3 = in.groupBy(0).sortGroup(1, Order.ASCENDING).first(3)
{% endhighlight %}
      </td>
    </tr>
  </tbody>
</table>

</div>
</div>


当对一个转换自定义一个名字时，可以通过`setParallelism(int)`来设置转换的[并行度](#parallel-execution), 这种方式可以帮助调试。这对于 DataSource 和 DataSinks 都是通用的。

传递给`withParameters(Configuration)`的 Configuration 对象，可以在用户函数内的 `open()` 函数中被访问。

{% top %}

Data Sources
------------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">


Data sources 从文件或 Java 集合中创建了初始的数据集。背后的机制请参考{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/InputFormat.java "InputFormat"%}。Flink 内建了多种常见格式以方便从常用文件中创建数据集。它们中的大多数都可以通过 *ExecutionEnvironment* 的函数直接使用。

File-based:

- `readTextFile(path)` / `TextInputFormat` - Reads files line wise and returns them as Strings.

- `readTextFileWithValue(path)` / `TextValueInputFormat` - Reads files line wise and returns them as
  StringValues. StringValues are mutable strings.

- `readCsvFile(path)` / `CsvInputFormat` - Parses files of comma (or another char) delimited fields.
  Returns a DataSet of tuples or POJOs. Supports the basic java types and their Value counterparts as field
  types.

- `readFileOfPrimitives(path, Class)` / `PrimitiveInputFormat` - Parses files of new-line (or another char sequence)
  delimited primitive data types such as `String` or `Integer`.

- `readFileOfPrimitives(path, delimiter, Class)` / `PrimitiveInputFormat` - Parses files of new-line (or another char sequence)
   delimited primitive data types such as `String` or `Integer` using the given delimiter.

- `readHadoopFile(FileInputFormat, Key, Value, path)` / `FileInputFormat` - Creates a JobConf and reads file from the specified
   path with the specified FileInputFormat, Key class and Value class and returns them as Tuple2<Key, Value>.

- `readSequenceFile(Key, Value, path)` / `SequenceFileInputFormat` - Creates a JobConf and reads file from the specified path with
   type SequenceFileInputFormat, Key class and Value class and returns them as Tuple2<Key, Value>.


Collection-based:

- `fromCollection(Collection)` - Creates a data set from the Java Java.util.Collection. All elements
  in the collection must be of the same type.

- `fromCollection(Iterator, Class)` - Creates a data set from an iterator. The class specifies the
  data type of the elements returned by the iterator.

- `fromElements(T ...)` - Creates a data set from the given sequence of objects. All objects must be
  of the same type.

- `fromParallelCollection(SplittableIterator, Class)` - Creates a data set from an iterator, in
  parallel. The class specifies the data type of the elements returned by the iterator.

- `generateSequence(from, to)` - Generates the sequence of numbers in the given interval, in
  parallel.

Generic:

- `readFile(inputFormat, path)` / `FileInputFormat` - Accepts a file input format.

- `createInput(inputFormat)` / `InputFormat` - Accepts a generic input format.

**Examples**

{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// read text file from local files system
DataSet<String> localLines = env.readTextFile("file:///path/to/my/textfile");

// read text file from a HDFS running at nnHost:nnPort
DataSet<String> hdfsLines = env.readTextFile("hdfs://nnHost:nnPort/path/to/my/textfile");

// read a CSV file with three fields
DataSet<Tuple3<Integer, String, Double>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
	                       .types(Integer.class, String.class, Double.class);

// read a CSV file with five fields, taking only two of them
DataSet<Tuple2<String, Double>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
                               .includeFields("10010")  // take the first and the fourth field
	                       .types(String.class, Double.class);

// read a CSV file with three fields into a POJO (Person.class) with corresponding fields
DataSet<Person>> csvInput = env.readCsvFile("hdfs:///the/CSV/file")
                         .pojoType(Person.class, "name", "age", "zipcode");


// read a file from the specified path of type TextInputFormat
DataSet<Tuple2<LongWritable, Text>> tuples =
 env.readHadoopFile(new TextInputFormat(), LongWritable.class, Text.class, "hdfs://nnHost:nnPort/path/to/file");

// read a file from the specified path of type SequenceFileInputFormat
DataSet<Tuple2<IntWritable, Text>> tuples =
 env.readSequenceFile(IntWritable.class, Text.class, "hdfs://nnHost:nnPort/path/to/file");

// creates a set from some given elements
DataSet<String> value = env.fromElements("Foo", "bar", "foobar", "fubar");

// generate a number sequence
DataSet<Long> numbers = env.generateSequence(1, 10000000);

// Read data from a relational database using the JDBC input format
DataSet<Tuple2<String, Integer> dbData =
    env.createInput(
      // create and configure input format
      JDBCInputFormat.buildJDBCInputFormat()
                     .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                     .setDBUrl("jdbc:derby:memory:persons")
                     .setQuery("select name, age from persons")
                     .finish(),
      // specify type information for DataSet
      new TupleTypeInfo(Tuple2.class, STRING_TYPE_INFO, INT_TYPE_INFO)
    );

// Note: Flink's program compiler needs to infer the data types of the data items which are returned
// by an InputFormat. If this information cannot be automatically inferred, it is necessary to
// manually provide the type information as shown in the examples above.
{% endhighlight %}

#### Configuring CSV Parsing

Flink 提供一些 CSV 解析的配置:

- `types(Class ... types)` specifies the types of the fields to parse. **It is mandatory to configure the types of the parsed fields.**
  In case of the type class Boolean.class, "True" (case-insensitive), "False" (case-insensitive), "1" and "0" are treated as booleans.

- `lineDelimiter(String del)` specifies the delimiter of individual records. The default line delimiter is the new-line character `'\n'`.

- `fieldDelimiter(String del)` specifies the delimiter that separates fields of a record. The default field delimiter is the comma character `','`.

- `includeFields(boolean ... flag)`, `includeFields(String mask)`, or `includeFields(long bitMask)` defines which fields to read from the input file (and which to ignore). By default the first *n* fields (as defined by the number of types in the `types()` call) are parsed.

- `parseQuotedStrings(char quoteChar)` enables quoted string parsing. Strings are parsed as quoted strings if the first character of the string field is the quote character (leading or tailing whitespaces are *not* trimmed). Field delimiters within quoted strings are ignored. Quoted string parsing fails if the last character of a quoted string field is not the quote character or if the quote character appears at some point which is not the start or the end of the quoted string field (unless the quote character is escaped using '\'). If quoted string parsing is enabled and the first character of the field is *not* the quoting string, the string is parsed as unquoted string. By default, quoted string parsing is disabled.

- `ignoreComments(String commentPrefix)` specifies a comment prefix. All lines that start with the specified comment prefix are not parsed and ignored. By default, no lines are ignored.

- `ignoreInvalidLines()` enables lenient parsing, i.e., lines that cannot be correctly parsed are ignored. By default, lenient parsing is disabled and invalid lines raise an exception.

- `ignoreFirstLine()` configures the InputFormat to ignore the first line of the input file. By default no line is ignored.


#### Recursive Traversal of the Input Path Directory

遍历一个目录。对于基于文件的输入，如果输入的是一个目录，默认不会遍历子目录的文件。而是只会读取第一层目录的文件，忽略子目录的文件。可以通过设置`recursive.file.enumeration`来开启遍历子目录的设置。如下示例所示：


{% highlight java %}
// enable recursive enumeration of nested input files
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// create a configuration object
Configuration parameters = new Configuration();

// set the recursive enumeration parameter
parameters.setBoolean("recursive.file.enumeration", true);

// pass the configuration to the data source
DataSet<String> logs = env.readTextFile("file:///path/with.nested/files")
			  .withParameters(parameters);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

Data sources 从文件或 Java 集合中创建了初始的数据集。背后的机制请参考{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/InputFormat.java "InputFormat"%}。Flink 内建了多种常见格式以方便从常用文件中创建数据集。它们中的大多数都可以通过 *ExecutionEnvironment* 的函数直接使用。

File-based:

- `readTextFile(path)` / `TextInputFormat` - Reads files line wise and returns them as Strings.

- `readTextFileWithValue(path)` / `TextValueInputFormat` - Reads files line wise and returns them as
  StringValues. StringValues are mutable strings.

- `readCsvFile(path)` / `CsvInputFormat` - Parses files of comma (or another char) delimited fields.
  Returns a DataSet of tuples, case class objects, or POJOs. Supports the basic java types and their Value counterparts as field
  types.

- `readFileOfPrimitives(path, delimiter)` / `PrimitiveInputFormat` - Parses files of new-line (or another char sequence)
  delimited primitive data types such as `String` or `Integer` using the given delimiter.

- `readHadoopFile(FileInputFormat, Key, Value, path)` / `FileInputFormat` - Creates a JobConf and reads file from the specified
   path with the specified FileInputFormat, Key class and Value class and returns them as Tuple2<Key, Value>.

- `readSequenceFile(Key, Value, path)` / `SequenceFileInputFormat` - Creates a JobConf and reads file from the specified path with
   type SequenceFileInputFormat, Key class and Value class and returns them as Tuple2<Key, Value>.

Collection-based:

- `fromCollection(Seq)` - Creates a data set from a Seq. All elements
  in the collection must be of the same type.

- `fromCollection(Iterator)` - Creates a data set from an Iterator. The class specifies the
  data type of the elements returned by the iterator.

- `fromElements(elements: _*)` - Creates a data set from the given sequence of objects. All objects
  must be of the same type.

- `fromParallelCollection(SplittableIterator)` - Creates a data set from an iterator, in
  parallel. The class specifies the data type of the elements returned by the iterator.

- `generateSequence(from, to)` - Generates the squence of numbers in the given interval, in
  parallel.

Generic:

- `readFile(inputFormat, path)` / `FileInputFormat` - Accepts a file input format.

- `createInput(inputFormat)` / `InputFormat` - Accepts a generic input format.

**Examples**

{% highlight scala %}
val env  = ExecutionEnvironment.getExecutionEnvironment

// read text file from local files system
val localLines = env.readTextFile("file:///path/to/my/textfile")

// read text file from a HDFS running at nnHost:nnPort
val hdfsLines = env.readTextFile("hdfs://nnHost:nnPort/path/to/my/textfile")

// read a CSV file with three fields
val csvInput = env.readCsvFile[(Int, String, Double)]("hdfs:///the/CSV/file")

// read a CSV file with five fields, taking only two of them
val csvInput = env.readCsvFile[(String, Double)](
  "hdfs:///the/CSV/file",
  includedFields = Array(0, 3)) // take the first and the fourth field

// CSV input can also be used with Case Classes
case class MyCaseClass(str: String, dbl: Double)
val csvInput = env.readCsvFile[MyCaseClass](
  "hdfs:///the/CSV/file",
  includedFields = Array(0, 3)) // take the first and the fourth field

// read a CSV file with three fields into a POJO (Person) with corresponding fields
val csvInput = env.readCsvFile[Person](
  "hdfs:///the/CSV/file",
  pojoFields = Array("name", "age", "zipcode"))

// create a set from some given elements
val values = env.fromElements("Foo", "bar", "foobar", "fubar")

// generate a number sequence
val numbers = env.generateSequence(1, 10000000);

// read a file from the specified path of type TextInputFormat
val tuples = env.readHadoopFile(new TextInputFormat, classOf[LongWritable],
 classOf[Text], "hdfs://nnHost:nnPort/path/to/file")

// read a file from the specified path of type SequenceFileInputFormat
val tuples = env.readSequenceFile(classOf[IntWritable], classOf[Text],
 "hdfs://nnHost:nnPort/path/to/file")

{% endhighlight %}

#### Configuring CSV Parsing


Flink 提供一些 CSV 解析的配置参数：


- `lineDelimiter: String` specifies the delimiter of individual records. The default line delimiter is the new-line character `'\n'`.

- `fieldDelimiter: String` specifies the delimiter that separates fields of a record. The default field delimiter is the comma character `','`.

- `includeFields: Array[Int]` defines which fields to read from the input file (and which to ignore). By default the first *n* fields (as defined by the number of types in the `types()` call) are parsed.

- `pojoFields: Array[String]` specifies the fields of a POJO that are mapped to CSV fields. Parsers for CSV fields are automatically initialized based on the type and order of the POJO fields.

- `parseQuotedStrings: Character` enables quoted string parsing. Strings are parsed as quoted strings if the first character of the string field is the quote character (leading or tailing whitespaces are *not* trimmed). Field delimiters within quoted strings are ignored. Quoted string parsing fails if the last character of a quoted string field is not the quote character. If quoted string parsing is enabled and the first character of the field is *not* the quoting string, the string is parsed as unquoted string. By default, quoted string parsing is disabled.

- `ignoreComments: String` specifies a comment prefix. All lines that start with the specified comment prefix are not parsed and ignored. By default, no lines are ignored.

- `lenient: Boolean` enables lenient parsing, i.e., lines that cannot be correctly parsed are ignored. By default, lenient parsing is disabled and invalid lines raise an exception.

- `ignoreFirstLine: Boolean` configures the InputFormat to ignore the first line of the input file. By default no line is ignored.

#### Recursive Traversal of the Input Path Directory

遍历一个目录。对于基于文件的输入，如果输入的是一个目录，默认不会遍历子目录的文件。而是只会读取第一层目录的文件，忽略子目录的文件。可以通过设置`recursive.file.enumeration`来开启遍历子目录的设置。如下示例所示：

{% highlight scala %}
// enable recursive enumeration of nested input files
val env  = ExecutionEnvironment.getExecutionEnvironment

// create a configuration object
val parameters = new Configuration

// set the recursive enumeration parameter
parameters.setBoolean("recursive.file.enumeration", true)

// pass the configuration to the data source
env.readTextFile("file:///path/with.nested/files").withParameters(parameters)
{% endhighlight %}

</div>
</div>

### Read Compressed Files

Flink 对于一些扩展名确定的压缩文件能自动解压。也就是说不需要配置 input format 和 `FileInputFormat`来做压缩。注意，并不能并行来读取压缩文件，这样会影响scalability。


<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">压缩方法</th>
      <th class="text-left">文件后缀</th>
      <th class="text-left" style="width: 20%">可并行化</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>DEFLATE</strong></td>
      <td>`.deflate`</td>
      <td>no</td>
    </tr>
    <tr>
      <td><strong>GZip</strong></td>
      <td>`.gz`, `.gzip`</td>
      <td>no</td>
    </tr>
  </tbody>
</table>


{% top %}

Data Sinks
----------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">


Data sinks 消费 DataSets，用来存储或返回它们。Data sink 的操作可以用{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/OutputFormat.java "OutputFormat" %}来描述。Flink 在 DataSet 中内建了一些 output format：


- `writeAsText()` / `TextOuputFormat` - Writes elements line-wise as Strings. The Strings are
  obtained by calling the *toString()* method of each element.
- `writeAsFormattedText()` / `TextOutputFormat` - Write elements line-wise as Strings. The Strings
  are obtained by calling a user-defined *format()* method for each element.
- `writeAsCsv(...)` / `CsvOutputFormat` - Writes tuples as comma-separated value files. Row and field
  delimiters are configurable. The value for each field comes from the *toString()* method of the objects.
- `print()` / `printToErr()` / `print(String msg)` / `printToErr(String msg)` - Prints the *toString()* value
of each element on the standard out / strandard error stream. Optionally, a prefix (msg) can be provided which is
prepended to the output. This can help to distinguish between different calls to *print*. If the parallelism is
greater than 1, the output will also be prepended with the identifier of the task which produced the output.
- `write()` / `FileOutputFormat` - Method and base class for custom file outputs. Supports
  custom object-to-bytes conversion.
- `output()`/ `OutputFormat` - Most generic output method, for data sinks that are not file based
  (such as storing the result in a database).

一个 DataSet 可以输入到多个操作中。程序在数据集上运行其他转换的同时，还可以写或打印数据集。

**Examples**

标准的 data sink 函数：

{% highlight java %}
// text data
DataSet<String> textData = // [...]

// write DataSet to a file on the local file system
textData.writeAsText("file:///my/result/on/localFS");

// write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
textData.writeAsText("hdfs://nnHost:nnPort/my/result/on/localFS");

// write DataSet to a file and overwrite the file if it exists
textData.writeAsText("file:///my/result/on/localFS", WriteMode.OVERWRITE);

// tuples as lines with pipe as the separator "a|b|c"
DataSet<Tuple3<String, Integer, Double>> values = // [...]
values.writeAsCsv("file:///path/to/the/result/file", "\n", "|");

// this writes tuples in the text formatting "(a, b, c)", rather than as CSV lines
values.writeAsText("file:///path/to/the/result/file");

// this wites values as strings using a user-defined TextFormatter object
values.writeAsFormattedText("file:///path/to/the/result/file",
    new TextFormatter<Tuple2<Integer, Integer>>() {
        public String format (Tuple2<Integer, Integer> value) {
            return value.f1 + " - " + value.f0;
        }
    });
{% endhighlight %}

使用一个自定义的 output format：

{% highlight java %}
DataSet<Tuple3<String, Integer, Double>> myResult = [...]

// write Tuple DataSet to a relational database
myResult.output(
    // build and configure OutputFormat
    JDBCOutputFormat.buildJDBCOutputFormat()
                    .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
                    .setDBUrl("jdbc:derby:memory:persons")
                    .setQuery("insert into persons (name, age, height) values (?,?,?)")
                    .finish()
    );
{% endhighlight %}

#### Locally Sorted Output

data sink 的输出可以在某些字段上基于某个顺序做本地排序，可以使用[tuple field positions](#define-keys-for-tuples) 或 [field expressions](#define-keys-using-field-expressions)来指定 key。这对任何 output format 都有效。

下方例子展示了如何使用这一特性：

{% highlight java %}

DataSet<Tuple3<Integer, String, Double>> tData = // [...]
DataSet<Tuple2<BookPojo, Double>> pData = // [...]
DataSet<String> sData = // [...]

// sort output on String field in ascending order
tData.sortPartition(1, Order.ASCENDING).print();

// sort output on Double field in descending and Integer field in ascending order
tData.sortPartition(2, Order.DESCENDING).sortPartition(0, Order.ASCENDING).print();

// sort output on the "author" field of nested BookPojo in descending order
pData.sortPartition("f0.author", Order.DESCENDING).writeAsText(...);

// sort output on the full tuple in ascending order
tData.sortPartition("*", Order.ASCENDING).writeAsCsv(...);

// sort atomic type (String) output in descending order
sData.sortPartition("*", Order.DESCENDING).writeAsText(...);

{% endhighlight %}

目前还不支持全局的排序输出。

</div>
<div data-lang="scala" markdown="1">

Data sinks 消费 DataSets，用来存储或返回它们。Data sink 的操作可以用{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/OutputFormat.java "OutputFormat" %}来描述。Flink 在 DataSet 中内建了一些 output format：

- `writeAsText()` / `TextOuputFormat` - Writes elements line-wise as Strings. The Strings are
  obtained by calling the *toString()* method of each element.
- `writeAsCsv(...)` / `CsvOutputFormat` - Writes tuples as comma-separated value files. Row and field
  delimiters are configurable. The value for each field comes from the *toString()* method of the objects.
- `print()` / `printToErr()` - Prints the *toString()* value of each element on the
  standard out / strandard error stream.
- `write()` / `FileOutputFormat` - Method and base class for custom file outputs. Supports
  custom object-to-bytes conversion.
- `output()`/ `OutputFormat` - Most generic output method, for data sinks that are not file based
  (such as storing the result in a database).

一个 DataSet 可以输入到多个操作中。程序在数据集上运行其他转换的同时，还可以写或打印数据集。

**Examples**

标准的 data sink 函数：

{% highlight scala %}
// text data
val textData: DataSet[String] = // [...]

// write DataSet to a file on the local file system
textData.writeAsText("file:///my/result/on/localFS")

// write DataSet to a file on a HDFS with a namenode running at nnHost:nnPort
textData.writeAsText("hdfs://nnHost:nnPort/my/result/on/localFS")

// write DataSet to a file and overwrite the file if it exists
textData.writeAsText("file:///my/result/on/localFS", WriteMode.OVERWRITE)

// tuples as lines with pipe as the separator "a|b|c"
val values: DataSet[(String, Int, Double)] = // [...]
values.writeAsCsv("file:///path/to/the/result/file", "\n", "|")

// this writes tuples in the text formatting "(a, b, c)", rather than as CSV lines
values.writeAsText("file:///path/to/the/result/file");

// this wites values as strings using a user-defined formatting
values map { tuple => tuple._1 + " - " + tuple._2 }
  .writeAsText("file:///path/to/the/result/file")
{% endhighlight %}


#### Locally Sorted Output

data sink 的输出可以在某些字段上基于某个顺序做本地排序，可以使用[tuple field positions](#define-keys-for-tuples) 或 [field expressions](#define-keys-using-field-expressions)来指定 key。这对任何 output format 都有效。

{% highlight scala %}

val tData: DataSet[(Int, String, Double)] = // [...]
val pData: DataSet[(BookPojo, Double)] = // [...]
val sData: DataSet[String] = // [...]

// sort output on String field in ascending order
tData.sortPartition(1, Order.ASCENDING).print;

// sort output on Double field in descending and Int field in ascending order
tData.sortPartition(2, Order.DESCENDING).sortPartition(0, Order.ASCENDING).print;

// sort output on the "author" field of nested BookPojo in descending order
pData.sortPartition("_1.author", Order.DESCENDING).writeAsText(...);

// sort output on the full tuple in ascending order
tData.sortPartition("_", Order.ASCENDING).writeAsCsv(...);

// sort atomic type (String) output in descending order
sData.sortPartition("_", Order.DESCENDING).writeAsText(...);

{% endhighlight %}

目前还不支持全局的排序输出。

</div>
</div>

{% top %}


Iteration Operators
-------------------

迭代实现了 Flink 程序中的循环。迭代算子封装部分程序，并重复执行，将一次迭代的结果返回到下一次迭代中。 Flink 中有两种迭代：**BulkIteration** 和 **DeltaIteration**

本节给出了如何快速使用这两种迭代的示例。请参见 [迭代介绍](iterations.html) 了解更详细的说明。


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

#### Bulk Iterations

可以通过 DataSet 的 `iterate(int)` 方法来创建一个 BulkIteration, 它会返回`IterativeDataSet`， 这个`IterativeDataSet` 可以用来执行常见的转换。 该方法的参数表示执行迭代的最大次数。

在`IterativeDataSet`上调用`closeWith(DataSet)`来设定迭代的结束，并确定哪个转换会反馈给下一次迭代。
有一种可选方式是通过`closeWith(DataSet, DataSet)`，当该 DataSet 为空时，它会计算第二个 DataSet 并结束迭代。如果没有指定终止条件，迭代会执行给定的最大次数后再结束。

下例展示了如何计算 Pi。目标是统计落在单位圆中随机点的个数。在每次迭代中，会随机算则一个随机点。如果该点落在的单位圆中，我们会增加 count。Pi 就是通过最终的 count 除上乘以 4 的迭代次数。

{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// Create initial IterativeDataSet
IterativeDataSet<Integer> initial = env.fromElements(0).iterate(10000);

DataSet<Integer> iteration = initial.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer i) throws Exception {
        double x = Math.random();
        double y = Math.random();

        return i + ((x * x + y * y < 1) ? 1 : 0);
    }
});

// Iteratively transform the IterativeDataSet
DataSet<Integer> count = initial.closeWith(iteration);

count.map(new MapFunction<Integer, Double>() {
    @Override
    public Double map(Integer count) throws Exception {
        return count / (double) 10000 * 4;
    }
}).print();

env.execute("Iterative Pi Example");
{% endhighlight %}

你可以查看 {% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java "K-Means example" %}, 该示例使用了 BulkIteration 来对未标记的点进行聚类。


#### Delta Iterations

Delta 迭代解决了这种场景， 每一次迭代并不是改变数据的每一点。

在每一次迭代中，返回部分方案(称为workset)， delta 迭代维护跨迭代的状态(称为solution set), 这些状态通过deltas来更新。
迭代计算的结果就是最后迭代后的结果。 如果想了解delta 迭代的基本原则，请参考[迭代介绍](iterations.html)。

定义 DeltaIteration 的方式和定义 BulkIteration 很类似。对于 delta 迭代，两个数据集（workset和solution set）组成了每一次迭代的输入，新的 workset 和新的 soution set 做为每次迭代的输出。

调用`iterateDelta(DataSet, int, int)` 或 `iterateDelta(DataSet, int, int[])` 来创建一个DeltaIteration。该方法需要在初始 solution set 上调用。参数分别是初始数据集，最大迭代次数和key下标。返回的`DeltaIteration`对象， 
可以通过`iteration.getWorkset()` and `iteration.getSolutionSet()` 方法来访问 workset 和 solution set。

下方是 delta 迭代的一个示例：

{% highlight java %}
// read the initial data sets
DataSet<Tuple2<Long, Double>> initialSolutionSet = // [...]

DataSet<Tuple2<Long, Double>> initialDeltaSet = // [...]

int maxIterations = 100;
int keyPosition = 0;

DeltaIteration<Tuple2<Long, Double>, Tuple2<Long, Double>> iteration = initialSolutionSet
    .iterateDelta(initialDeltaSet, maxIterations, keyPosition);

DataSet<Tuple2<Long, Double>> candidateUpdates = iteration.getWorkset()
    .groupBy(1)
    .reduceGroup(new ComputeCandidateChanges());

DataSet<Tuple2<Long, Double>> deltas = candidateUpdates
    .join(iteration.getSolutionSet())
    .where(0)
    .equalTo(0)
    .with(new CompareChangesToCurrent());

DataSet<Tuple2<Long, Double>> nextWorkset = deltas
    .filter(new FilterByThreshold());

iteration.closeWith(deltas, nextWorkset)
	.writeAsCsv(outputPath);
{% endhighlight %}

</div>
<div data-lang="scala" markdown="1">

#### Bulk Iterations

可以通过 DataSet 的 `iterate(int)` 方法来创建一个 BulkIteration, 它会返回`IterativeDataSet`， 这个`IterativeDataSet` 可以用来执行常见的转换。 该方法的参数表示执行迭代的最大次数。

在`IterativeDataSet`上调用`closeWith(DataSet)`来设定迭代的结束，并确定哪个转换会反馈给下一次迭代。
有一种可选方式是通过`closeWith(DataSet, DataSet)`，当该 DataSet 为空时，它会计算第二个 DataSet 并结束迭代。如果没有指定终止条件，迭代会执行给定的最大次数后再结束。

下例展示了如何计算 Pi。目标是统计落在单位圆中随机点的个数。在每次迭代中，会随机算则一个随机点。如果该点落在的单位圆中，我们会增加 count。Pi 就是通过最终的 count 除上乘以 4 的迭代次数。

{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment()

// Create initial DataSet
val initial = env.fromElements(0)

val count = initial.iterate(10000) { iterationInput: DataSet[Int] =>
  val result = iterationInput.map { i =>
    val x = Math.random()
    val y = Math.random()
    i + (if (x * x + y * y < 1) 1 else 0)
  }
  result
}

val result = count map { c => c / 10000.0 * 4 }

result.print()

env.execute("Iterative Pi Example");
{% endhighlight %}


可以checkout 
{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java "K-Means example" %},
来查看更多BulkIteration操作

#### Delta Iterations

Delta 迭代解决了这种场景：每一次迭代并不是改变数据的每一点。

在每一次迭代中，返回部分方案(称为workset)， delta 迭代维护跨迭代的状态(称为solution set), 这些状态通过deltas来更新。
迭代计算的结果就是最后迭代后的结果。 如果想了解delta 迭代的基本原则，请参考[迭代介绍](iterations.html)。

定义 DeltaIteration 的方式和定义 BulkIteration 很类似。对于 delta 迭代，两个数据集（workset和solution set）组成了每一次迭代的输入，新的 workset 和新的 soution set 做为每次迭代的输出。

要创建一个 DeltaIteration，需要在初始 solution set 上调用 `iterateDelta(initialWorkset, maxIterations, key)`。迭代步骤函数需要两个参数：(solutionSet, workset)，并且必须返回两个值：(solutionSetDelta, newWorkset)。

下方是 delta 迭代的一个示例：

{% highlight scala %}
// read the initial data sets
val initialSolutionSet: DataSet[(Long, Double)] = // [...]

val initialWorkset: DataSet[(Long, Double)] = // [...]

val maxIterations = 100
val keyPosition = 0

val result = initialSolutionSet.iterateDelta(initialWorkset, maxIterations, Array(keyPosition)) {
  (solution, workset) =>
    val candidateUpdates = workset.groupBy(1).reduceGroup(new ComputeCandidateChanges())
    val deltas = candidateUpdates.join(solution).where(0).equalTo(0)(new CompareChangesToCurrent())

    val nextWorkset = deltas.filter(new FilterByThreshold())

    (deltas, nextWorkset)
}

result.writeAsCsv(outputPath)

env.execute()
{% endhighlight %}

</div>
</div>

{% top %}

>注：本文剩余部分未经校验，如有问题欢迎提issue

Operating on data objects in functions
--------------------------------------

flink的runtime 在用户函数内以java对象的方式来交换数据。 函数从runtime从参数中接受输入数据， 返回结果作为输出。 因为在用户函数可以访问这些对象， 必须注意用户代码访问，比如
读，修改这些对象的方式。

用户从flink runtime接收对象，可以像普通函数参数（`MapFunction`）或通过`Iterable`参数（像`GroupReduceFunction`）。 我们称runtime传来的object为*input objects*。 
用户函数可以通过函数返回值（比如`MapFunction`）或`Collector` (像`FlatMapFunction`)来emit对象。 我们称这些emitted的对象为*output objects*.

flik DataSet api 设定2种模式， 它会导致runtime 创建或reuse input object的方式不同。 它同样影响了用户代码如何和输入object和输出object 交互的方式。下面章节将介绍这些
限制并展示如何实现一个安全的用户函数。

### Object-Reuse Disabled (DEFAULT)


默认， flink 会禁止reuse object。 这种模式会保证在调用一个函数时，这个函数始终接收到新的对象。 这种禁止reuse方式会更安全使用。， 但会带来一定处理开销并引起更多的gc操作。 

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Guarantees and Restrictions</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Reading Input Objects</strong></td>
      <td>
        在函数内部， 它可以保证input object值不会变化。 这些包含的object 由一个iterable来提供。 举例来说， 收集iterable内的对象到一个list或map中是很安全的。注意， 这些对象可能在函数调用完被修改。所以跨函数调用时，
        记住这些历史对象是不安全的。
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Input Objects</strong></td>
      <td>可以修改输入的对象</td>
   </tr>
   <tr>
      <td><strong>Emitting Input Objects</strong></td>
      <td>
        可以emit 输入的对象。 输入的对象值可能在emit后发生变化。 所以读取一个已经emitted的输入对象是不安全的。
      </td>
   </tr>
   <tr>
      <td><strong>Reading Output Objects</strong></td>
      <td>
        传给collector的对象或函数返回值有可能发生变化， 因此读取一个output 对象是不安全的。
      </td>
   </tr>
   <tr>
      <td><strong>Modifying Output Objects</strong></td>
      <td>可以modify一个emitted的对象，然后再emit一次</td>
   </tr>
  </tbody>
</table>

**disable reuse 模式的原则:**

- 跨函数调用时， 不要remember和读取输入对象。
- 不要在emit完对象后，再读取它。


### Object-Reuse Enabled


在reuse 模式下， flink runtime 最小化对象的实例化次数。 这样可以提高性能和减少gc压力。 可以通过`ExecutionConfig.enableObjectReuse()`来打开reuse模式。 

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Operation</th>
      <th class="text-center">Guarantees and Restrictions</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>读取普通函数参数作为输入对象</strong></td>
      <td>
        普通函数的参数作为输入对象时， 这些对象在函数内没有被修改，但函数调用完可能被修改， 因此跨函数调用时记住这些对象是不安全的。
      </td>
   </tr>
   <tr>
      <td><strong>读取从迭代参数作为输入对象</strong></td>
      <td>
        当输入数据来自迭代器时， 输入对象仅仅当next（）函数被调用时有效。 一个iterable或iterator可能操作相同的对象很多次。 把葱iterable输入的数据放到一个list或map中是不安全的。
      </td>
   </tr>
   <tr>
      <td><strong>修改输入数据</strong></td>
      <td>千万不要修改输入的数据， 除非输入数据来自MapFunction, FlatMapFunction, MapPartitionFunction, GroupReduceFunction, GroupCombineFunction, CoGroupFunction, and InputFormat.next(reuse)</td>
   </tr>
   <tr>
      <td><strong>发送输入对象</strong></td>
      <td>
        千万不要发送输入数据，除非输入数据来自MapFunction, FlatMapFunction, MapPartitionFunction, GroupReduceFunction, GroupCombineFunction, CoGroupFunction, and InputFormat.next(reuse).</td>
      </td>
   </tr>
   <tr>
      <td><strong>读取输出对象</strong></td>
      <td>
        丢给collector或作为函数结果返回的对象可能会被修改， 因此读取一个output对象是不安全的。
      </td>
   </tr>
   <tr>
      <td><strong>修改输出对象</strong></td>
      <td>用户可以修改一个output对象并再次emit</td>
   </tr>
  </tbody>
</table>

**object reuse的原则:**

- 不要保留来自`Iterable`的输入对象.
- 在跨函数调用时，不要保留并读取输入对象.
- 不要修改或发送输入对象，除非输入对象来自`MapFunction`, `FlatMapFunction`, `MapPartitionFunction`, `GroupReduceFunction`, `GroupCombineFunction`, `CoGroupFunction`, and `InputFormat.next(reuse)`.
- 为了减少对象实例化次数， 用户可以一直发送一个专用的输出对象， 并且这个对象重复被修改但从不读取.

{% top %}

Debugging
---------


在分布式集群运行一个大型数据分析程序前， 最好是可以确认实现的算法能按预期的工作。 因此通常做法是，通过检查结果， debuggin和修正 不断逐步演进的过程。

flink 有个很好的简化开发数据分析程序的特性， 这个特性就是它制成在一个ide内本地调试程序， 注入测试程序，收集结果。

### Local Execution Environment


`LocalEnvironment` 会在一个单jvm 进程内启动flink系统。 如果用户是在ide里面启动LocalEnvironement， 用户就可以设置断点并轻松调试程序


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

DataSet<String> lines = env.readTextFile(pathToTextFile);
// build your program

env.execute();
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
val env = ExecutionEnvironment.createLocalEnvironment()

val lines = env.readTextFile(pathToTextFile)
// build your program

env.execute();
{% endhighlight %}
</div>
</div>

### Collection Data Sources and Sinks

通过创建输入文件并读取输出文件来提供分析程序的输入和检查结果有些笨重。 flink 可以提供一些特别的data source和data sink
，它们依赖java collection可以简化测试过程。 一旦一个程序完成测试， 它的source和sink可以很轻松被从外部存储比如hdfs上的
读取或写入 source和sink替换。

Collection data sources can be used as follows:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();

// Create a DataSet from a list of elements
DataSet<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataSet from any Java collection
List<Tuple2<String, Integer>> data = ...
DataSet<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataSet from an Iterator
Iterator<Long> longIt = ...
DataSet<Long> myLongs = env.fromCollection(longIt, Long.class);
{% endhighlight %}

A collection data sink is specified as follows:

{% highlight java %}
DataSet<Tuple2<String, Integer>> myResult = ...

List<Tuple2<String, Integer>> outData = new ArrayList<Tuple2<String, Integer>>();
myResult.output(new LocalCollectionOutputFormat(outData));
{% endhighlight %}

**Note:** 目前， coolection data sink只能在本地模式下使用，作为一个debug 工具

</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.createLocalEnvironment()

// Create a DataSet from a list of elements
val myInts = env.fromElements(1, 2, 3, 4, 5)

// Create a DataSet from any Collection
val data: Seq[(String, Int)] = ...
val myTuples = env.fromCollection(data)

// Create a DataSet from an Iterator
val longIt: Iterator[Long] = ...
val myLongs = env.fromCollection(longIt)
{% endhighlight %}
</div>
</div>

**Note:** 目前， collection data source 要求数据类型或iterator 实现`Serializable`。 更进一步说，
collection data sources不能够并行执行（parallelism 被限制为1）

{% top %}

Semantic Annotations
-----------


可用用语法注解来提示flink 一个函数的行为特征。 它可用告诉系统输入数据的哪些field 被读取并evaluate， 哪些字段不经修改就直接发送到输出。

语法注解是一个加速执行的有效途径， 因为它允许系统去判断重复使用排序顺序或者在多个操作之间进行partition。 使用语法注解可以节省程序
不必要的shuffle和不必要的sort，显著提高性能。

**注意** 使用语法注解是可选的。 保守提供语法注解非常重要。不正确的语法注解会让flink做出错误判断并导致错误结果。
如果一个操作的行为不是很清晰的提起预判， 不要提供语法注解。 请仔细阅读文档。

目前，下面的语法注解是支持的。

#### Forwarded Fields Annotation

转发字段注解。 转发字段注解定义了输入对象中哪些字段是在函数中不会被修改，直接转发到output中相同位置或其他位置。

这些信息可用来优化判断在sorting或partition中的数据属性在函数中保留下来。

对于像输入数据像组数据的函数，比如`GroupReduce`, `GroupCombine`, `CoGroup`, and `MapPartition`， 定义为转发的所有字段必须在相同的输入元素内联合转发。 
由组操作（group-wise）函数发送的每个元素的的转发字段可以由函数的输入group的不同元素来组成。

用[field expressions](#define-keys-using-field-expressions)来确定field转发信息。
在output中转发位置相同的filed由它们的位置来确定。
确定的位置必须是input中有效和houtput 中数据类型必须相同
举例来说， “f2”定义了java input tuple中第三个字段， 它同样等同于output tuple中第三个字段。


不做修改直接转发到其他位置的field， 通过“filed express”来定义。 比如"f0->f2"表示 java input tuple中第一个字段将不做修改直接copy到java 输出的第三个字段。 
“＊”可以表示整个输入或输出， 比如"f0->*" 表示函数的输出就是等同于java 输入tuple的第一个字段。


可以在一个string中定义多个字段转发 `"f0; f2->f1; f3->f2"`或者多个单独string比如`"f0", "f2->f1", "f3->f2"`。 
注解转发字段并不要求所有的转发字段都被定义， 但所有的定义都必须是正确的。

在函数定义前增加java注解可以用来定义转发字段，或者将它们作为函数参数。

##### Function Class Annotations

* `@ForwardedFields` 用于单一输入的函数比如map或reduce
* `@ForwardedFieldsFirst` 用于2个输入的函数比如join或cogroup的第一个输入
* `@ForwardedFieldsSecond` 用于2个输入的函数比如join或cogroup的第二个输入

##### Operator Arguments

* `data.map(myMapFnc).withForwardedFields()` 用于单一输入的函数比如map或reduce
* `data1.join(data2).where().equalTo().with(myJoinFnc).withForwardFieldsFirst()` 用于2个输入的函数比如join或cogroup的第一个输入
* `data1.join(data2).where().equalTo().with(myJoinFnc).withForwardFieldsSecond()` 用于2个输入的函数比如join或cogroup的第二个输入

注意， 不可能用函数参数的方式来覆盖class 注解定义的转发字段。

##### Example

The following example shows how to declare forwarded field information using a function class annotation:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@ForwardedFields("f0->f2")
public class MyMap implements
              MapFunction<Tuple2<Integer, Integer>, Tuple3<String, Integer, Integer>> {
  @Override
  public Tuple3<String, Integer, Integer> map(Tuple2<Integer, Integer> val) {
    return new Tuple3<String, Integer, Integer>("foo", val.f1 / 2, val.f0);
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
@ForwardedFields("_1->_3")
class MyMap extends MapFunction[(Int, Int), (String, Int, Int)]{
   def map(value: (Int, Int)): (String, Int, Int) = {
    return ("foo", value._2 / 2, value._1)
  }
}
{% endhighlight %}

</div>
</div>

#### Non-Forwarded Fields


定义非转发字段表示这些字段在函数输出相同位置上不再保留。 
其他字段将在输出的相同位置上保留。
因此 非转发字段就是转发字段功能的相反。
在一些组操作（group－wise）函数中，比如`GroupReduce`, `GroupCombine`, `CoGroup`, and `MapPartition` ， 非转发字段必须实现和转发字段相同的要求。

**IMPORTANT** 非转发字段的定义是可选的， 然而， 如果一旦使用， 其他字段就会被定义为forward。 因此相对来说，将一个转发字段定义为非转发字段会更安全一点。


用[field expressions](#define-keys-using-field-expressions) list来表示非转发字段。 可以是`"f1; f3"` 和 `"f1", "f3"`， 
一个语句多个字段或多个独立语句组成。 非转发字段要求函数输入和输出类型相同。

用类注解的方式定义非转发字段

* `@NonForwardedFields` 用于单一输入的函数比如map或reduce
* `@NonForwardedFieldsFirst` 用于2个输入的函数比如join或cogroup的第一个输入
* `@NonForwardedFieldsSecond` 用于2个输入的函数比如join或cogroup的第二个输入

##### Example

The following example shows how to declare non-forwarded field information:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@NonForwardedFields("f1") // second field is not forwarded
public class MyMap implements
              MapFunction<Tuple2<Integer, Integer>, Tuple2<Integer, Integer>> {
  @Override
  public Tuple2<Integer, Integer> map(Tuple2<Integer, Integer> val) {
    return new Tuple2<Integer, Integer>(val.f0, val.f1 / 2);
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
@NonForwardedFields("_2") // second field is not forwarded
class MyMap extends MapFunction[(Int, Int), (Int, Int)]{
  def map(value: (Int, Int)): (Int, Int) = {
    return (value._1, value._2 / 2)
  }
}
{% endhighlight %}

</div>
</div>

#### Read Fields

定义一个函数将会访问并evaluate的读取字段， 举例 函数的所有字段都用来计算结果。
在条件语句中evalue的字段或用来计算结果的字段必须标记为read，当确定读字段信息时。
那些不做修改直接转发给输出的字段或者不访问的字段将不被考虑为读。

**IMPORTANT**: 
定义读字段操作是可选的， 然而，一旦使用， 则所有的读字段必须明确。 定义一个非读字段为读其实会比较安全。

Read fields are specified as a list of [field expressions](#define-keys-using-field-expressions). The list can be either given as a single String with field expressions separated by semicolons or as multiple Strings.
For example both `"f1; f3"` and `"f1", "f3"` declare that the second and fourth field of a Java tuple are read and evaluated by the function.

Read field information is specified as function class annotations using the following annotations:

* `@ReadFields` for single input functions such as Map and Reduce.
* `@ReadFieldsFirst` for the first input of a function with two inputs such as Join and CoGroup.
* `@ReadFieldsSecond` for the second input of a function with two inputs such as Join and CoGroup.

##### Example

The following example shows how to declare read field information:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
@ReadFields("f0; f3") // f0 and f3 are read and evaluated by the function.
public class MyMap implements
              MapFunction<Tuple4<Integer, Integer, Integer, Integer>,
                          Tuple2<Integer, Integer>> {
  @Override
  public Tuple2<Integer, Integer> map(Tuple4<Integer, Integer, Integer, Integer> val) {
    if(val.f0 == 42) {
      return new Tuple2<Integer, Integer>(val.f0, val.f1);
    } else {
      return new Tuple2<Integer, Integer>(val.f3+10, val.f1);
    }
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
@ReadFields("_1; _4") // _1 and _4 are read and evaluated by the function.
class MyMap extends MapFunction[(Int, Int, Int, Int), (Int, Int)]{
   def map(value: (Int, Int, Int, Int)): (Int, Int) = {
    if (value._1 == 42) {
      return (value._1, value._2)
    } else {
      return (value._4 + 10, value._2)
    }
  }
}
{% endhighlight %}

</div>
</div>

{% top %}


Broadcast Variables
-------------------


广播变量允许用户在所有的并发实例的函数中操作一个data set。 这种特性对辅助的data set或数据依赖参数信息非常有用。
这个data set在函数中可以成为一个collection。

- **Broadcast**: 广播set通过`withBroadcastSet(DataSet, String)`来注册
- **Access**: 在一个函数内通过`getRuntimeContext().getBroadcastVariable(String)`来访问 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// 1. The DataSet to be broadcasted
DataSet<Integer> toBroadcast = env.fromElements(1, 2, 3);

DataSet<String> data = env.fromElements("a", "b");

data.map(new RichMapFunction<String, String>() {
    @Override
    public void open(Configuration parameters) throws Exception {
      // 3. Access the broadcasted DataSet as a Collection
      Collection<Integer> broadcastSet = getRuntimeContext().getBroadcastVariable("broadcastSetName");
    }


    @Override
    public String map(String value) throws Exception {
        ...
    }
}).withBroadcastSet(toBroadcast, "broadcastSetName"); // 2. Broadcast the DataSet
{% endhighlight %}

必须确保这个名字(在上一例子中`broadcastSetName` )在注册和使用中，二者是匹配的. 完整的例子可以参考
{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java#L96 "K-Means Algorithm" %}.
</div>
<div data-lang="scala" markdown="1">

{% highlight scala %}
// 1. The DataSet to be broadcasted
val toBroadcast = env.fromElements(1, 2, 3)

val data = env.fromElements("a", "b")

data.map(new RichMapFunction[String, String]() {
    var broadcastSet: Traversable[String] = null

    override def open(config: Configuration): Unit = {
      // 3. Access the broadcasted DataSet as a Collection
      broadcastSet = getRuntimeContext().getBroadcastVariable[String]("broadcastSetName").asScala
    }

    def map(in: String): String = {
        ...
    }
}).withBroadcastSet(toBroadcast, "broadcastSetName") // 2. Broadcast the DataSet
{% endhighlight %}


必须确保这个名字(在上一例子中`broadcastSetName` )在注册和使用中，二者是匹配的. 完整的例子可以参考
{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java#L96 "K-Means Algorithm" %}.
</div>
</div>

**Note**: 
因为broadcaset 变量是维持在每个节点的内存中， 数据量不能太大。 对于一些简单的事情， 用户可以用将函数参数化， 或使用`withParameters(...)`来在配置中传递。

{% top %}

Passing Parameters to Functions
-------------------

传递给函数的参数可以使用构造函数或`withParameters(Configuration)` 。 这些参数被序列化作为函数对象的部分并被发送到每个task实例

可以查看更多 [best practices guide on how to pass command line arguments to functions]({{ site.baseurl }}/apis/best_practices.html#parsing-command-line-arguments-and-passing-them-around-in-your-flink-application).


#### Via Constructor

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataSet<Integer> toFilter = env.fromElements(1, 2, 3);

toFilter.filter(new MyFilter(2));

private static class MyFilter implements FilterFunction<Integer> {

  private final int limit;

  public MyFilter(int limit) {
    this.limit = limit;
  }

  @Override
  public boolean filter(Integer value) throws Exception {
    return value > limit;
  }
}
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val toFilter = env.fromElements(1, 2, 3)

toFilter.filter(new MyFilter(2))

class MyFilter(limit: Int) extends FilterFunction[Int] {
  override def filter(value: Int): Boolean = {
    value > limit
  }
}
{% endhighlight %}
</div>
</div>

#### Via `withParameters(Configuration)`

这个函数用一个configuration对象作为参数， 它会被传递给[rich function](#rich-functions)'s `open()`。 configuration对象是一个map， 
key类型是string，value是其他类型。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataSet<Integer> toFilter = env.fromElements(1, 2, 3);

Configuration config = new Configuration();
config.setInteger("limit", 2);

toFilter.filter(new RichFilterFunction<Integer>() {
    private int limit;

    @Override
    public void open(Configuration parameters) throws Exception {
      limit = parameters.getInteger("limit", 0);
    }

    @Override
    public boolean filter(Integer value) throws Exception {
      return value > limit;
    }
}).withParameters(config);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val toFilter = env.fromElements(1, 2, 3)

val c = new Configuration()
c.setInteger("limit", 2)

toFilter.filter(new RichFilterFunction[Int]() {
    var limit = 0

    override def open(config: Configuration): Unit = {
      limit = config.getInteger("limit", 0)
    }

    def filter(in: Int): Boolean = {
        in > limit
    }
}).withParameters(c)
{% endhighlight %}
</div>
</div>

#### Globally via the `ExecutionConfig`


flink 同样允许传递实现`ExecutionConfig`接口的自定义对象到环境中。 这个执行config在所有的rich 用户函数中都是可以访问的， 它是所有函数都可以获取。

**Setting a custom global configuration**

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

Configuration conf = new Configuration();
conf.setString("mykey","myvalue");
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(conf);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = ExecutionEnvironment.getExecutionEnvironment
val conf = new Configuration()
conf.setString("mykey", "myvalue")
env.getConfig.setGlobalJobParameters(conf)
{% endhighlight %}
</div>
</div>


可以通过一个继承`ExecutionConfig.GlobalJobParameters`类的自定义configuration。它的内容同样可以在web ui上显示


**Accessing values from the global configuration**


在系统中很多地方可以访问全局任务参数对象。 所有实现`Rich*Function` 接口的函数都可以访问这些对象通过runtime 的context。

{% highlight java %}
public static final class Tokenizer extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

    private String mykey;
    @Override
    public void open(Configuration parameters) throws Exception {
      super.open(parameters);
      ExecutionConfig.GlobalJobParameters globalParams = getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
      Configuration globConf = (Configuration) globalParams;
      mykey = globConf.getString("mykey", null);
    }
    // ... more here ...
{% endhighlight %}

{% top %}
