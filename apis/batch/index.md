---
title: "Flink DataSet API Programming Guide"

# Top-level navigation
top-nav-group: apis
top-nav-pos: 3
top-nav-title: <strong>Batch Guide</strong> (DataSet API)

# Sub-level navigation
sub-nav-group: batch
sub-nav-group-title: Batch Guide
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


DataSet 程序是flink中常用程序，这些程序一般会对数据集执行转换操作，比如filtering／mapping／joining／grouping。 这些数据集一般从特定的source（文件或本地collection集合）中创建出来，通过sink输出结果，比如写数据到（分布式）文件中，或标准输出。 flink运行在各种context下，standalone或嵌入到其他程序中。可能在本地jvm中或多个机器的集群中执行。

如果想要学习DataSet,建议从[basic concepts]({{ site.baseurl }}/apis/common/index.html)和[anatomy of a Flink Program]({{ site.baseurl }}/apis/common/index.html#anatomy-of-a-flink-program)开始入手，并逐步增加自己的[transformations](#dataset-transformations)操作。其他章节讲介绍额外的操作或高级特性。

* This will be replaced by the TOC
{:toc}

Example Program
---------------


下面是一个完整的wordcount例子。 只需要把flink library加入到项目中即可（参考[Linking with Flink](#linking-with-flink)）

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

DataSet Transformations
-----------------------


Transformations 是将一个或多个DataSet转化为一个新的DataSet。 程序将巧妙的把多个transformations打包在一起。 这里给一个简单的介绍，详情可以参考[transformations
documentation](dataset_transformations.html)， 那里会有详细介绍并给出example.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

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
        <p>Takes one element and produces one element.</p>
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
        <p>Takes one element and produces zero, one, or more elements. </p>
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
        <p>在一个函数内对一个分区进行transform。Transforms a parallel partition in a single function call. The function get the partition
        as an `Iterable` stream and can produce an arbitrary number of result values. The number of
        elements in each partition depends on the degree-of-parallelism and previous operations.</p>
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
        <p>执行过滤操作， 对每个元素进行判断， 函数返回为true的元素会加入到新的DataSet中。Evaluates a boolean function for each element and retains those for which the function
        returns true.<br/>

        <strong>IMPORTANT:</strong> The system assumes that the function does not modify the elements on which the predicate is applied. Violating this assumption
        can lead to incorrect results.
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
        <p>将一组元素合并为一个单个元素，通过不断重复执行合并2个元素到一个元素的操作。Combines a group of elements into a single element by repeatedly combining two elements
        into one. Reduce may be applied on a full data set, or on a grouped data set.</p>
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
        <p>将一组元素合并为一个或多个元素。Combines a group of elements into one or more elements. ReduceGroup may be applied on a
        full data set, or on a grouped data set.</p>
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
        <p>将一组值合并为一个值中。Aggregates a group of values into a single value. Aggregation functions can be thought of
        as built-in reduce functions. Aggregate may be applied on a full data set, or on a grouped
        data set.</p>
{% highlight java %}
Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.aggregate(SUM, 0).and(MIN, 2);
{% endhighlight %}
	<p>You can also use short-hand syntax for minimum, maximum, and sum aggregations.</p>
	{% highlight java %}
	Dataset<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input.sum(0).andMin(2);
	{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Distinct</strong></td>
      <td>
        <p>对一个DataSet 去掉重复元素。Returns the distinct elements of a data set. It removes the duplicate entries
        from the input DataSet, with respect to all fields of the elements, or a subset of fields.</p>
    {% highlight java %}
        data.distinct();
    {% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Join</strong></td>
      <td>
        对2个data set进行join， 基于相同key生成pairs。Joins two data sets by creating all pairs of elements that are equal on their keys.
        Optionally uses a JoinFunction to turn the pair of elements into a single element, or a
        FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)
        elements. See the <a href="#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight java %}
result = input1.join(input2)
               .where(0)       // key of the first input (tuple field 0)
               .equalTo(1);    // key of the second input (tuple field 1)
{% endhighlight %}
        You can specify the way that the runtime executes the join via <i>Join Hints</i>. The hints
        describe whether the join happens through partitioning or broadcasting, and whether it uses
        a sort-based or a hash-based algorithm. Please refer to the
        <a href="dataset_transformations.html#join-algorithm-hints">Transformations Guide</a> for
        a list of possible hints and an example.</br>
        If no hint is specified, the system will try to make an estimate of the input sizes and
        pick a the best strategy according to those estimates.
{% highlight java %}
// This executes a join by broadcasting the first data set
// using a hash table for the broadcasted data
result = input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
               .where(0).equalTo(1);
{% endhighlight %}
        Note that the join transformation works only for equi-joins. Other join types need to be expressed using OuterJoin or CoGroup.
      </td>
    </tr>

    <tr>
      <td><strong>OuterJoin</strong></td>
      <td>
        对2个data set进行left／right／full－outer join， 类似join函数（inner join），基于相同key生成pairs。Performs a left, right, or full outer join on two data sets. Outer joins are similar to regular (inner) joins and create all pairs of elements that are equal on their keys. In addition, records of the "outer" side (left, right, or both in case of full) are preserved if no matching key is found in the other side. Matching pairs of elements (or one element and a `null` value for the other input) are given to a JoinFunction to turn the pair of elements into a single element, or to a FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)         elements. See the <a href="#specifying-keys">keys section</a> to learn how to define join keys.
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
        <p>对两维数据进行reduce操作， 对一个或多个filed进行group操作，然后进行join这些group。The two-dimensional variant of the reduce operation. Groups each input on one or more
        fields and then joins the groups. The transformation function is called per pair of groups.
        See the <a href="#specifying-keys">keys section</a> to learn how to define coGroup keys.</p>
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
        <p>对2个输入进行cros操作，创建元素的所有pair。Builds the Cartesian product (cross product) of two inputs, creating all pairs of
        elements. Optionally uses a CrossFunction to turn the pair of elements into a single
        element</p>
{% highlight java %}
DataSet<Integer> data1 = // [...]
DataSet<String> data2 = // [...]
DataSet<Tuple2<Integer, String>> result = data1.cross(data2);
{% endhighlight %}
      <p>Note: Cross is potentially a <b>very</b> compute-intensive operation which can challenge even large compute clusters! It is adviced to hint the system with the DataSet sizes by using <i>crossWithTiny()</i> and <i>crossWithHuge()</i>.</p>
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>当一个dataset不能满足数据需求时，需要加入其他data set的数据。Produces the union of two data sets. This operation happens implicitly if more than one
        data set is used for a specific function input.</p>
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
        <p>主要是解决在多分区情况下，数据倾斜问题。Evenly rebalances the parallel partitions of a data set to eliminate data skew. Only Map-like transformations may follow a rebalance transformation.</p>
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
        <p>在某个key上执行hash partitions。Hash-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
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
        <p>在某个key上range partition。 Range-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
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
        <p>自定义parition操作。Manually specify a partitioning over the data.
          <br/>
          <i>Note</i>: This method works only on single field keys.</p>
{% highlight java %}
DataSet<Tuple2<String,Integer>> in = // [...]
DataSet<Integer> result = in.partitionCustom(Partitioner<K> partitioner, key)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Sort Partition</strong></td>
      <td>
        <p>在某个字段上本地sort 所有分区的数据。Locally sorts all partitions of a data set on a specified field in a specified order.
          Fields can be specified as tuple positions or field expressions.
          Sorting on multiple fields is done by chaining sortPartition() calls.</p>
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
        <p>返回一个data set的前n个元素。 Returns the first n (arbitrary) elements of a data set. First-n can be applied on a regular data set, a grouped data set, or a grouped-sorted data set. Grouping keys can be specified as key-selector functions or field position keys.</p>
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

The following transformations are available on data sets of Tuples:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Transformation</th>
      <th class="text-center">Description</th>
    </tr>
  </thead>
  <tbody>
   <tr>
      <td><strong>Project</strong></td>
      <td>
        <p>选择tuple field子集组成新的data set。Selects a subset of fields from the tuples</p>
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
        <p>输入一个元素输出一个元素。 Takes one element and produces one element.</p>
{% highlight scala %}
data.map { x => x.toInt }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>FlatMap</strong></td>
      <td>
        <p>输入一个元素，产生0个或1个或多个元素。Takes one element and produces zero, one, or more elements. </p>
{% highlight scala %}
data.flatMap { str => str.split(" ") }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>MapPartition</strong></td>
      <td>
        <p>在一个函数内对一个分区进行transform。Transforms a parallel partition in a single function call. The function get the partition
        as an `Iterator` and can produce an arbitrary number of result values. The number of
        elements in each partition depends on the degree-of-parallelism and previous operations.</p>
{% highlight scala %}
data.mapPartition { in => in map { (_, 1) } }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Filter</strong></td>
      <td>
        <p>执行过滤操作， 对每个元素进行判断， 函数返回为true的元素会加入到新的DataSet中。Evaluates a boolean function for each element and retains those for which the function
        returns true.<br/>
        <strong>IMPORTANT:</strong> The system assumes that the function does not modify the element on which the predicate is applied.
        Violating this assumption can lead to incorrect results.</p>
{% highlight scala %}
data.filter { _ > 1000 }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Reduce</strong></td>
      <td>
        <p>将一组元素合并为一个单个元素，通过不断重复执行合并2个元素到一个元素的操作。Combines a group of elements into a single element by repeatedly combining two elements
        into one. Reduce may be applied on a full data set, or on a grouped data set.</p>
{% highlight scala %}
data.reduce { _ + _ }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>ReduceGroup</strong></td>
      <td>
        <p>将一组元素合并为一个或多个元素。Combines a group of elements into one or more elements. ReduceGroup may be applied on a
        full data set, or on a grouped data set.</p>
{% highlight scala %}
data.reduceGroup { elements => elements.sum }
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Aggregate</strong></td>
      <td>
        <p>将一组值合并为一个值中。Aggregates a group of values into a single value. Aggregation functions can be thought of
        as built-in reduce functions. Aggregate may be applied on a full data set, or on a grouped
        data set.</p>
{% highlight scala %}
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Doublr)] = input.aggregate(SUM, 0).aggregate(MIN, 2);
{% endhighlight %}
  <p>You can also use short-hand syntax for minimum, maximum, and sum aggregations.</p>
{% highlight scala %}
val input: DataSet[(Int, String, Double)] = // [...]
val output: DataSet[(Int, String, Doublr)] = input.sum(0).min(2)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Distinct</strong></td>
      <td>
        <p>对一个DataSet 去掉重复元素。Returns the distinct elements of a data set. It removes the duplicate entries
        from the input DataSet, with respect to all fields of the elements, or a subset of fields.</p>
      {% highlight scala %}
         data.distinct()
      {% endhighlight %}
      </td>
    </tr>

    </tr>
      <td><strong>Join</strong></td>
      <td>
        对2个data set进行join， 基于相同key生成pairs。Joins two data sets by creating all pairs of elements that are equal on their keys.
        Optionally uses a JoinFunction to turn the pair of elements into a single element, or a
        FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)
        elements. See the <a href="#specifying-keys">keys section</a> to learn how to define join keys.
{% highlight scala %}
// In this case tuple fields are used as keys. "0" is the join field on the first tuple
// "1" is the join field on the second tuple.
val result = input1.join(input2).where(0).equalTo(1)
{% endhighlight %}
        You can specify the way that the runtime executes the join via <i>Join Hints</i>. The hints
        describe whether the join happens through partitioning or broadcasting, and whether it uses
        a sort-based or a hash-based algorithm. Please refer to the
        <a href="dataset_transformations.html#join-algorithm-hints">Transformations Guide</a> for
        a list of possible hints and an example.</br>
        If no hint is specified, the system will try to make an estimate of the input sizes and
        pick a the best strategy according to those estimates.
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
        对2个data set进行left／right／full－outer join， 类似join（inner），基于相同key生成pairs。Performs a left, right, or full outer join on two data sets. Outer joins are similar to regular (inner) joins and create all pairs of elements that are equal on their keys. In addition, records of the "outer" side (left, right, or both in case of full) are preserved if no matching key is found in the other side. Matching pairs of elements (or one element and a `null` value for the other input) are given to a JoinFunction to turn the pair of elements into a single element, or to a FlatJoinFunction to turn the pair of elements into arbitrarily many (including none)         elements. See the <a href="#specifying-keys">keys section</a> to learn how to define join keys.
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
        <p>对两维数据进行reduce操作， 对一个或多个filed进行group操作，然后进行join这些group。The two-dimensional variant of the reduce operation. Groups each input on one or more
        fields and then joins the groups. The transformation function is called per pair of groups.
        See the <a href="#specifying-keys">keys section</a> to learn how to define coGroup keys.</p>
{% highlight scala %}
data1.coGroup(data2).where(0).equalTo(1)
{% endhighlight %}
      </td>
    </tr>

    <tr>
      <td><strong>Cross</strong></td>
      <td>
        <p>基于两个输入做cross操作。Builds the Cartesian product (cross product) of two inputs, creating all pairs of
        elements. Optionally uses a CrossFunction to turn the pair of elements into a single
        element</p>
{% highlight scala %}
val data1: DataSet[Int] = // [...]
val data2: DataSet[String] = // [...]
val result: DataSet[(Int, String)] = data1.cross(data2)
{% endhighlight %}
        <p>Note: Cross is potentially a <b>very</b> compute-intensive operation which can challenge even large compute clusters! It is adviced to hint the system with the DataSet sizes by using <i>crossWithTiny()</i> and <i>crossWithHuge()</i>.</p>
      </td>
    </tr>
    <tr>
      <td><strong>Union</strong></td>
      <td>
        <p>合并2个data set。Produces the union of two data sets.</p>
{% highlight scala %}
data.union(data2)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Rebalance</strong></td>
      <td>
        <p>主要是解决在多分区情况下，数据倾斜问题。Evenly rebalances the parallel partitions of a data set to eliminate data skew. Only Map-like transformations may follow a rebalance transformation.</p>
{% highlight scala %}
val data1: DataSet[Int] = // [...]
val result: DataSet[(Int, String)] = data1.rebalance().map(...)
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Hash-Partition</strong></td>
      <td>
        <p>在某个key上执行hash partitions。Hash-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.partitionByHash(0).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>Range-Partition</strong></td>
      <td>
        <p>在一个指定的key上作range parition。Range-partitions a data set on a given key. Keys can be specified as position keys, expression keys, and key selector functions.</p>
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
        <p>自定义partition。Manually specify a partitioning over the data.
          <br/>
          <i>Note</i>: This method works only on single field keys.</p>
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
        <p>在某个字段上本地sort 所有分区的数据。Locally sorts all partitions of a data set on a specified field in a specified order.
          Fields can be specified as tuple positions or field expressions.
          Sorting on multiple fields is done by chaining sortPartition() calls.</p>
{% highlight scala %}
val in: DataSet[(Int, String)] = // [...]
val result = in.sortPartition(1, Order.ASCENDING).mapPartition { ... }
{% endhighlight %}
      </td>
    </tr>
    <tr>
      <td><strong>First-n</strong></td>
      <td>
        <p>返回一个data set的first n个元素。Returns the first n (arbitrary) elements of a data set. First-n can be applied on a regular data set, a grouped data set, or a grouped-sorted data set. Grouping keys can be specified as key-selector functions,
        tuple positions or case class fields.</p>
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


当对一个transformation自定义一个名字时， 可以通过`setParallelism(int)`来设置transformation的[parallelism](#parallel-execution), 这种方式可以帮组debug。 DataSource/DataSinks 都可以使用。

传递给`withParameters(Configuration)`的configuration对象， 可以在用户函数内的open函数中被访问。

{% top %}

Data Sources
------------

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">


Data sources 从文件或java collection中创建初始的data set。 背后的机制请参考{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/InputFormat.java "InputFormat"%}.

Flink 内建了常见文件格式。 可以从*ExecutionEnvironment*函数直接使用。

File-based:

- `readTextFile(path)` / `TextInputFormat` - 按行读取文件并返回strings. Reads files line wise and returns them as Strings.

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

Flink 提供一些csv解析的配置:

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


遍历一个目录。对于基于文件的输入， 如果输入的是一个目录， 默认不会遍历子目录的文件。默认，只会读取第一层目录的文件， 而忽略子目录的文件。 可以通过设置`recursive.file.enumeration`来打开遍历子目录的设置。


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


Data sources 从文件或java collection中创建初始的data set。 背后的机制请参考{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/InputFormat.java "InputFormat"%}.

Flink 内建了常见文件格式。 可以从*ExecutionEnvironment*函数直接使用。

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


Flink 提供一些csv 解析的配置参数：


- `lineDelimiter: String` specifies the delimiter of individual records. The default line delimiter is the new-line character `'\n'`.

- `fieldDelimiter: String` specifies the delimiter that separates fields of a record. The default field delimiter is the comma character `','`.

- `includeFields: Array[Int]` defines which fields to read from the input file (and which to ignore). By default the first *n* fields (as defined by the number of types in the `types()` call) are parsed.

- `pojoFields: Array[String]` specifies the fields of a POJO that are mapped to CSV fields. Parsers for CSV fields are automatically initialized based on the type and order of the POJO fields.

- `parseQuotedStrings: Character` enables quoted string parsing. Strings are parsed as quoted strings if the first character of the string field is the quote character (leading or tailing whitespaces are *not* trimmed). Field delimiters within quoted strings are ignored. Quoted string parsing fails if the last character of a quoted string field is not the quote character. If quoted string parsing is enabled and the first character of the field is *not* the quoting string, the string is parsed as unquoted string. By default, quoted string parsing is disabled.

- `ignoreComments: String` specifies a comment prefix. All lines that start with the specified comment prefix are not parsed and ignored. By default, no lines are ignored.

- `lenient: Boolean` enables lenient parsing, i.e., lines that cannot be correctly parsed are ignored. By default, lenient parsing is disabled and invalid lines raise an exception.

- `ignoreFirstLine: Boolean` configures the InputFormat to ignore the first line of the input file. By default no line is ignored.

#### Recursive Traversal of the Input Path Directory


遍历一个目录。对于基于文件的输入， 如果输入的是一个目录， 默认不会遍历子目录的文件。默认，只会读取第一层目录的文件， 而忽略子目录的文件。 可以通过设置`recursive.file.enumeration`来打开遍历子目录的设置。

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


flink对于一些扩展名确定的压缩文件自动解压。 也就是说不需要配置input format和 `FileInputFormat`来做压缩。 注意， 并不能并行来读取压缩文件，这样会影响scalability。

<br />

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">Compression method</th>
      <th class="text-left">File extensions</th>
      <th class="text-left" style="width: 20%">Parallelizable</th>
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


Data sinks 消费DataSets, 存储并返回他们。Data sink的操作可以用{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/OutputFormat.java "OutputFormat" %}来描叙。 
flink 内建了一些output format：


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


一个Dataset可以输出多个操作， 程序可以写或打印一个data set，并同时运行transform。

**Examples**

标准data sink 函数：

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

使用一个自定义的output format

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


data sink的output 能够在某些field 上基于某个顺序做本地sort， 通过[tuple field positions](#define-keys-for-tuples)
 or [field expressions](#define-keys-using-field-expressions)。

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

然而还不支持全局sort output。

</div>
<div data-lang="scala" markdown="1">

Data sinks 消费DataSets, 存储并返回他们。Data sink的操作可以用
{% gh_link /flink-core/src/main/java/org/apache/flink/api/common/io/OutputFormat.java "OutputFormat" %}来描叙。 
flink 内建了一些output format：

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

A DataSet can be input to multiple operations. Programs can write or print a data set and at the
same time run additional transformations on them.

**Examples**

标准data sink 函数：

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

data sink的output 能够在某些field 上基于某个顺序做本地sort， 通过[tuple field positions](#define-keys-for-tuples)
 or [field expressions](#define-keys-using-field-expressions)。

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

Globally sorted output is not supported yet.

</div>
</div>

{% top %}


Iteration Operators
-------------------

Iterations implement loops in Flink programs. The iteration operators encapsulate a part of the
program and execute it repeatedly, feeding back the result of one iteration (the partial solution)
into the next iteration. There are two types of iterations in Flink: **BulkIteration** and
**DeltaIteration**.

This section provides quick examples on how to use both operators. Check out the [Introduction to
Iterations](iterations.html) page for a more detailed introduction.

迭代实现flink程序里面的循环。 迭代操作封装部分程序，并重复执行， 将依次迭代的结果输入到下一次迭代中。 
flink中有2种迭代**BulkIteration** 和**DeltaIteration**

本章提供这2种迭代的example， 详情可以参考Check out the [Introduction toIterations](iterations.html) 


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

#### Bulk Iterations

可以通过dataset的`iterate(int)`来创建一个BulkIteration, 它会返回`IterativeDataSet`， 这个`IterativeDataSet` 
可以用来执行常见的transformation。 唯一的参数表示执行迭代的最大次数。

在`IterativeDataSet`上调用`closeWith(DataSet)`来设定迭代的结束， 并会确定哪个transformation会反馈给下一次迭代。
有一种可选方式是通过`closeWith(DataSet, DataSet)`， 当第一个dataset 为空时，它会结束迭代并evaluate 第二个dataset
如果结束条件没有触发， 迭代会执行最大次数后再结束。

下例展示了如何计算pi. 

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

可以checkout 
{% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/clustering/KMeans.java "K-Means example" %},
来查看更多BulkIteration操作


#### Delta Iterations



delta 迭代解决了这种场景， 每一次迭代并不是改变数据的每一点。

在每一次迭代中，返回部分方案(称为workset)， delta 迭代维护跨迭代的状态(称为solution set), 这些状态通过deltas来更新。
迭代计算的结果就是最后迭代后的结果。 如果想了解delta 迭代的基本原则，请参考[Introduction to Iterations](iterations.html)。

定义DeltaIteration方式和定义BulkIteration 很类似。 对于delta迭代， 2个data set（workset和solution set）组成了
每一次迭代的输入， 新的workset和新的soution set做为每次迭代的输出。

调用`iterateDelta(DataSet, int, int)` 或 `iterateDelta(DataSet, int, int[])` 来创建一个DeltaIteration。
在最开始的solution set上调用这个函数。 参数是起始data set， 最大迭代次数和key的position。 返回`DeltaIteration`， 
可以通过`iteration.getWorkset()` and `iteration.getSolutionSet()` 来拿到workset和solution set。


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

可以通过dataset的`iterate(int)`来创建一个BulkIteration, 它会返回`IterativeDataSet`， 这个`IterativeDataSet` 
可以用来执行常见的transformation。 唯一的参数表示执行迭代的最大次数。

在`IterativeDataSet`上调用`closeWith(DataSet)`来设定迭代的结束， 并会确定哪个transformation会反馈给下一次迭代。
有一种可选方式是通过`closeWith(DataSet, DataSet)`， 当第一个dataset 为空时，它会结束迭代并evaluate 第二个dataset
如果结束条件没有触发， 迭代会执行最大次数后再结束。

下例展示了如何计算pi. 

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

delta 迭代解决了这种场景， 每一次迭代并不是改变数据的每一点。

在每一次迭代中，返回部分方案(称为workset)， delta 迭代维护跨迭代的状态(称为solution set), 这些状态通过deltas来更新。
迭代计算的结果就是最后迭代后的结果。 如果想了解delta 迭代的基本原则，请参考[Introduction to Iterations](iterations.html)。

定义DeltaIteration方式和定义BulkIteration 很类似。 对于delta迭代， 2个data set（workset和solution set）组成了
每一次迭代的输入， 新的workset和新的soution set做为每次迭代的输出。

调用`iterateDelta(DataSet, int, int)` 或 `iterateDelta(DataSet, int, int[])` 来创建一个DeltaIteration。
在最开始的solution set上调用这个函数。 参数是起始data set， 最大迭代次数和key的position。 返回`DeltaIteration`， 
可以通过`iteration.getWorkset()` and `iteration.getSolutionSet()` 来拿到workset和solution set。

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
