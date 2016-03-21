---
title: "DataSet Transformations"

# Sub-level navigation
sub-nav-group: batch
sub-nav-parent: dataset_api
sub-nav-pos: 1
sub-nav-title: Transformations
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


本文档将进一步介绍DataSets的transformation。 如果想要一个基本了解，可以参考[Programming Guide](index.html)。

在数据集中zip元素并赋给元素index时， 可以参考[Zip Elements Guide](zip_elements_guide.html).

* This will be replaced by the TOC
{:toc}

### Map


Map transformation接收用户定义的一个map 函数， 它操作DataSet里面每个元素。
它实现1对1的映射（mapping），也就是， 一个元素会返回一个结果。

下面例子展示将Integer pairs的DataSet转化为Integers的DataSet：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// MapFunction that adds two integer values
public class IntAdder implements MapFunction<Tuple2<Integer, Integer>, Integer> {
  @Override
  public Integer map(Tuple2<Integer, Integer> in) {
    return in.f0 + in.f1;
  }
}

// [...]
DataSet<Tuple2<Integer, Integer>> intPairs = // [...]
DataSet<Integer> intSums = intPairs.map(new IntAdder());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val intPairs: DataSet[(Int, Int)] = // [...]
val intSums = intPairs.map { pair => pair._1 + pair._2 }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 intSums = intPairs.map(lambda x: sum(x))
~~~

</div>
</div>

### FlatMap

FlatMap transformation 接收用户定义的一个flat-map函数，它会操作DataSet的每个元素。
每一个输入的元素，flat-map会返回任意个结果。

The following code transforms a DataSet of text lines into a DataSet of words:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// FlatMapFunction that tokenizes a String by whitespace characters and emits all String tokens.
public class Tokenizer implements FlatMapFunction<String, String> {
  @Override
  public void flatMap(String value, Collector<String> out) {
    for (String token : value.split("\\W")) {
      out.collect(token);
    }
  }
}

// [...]
DataSet<String> textLines = // [...]
DataSet<String> words = textLines.flatMap(new Tokenizer());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val textLines: DataSet[String] = // [...]
val words = textLines.flatMap { _.split(" ") }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 words = lines.flat_map(lambda x,c: [line.split() for line in x])
~~~

</div>
</div>

### MapPartition

MapPartition 在一个单个程序中转化一个并行分区。 map-partition 函数从Iterable获得partition 并产生任意个结果。
每个分区的元素个数依赖于并行度（degree-of-parallelism）和前一次操作。

The following code transforms a DataSet of text lines into a DataSet of counts per partition:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class PartitionCounter implements MapPartitionFunction<String, Long> {

  public void mapPartition(Iterable<String> values, Collector<Long> out) {
    long c = 0;
    for (String s : values) {
      c++;
    }
    out.collect(c);
  }
}

// [...]
DataSet<String> textLines = // [...]
DataSet<Long> counts = textLines.mapPartition(new PartitionCounter());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val textLines: DataSet[String] = // [...]
// Some is required because the return value must be a Collection.
// There is an implicit conversion from Option to a Collection.
val counts = texLines.mapPartition { in => Some(in.size) }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 counts = lines.map_partition(lambda x,c: [sum(1 for _ in x)])
~~~

</div>
</div>

### Filter

Filter transformation接收一个用户定义的filter函数， 它操作DataSet上每一个元素。
它返回函数返回`true`的元素集合。

下面例子过滤掉小于0的Integers：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// FilterFunction that filters out all Integers smaller than zero.
public class NaturalNumberFilter implements FilterFunction<Integer> {
  @Override
  public boolean filter(Integer number) {
    return number >= 0;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> naturalNumbers = intNumbers.filter(new NaturalNumberFilter());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val intNumbers: DataSet[Int] = // [...]
val naturalNumbers = intNumbers.filter { _ > 0 }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 naturalNumbers = intNumbers.filter(lambda x: x > 0)
~~~

</div>
</div>

**注意:** 
系统假设在进行判定时函数并不修改元素。 进行修改会导致错误结果。

### Project (Tuple DataSets only) (Java/Python API Only)

Project transformation就是去掉或移动Tuple的字段。
`project(int...)` 函数将选择index标示的Tuple字段，并在输出Tuple定义他们的顺序。


Projections 不需要用户定义一个函数。

下面展示几种不同的Project transformation方式：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple3<Integer, Double, String>> in = // [...]
// converts Tuple3<Integer, Double, String> into Tuple2<String, Integer>
DataSet<Tuple2<String, Integer>> out = in.project(2,0);
~~~

#### Projection with Type Hint


注意， java的编译器并不能判断`project`操作的返回类型。 因此， 如果用户在`project`操作的结果上再执行其他的操作会出错。

~~~java
DataSet<Tuple5<String,String,String,String,String>> ds = ....
DataSet<Tuple1<String>> ds2 = ds.project(0).distinct(0);
~~~

This problem can be overcome by hinting the return type of `project` operator like this:

~~~java
DataSet<Tuple1<String>> ds2 = ds.<Tuple1<String>>project(0).distinct(0);
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
out = in.project(2,0);
~~~

</div>
</div>

### Transformations on Grouped DataSet


reduce操作可以操作grouped的数据集。 确定用哪些key来做grouping有很多种方式：

- key 表达式
- key选择函数（key-selector function）
- 一个或多个字段位置key(Tuple DataSet only)
- Case Class 字段(Case Classes only)

请查看reduce的exmaple来了解如何选择grouping key。

### Reduce on Grouped DataSet
在grouped的DataSet执行reduce


A Reduce transformation that is applied on a grouped DataSet reduces each group to a single
element using a user-defined reduce function.

Reduce transformation接收一个用户定义的reduce函数， 这个函数操作一个grouped的DataSet， 将每个group 合并到一个单个元素。


对于每组输入的元素， reduce 函数连续combine 元素对（pairs of elements）到一个元素中，知道一组元素全部合并为一个元素。



#### Reduce on DataSet Grouped by KeySelector Function

key-selector函数用于从DataSet中每个元素中解压出key值， 这个解压出来的key值用于 group 这个DataSet.


下面例子展示如何用key-selector函数group 一个pojo的DataSet， 并做reduce操作。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// some ordinary POJO
public class WC {
  public String word;
  public int count;
  // [...]
}

// ReduceFunction that sums Integer attributes of a POJO
public class WordCounter implements ReduceFunction<WC> {
  @Override
  public WC reduce(WC in1, WC in2) {
    return new WC(in1.word, in1.count + in2.count);
  }
}

// [...]
DataSet<WC> words = // [...]
DataSet<WC> wordCounts = words
                         // DataSet grouping on field "word"
                         .groupBy("word")
                         // apply ReduceFunction on grouped DataSet
                         .reduce(new WordCounter());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// some ordinary POJO
class WC(val word: String, val count: Int) {
  def this() {
    this(null, -1)
  }
  // [...]
}

val words: DataSet[WC] = // [...]
val wordCounts = words.groupBy { _.word } reduce {
  (w1, w2) => new WC(w1.word, w1.count + w2.count)
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~
</div>
</div>

#### Reduce on DataSet Grouped by Field Position Keys (Tuple DataSets only)

对 用字段位置做group的DataSet进行reduce操作(Tuple DataSets only)


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple3<String, Integer, Double>> tuples = // [...]
DataSet<Tuple3<String, Integer, Double>> reducedTuples =
                                         tuples
                                         // group DataSet on first and second field of Tuple
                                         .groupBy(0,1)
                                         // apply ReduceFunction on grouped DataSet
                                         .reduce(new MyTupleReducer());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val tuples = DataSet[(String, Int, Double)] = // [...]
// group on the first and second Tuple field
val reducedTuples = tuples.groupBy(0, 1).reduce { ... }
~~~


#### Reduce on DataSet grouped by Case Class Fields


当使用Case Classes时， 用户可以通过食用field的name来挑选grouping的key。

~~~scala
case class MyClass(val a: String, b: Int, c: Double)
val tuples = DataSet[MyClass] = // [...]
// group on the first and second field
val reducedTuples = tuples.groupBy("a", "b").reduce { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 reducedTuples = tuples.group_by(0, 1).reduce( ... )
~~~

</div>
</div>

### GroupReduce on Grouped DataSet
对一个grouped的DataSet进行GroupReduce



GroupReduce transformation 接收一个用户自定义的group－reduce函数， 这个函数对grouped DataSet上每一个group进行操作。 
它和*Reduce*的区别在于， group－reduce函数是一次拿到一个完整的group。
这个函数被一个含有一个group上所有元素的Iterable调用， 并返回任意个数的result elements。


#### GroupReduce on DataSet Grouped by Field Position Keys (Tuple DataSets only)

下面例子展示 如何将一个DataSet中重复的string给过滤掉， 这个DataSet是由integer来grouped。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class DistinctReduce
         implements GroupReduceFunction<Tuple2<Integer, String>, Tuple2<Integer, String>> {

  @Override
  public void reduce(Iterable<Tuple2<Integer, String>> in, Collector<Tuple2<Integer, String>> out) {

    Set<String> uniqStrings = new HashSet<String>();
    Integer key = null;

    // add all strings of the group to the set
    for (Tuple2<Integer, String> t : in) {
      key = t.f0;
      uniqStrings.add(t.f1);
    }

    // emit all unique strings.
    for (String s : uniqStrings) {
      out.collect(new Tuple2<Integer, String>(key, s));
    }
  }
}

// [...]
DataSet<Tuple2<Integer, String>> input = // [...]
DataSet<Tuple2<Integer, String>> output = input
                           .groupBy(0)            // group DataSet by the first tuple field
                           .reduceGroup(new DistinctReduce());  // apply GroupReduceFunction
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String)] = // [...]
val output = input.groupBy(0).reduceGroup {
      (in, out: Collector[(Int, String)]) =>
        in.toSet foreach (out.collect)
    }
~~~

#### GroupReduce on DataSet Grouped by Case Class Fields


类似*Reduce* transformations中Case Class fields 工作方式。


</div>
<div data-lang="python" markdown="1">

~~~python
 class DistinctReduce(GroupReduceFunction):
   def reduce(self, iterator, collector):
     dic = dict()
     for value in iterator:
       dic[value[1]] = 1
     for key in dic.keys():
       collector.collect(key)

 output = data.group_by(0).reduce_group(DistinctReduce())
~~~


</div>
</div>

#### GroupReduce on DataSet Grouped by KeySelector Function

类似*Reduce* transformations中key-selector  工作方式。

#### GroupReduce on sorted groups


group-reduce 函数通过Iterable来访问一个group的所有元素。 可选的是， 这个Iterable可以按照某种顺序在一个group上对所有元素进行排序。 
很多情况下， 这样可以减少group-reduce的复杂度和提供效率。

The following code shows another example how to remove duplicate Strings in a DataSet grouped by an Integer and sorted by String.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// GroupReduceFunction that removes consecutive identical elements
public class DistinctReduce
         implements GroupReduceFunction<Tuple2<Integer, String>, Tuple2<Integer, String>> {

  @Override
  public void reduce(Iterable<Tuple2<Integer, String>> in, Collector<Tuple2<Integer, String>> out) {
    Integer key = null;
    String comp = null;

    for (Tuple2<Integer, String> t : in) {
      key = t.f0;
      String next = t.f1;

      // check if strings are different
      if (com == null || !next.equals(comp)) {
        out.collect(new Tuple2<Integer, String>(key, next));
        comp = next;
      }
    }
  }
}

// [...]
DataSet<Tuple2<Integer, String>> input = // [...]
DataSet<Double> output = input
                         .groupBy(0)                         // group DataSet by first field
                         .sortGroup(1, Order.ASCENDING)      // sort groups on second tuple field
                         .reduceGroup(new DistinctReduce());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String)] = // [...]
val output = input.groupBy(0).sortGroup(1, Order.ASCENDING).reduceGroup {
      (in, out: Collector[(Int, String)]) =>
        var prev: (Int, String) = null
        for (t <- in) {
          if (prev == null || prev != t)
            out.collect(t)
        }
    }

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 class DistinctReduce(GroupReduceFunction):
   def reduce(self, iterator, collector):
     dic = dict()
     for value in iterator:
       dic[value[1]] = 1
     for key in dic.keys():
       collector.collect(key)

 output = data.group_by(0).sort_group(1, Order.ASCENDING).reduce_group(DistinctReduce())
~~~


</div>
</div>


**注意** GroupSort可以在reduce操作前自由调用， 当在一个可以sort的执行策略的基础上进行grouping时。 

#### Combinable GroupReduceFunctions


和reduce 函数相比， 一个group-reduce并不做潜在combine操作。 如果想要在group-reduce 函数的基础上做combine操作，
需要调用`GroupCombineFunction`接口。


**重要**： `GroupCombineFunction`接口的输入和输出类型必须等同于`GroupReduceFunction`函数的输入类型：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// Combinable GroupReduceFunction that computes a sum.
public class MyCombinableGroupReducer implements
  GroupReduceFunction<Tuple2<String, Integer>, String>,
  GroupCombineFunction<Tuple2<String, Integer>, Tuple2<String, Integer>> 
{
  @Override
  public void reduce(Iterable<Tuple2<String, Integer>> in,
                     Collector<String> out) {

    String key = null;
    int sum = 0;

    for (Tuple2<String, Integer> curr : in) {
      key = curr.f0;
      sum += curr.f1;
    }
    // concat key and sum and emit
    out.collect(key + "-" + sum);
  }

  @Override
  public void combine(Iterable<Tuple2<String, Integer>> in,
                      Collector<Tuple2<String, Integer>> out) {
    String key = null;
    int sum = 0;

    for (Tuple2<String, Integer> curr : in) {
      key = curr.f0;
      sum += curr.f1;
    }
    // emit tuple with key and sum
    out.collect(new Tuple2<>(key, sum)); 
  }
}
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala

// Combinable GroupReduceFunction that computes two sums.
class MyCombinableGroupReducer
  extends GroupReduceFunction[(String, Int), String]
  with GroupCombineFunction[(String, Int), (String, Int)]
{
  override def reduce(
    in: java.lang.Iterable[(String, Int)],
    out: Collector[String]): Unit =
  {
    val r: (String, Int) =
      in.asScala.reduce( (a,b) => (a._1, a._2 + b._2) )
    // concat key and sum and emit
    out.collect (r._1 + "-" + r._2)
  }

  override def combine(
    in: java.lang.Iterable[(String, Int)],
    out: Collector[(String, Int)]): Unit =
  {
    val r: (String, Int) =
      in.asScala.reduce( (a,b) => (a._1, a._2 + b._2) )
    // emit tuple with key and sum
    out.collect(r)
  }
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 class GroupReduce(GroupReduceFunction):
   def reduce(self, iterator, collector):
     key, int_sum = iterator.next()
     for value in iterator:
       int_sum += value[1]
     collector.collect(key + "-" + int_sum))

   def combine(self, iterator, collector):
     key, int_sum = iterator.next()
     for value in iterator:
       int_sum += value[1]
     collector.collect((key, int_sum))

data.reduce_group(GroupReduce(), combinable=True)
~~~

</div>
</div>

### GroupCombine on a Grouped DataSet


GroupCombine transformation 是在GroupReduceFunction上进行combine操作的常见方式。
通常情况下，它允许combine 输入类型`I`到一个任意输出类型`O`。然而， GroupReduce的combine操作
仅仅允许输入类型是`I`到输出类型`I`。 这是因为GroupReduceFunction上的reduce操作期望输入`I`


在一些应用中， 在执行一个额外的transformations前，期望combine一个DataSet到一个中间format。
可以通过CombineGroup transformation用一个很小的代价来达到这种目的。


**注意** 在grouped 的DataSet上GroupCombine是在内存中执行， 用一种贪婪的策略， 
这个策略有可能不是一次性处理所有数据，而是分阶段来完成。 它（GroupCombine）同样在单独的partition上执行，
这些单独的partition没有像GroupReduce transformation这样的数据交换操作。 这会导致部分结果。

下面例子展示使用CombineGroup transformation来完成一种可选wordcount的实现。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<String> input = [..] // The words received as input

DataSet<Tuple2<String, Integer>> combinedWords = input
  .groupBy(0); // group identical words
  .combineGroup(new GroupCombineFunction<String, Tuple2<String, Integer>() {

    public void combine(Iterable<String> words, Collector<Tuple2<String, Integer>>) { // combine
        int count = 0;
        for (String word : words) {
            count++;
        }
        out.collect(new Tuple2(word, count));
    }
});

DataSet<Tuple2<String, Integer>> output = combinedWords
  .groupBy(0);                             // group by words again
  .reduceGroup(new GroupReduceFunction() { // group reduce with full data exchange

    public void reduce(Iterable<Tuple2<String, Integer>>, Collector<Tuple2<String, Integer>>) {
        int count = 0;
        for (Tuple2<String, Integer> word : words) {
            count++;
        }
        out.collect(new Tuple2(word, count));
    }
});
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[String] = [..] // The words received as input

val combinedWords: DataSet[(String, Int)] = input
  .groupBy(0)
  .combineGroup {
    (words, out: Collector[(String, Int)]) =>
        var count = 0
        for (word <- words) {
            count++
        }
        out.collect(word, count)
}

val output: DataSet[(String, Int)] = combinedWords
  .groupBy(0)
  .reduceGroup {
    (words, out: Collector[(String, Int)]) =>
        var count = 0
        for ((word, Int) <- words) {
            count++
        }
        out.collect(word, count)
}

~~~

</div>
</div>


上例展示GroupCombine如何在GroupReduce之前combine words. 上例只是展示这种概念。
注意，在combine步骤中改变DataSet的类型通常需要在GroupReduce之前一个额外的Map transformation。

### Aggregate on Grouped Tuple DataSet


下面有些常用的聚合操作。 fink提供一些内置的聚合操作：

- Sum,
- Min, and
- Max.

仅仅只能在Tuple DataSet上进行聚会转换， 并且基于field postition 的key做grouping。

下面例子展示如何在field positition key上做group的DataSet上执行聚合操作：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple3<Integer, String, Double>> input = // [...]
DataSet<Tuple3<Integer, String, Double>> output = input
                                   .groupBy(1)        // group DataSet on second field
                                   .aggregate(SUM, 0) // compute sum of the first field
                                   .and(MIN, 2);      // compute minimum of the third field
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String, Double)] = // [...]
val output = input.groupBy(1).aggregate(SUM, 0).and(MIN, 2)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>


如果想要在一个DataSet同时执行多个聚合操作， 需要在上一个聚合后执行`.and()`函数， 比如`.aggregate(SUM, 0).and(MIN, 2)`表示对field 0进行sum并且在原始DataSet的field 2上执行minimum。
相反， `.aggregate(SUM, 0).aggregate(MIN, 2)`表示一个聚合基于另外一个聚合的结果， 
`.aggregate(SUM, 0).aggregate(MIN, 2)`表示在计算filed 0的sum 后计算field 2的最小值。

**注意** 聚合函数以后会扩展

### Reduce on full DataSet
全数据集进行reduce

Reduce transformation 使用一个用户定义的reduce函数， 这个函数操作DataSet所有元素。
这个reduce函数连续combine 元素对到一个元素，直到只剩下一个元素。

下面代码展示如何sum 一个Integer DataSet：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// ReduceFunction that sums Integers
public class IntSummer implements ReduceFunction<Integer> {
  @Override
  public Integer reduce(Integer num1, Integer num2) {
    return num1 + num2;
  }
}

// [...]
DataSet<Integer> intNumbers = // [...]
DataSet<Integer> sum = intNumbers.reduce(new IntSummer());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val intNumbers = env.fromElements(1,2,3)
val sum = intNumbers.reduce (_ + _)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 intNumbers = env.from_elements(1,2,3)
 sum = intNumbers.reduce(lambda x,y: x + y)
~~~

</div>
</div>

全数据集上的reduce操作意味着最终的reduce操作不能并行来执行。 然而， 一个reduce函数自动combinable，因此一个reduce转换多数情况下不会限制scalability

### GroupReduce on full DataSet
全数据集上GroupReduce

GroupReduce transformation 接收一个用户自定义的group－reduce函数， 这个函数对 DataSet上每一个元素进行操作。 
这个函数group－reduce迭代DataSet上所有的元素， 并返回任意个数的result elements。


The following example shows how to apply a GroupReduce transformation on a full DataSet:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Integer> input = // [...]
// apply a (preferably combinable) GroupReduceFunction to a DataSet
DataSet<Double> output = input.reduceGroup(new MyGroupReducer());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[Int] = // [...]
val output = input.reduceGroup(new MyGroupReducer())
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 output = data.reduce_group(MyGroupReducer())
~~~

</div>
</div>


**注意** 如果group-reduce不是combinable， 在全数据集上的一个GroupReduce不能被并行执行。
因此， 它是一个集中计算操作。 阅读 上面章节"Combinable GroupReduceFunctions"， 了解如何实现一个combinable的group-reduce函数。

### GroupCombine on a full DataSet
在全数据集上执行GroupCombine


在全数据集上执行GroupCombine类似在一个grouped的DataSet上执行GroupCombine。
数据被partition到所有的node上， 然后用一种贪婪的方式combine(仅仅放到内存中的数据马上combine)。



### Aggregate on full Tuple DataSet
在全数据集上聚合

flink内建如下的常用聚合操作：

- Sum,
- Min, and
- Max.

聚合转换只能在Tuple DataSet上使用。

下面代码展示如何在全数据集上执行聚合转换：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, Double>> input = // [...]
DataSet<Tuple2<Integer, Double>> output = input
                                     .aggregate(SUM, 0)    // compute sum of the first field
                                     .and(MIN, 1);    // compute minimum of the second field
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String, Double)] = // [...]
val output = input.aggregate(SUM, 0).and(MIN, 2)

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

**注意** 以后会支持新的聚合函数。

### Distinct


Distinct 转换 distinct source DataSet上所有元素(过滤掉重复的元素)：


<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, Double>> input = // [...]
DataSet<Tuple2<Integer, Double>> output = input.distinct();

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, String, Double)] = // [...]
val output = input.distinct()

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

可以设置如何在一个数据集上distinct元素的方式：
- 一个或多个field 位置keys(仅仅Tuple DataSets)
- 一个key-selector函数
- 一个key表达式

#### Distinct with field position keys

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, Double, String>> input = // [...]
DataSet<Tuple2<Integer, Double, String>> output = input.distinct(0,2);

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[(Int, Double, String)] = // [...]
val output = input.distinct(0,2)

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

#### Distinct with KeySelector function

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
private static class AbsSelector implements KeySelector<Integer, Integer> {
private static final long serialVersionUID = 1L;
	@Override
	public Integer getKey(Integer t) {
    	return Math.abs(t);
	}
}
DataSet<Integer> input = // [...]
DataSet<Integer> output = input.distinct(new AbsSelector());

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input: DataSet[Int] = // [...]
val output = input.distinct {x => Math.abs(x)}

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

#### Distinct with key expression

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// some ordinary POJO
public class CustomType {
  public String aName;
  public int aNumber;
  // [...]
}

DataSet<CustomType> input = // [...]
DataSet<CustomType> output = input.distinct("aName", "aNumber");

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// some ordinary POJO
case class CustomType(aName : String, aNumber : Int) { }

val input: DataSet[CustomType] = // [...]
val output = input.distinct("aName", "aNumber")

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

可以用通配符来表示使用所有的字段：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<CustomType> input = // [...]
DataSet<CustomType> output = input.distinct("*");

~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
// some ordinary POJO
val input: DataSet[CustomType] = // [...]
val output = input.distinct("_")

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

### Join

join 转换join 2个数据集到一个数据集。 2个数据集上的元素在一个或多个key上join起来， 可以通过下面方式来选择用于join的key：

- 一个或多个field 位置keys(仅仅Tuple DataSets)
- 一个key-selector函数
- 一个key表达式
- Case Class 字段

下面展示几种join转换的方式：

#### Default Join (Join into Tuple2)

默认join 转换产生一个新的Tuple带2个字段的数据集。 tuple第一个字段放第一个数据集的的join 元素， tuple第二个字段放置第二个数据集匹配的元素。

下例展示使用field 位置key的方式执行默认join转换：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public static class User { public String name; public int zip; }
public static class Store { public Manager mgr; public int zip; }
DataSet<User> input1 = // [...]
DataSet<Store> input2 = // [...]
// result dataset is typed as Tuple2
DataSet<Tuple2<User, Store>>
            result = input1.join(input2)
                           .where("zip")       // key of the first input (users)
                           .equalTo("zip");    // key of the second input (stores)
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[(Int, String)] = // [...]
val input2: DataSet[(Double, Int)] = // [...]
val result = input1.join(input2).where(0).equalTo(1)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 result = input1.join(input2).where(0).equal_to(1)
~~~

</div>
</div>

#### Join with Join Function
用join函数来进行join

A Join transformation can also call a user-defined join function to process joining tuples.
A join function receives one element of the first input DataSet and one element of the second input DataSet and returns exactly one element.
一个join转换可以使用自定义的join函数来join tuples。
这个join函数接受第一个数据集的一个元素和第二个数据集的元素，然后返回仅仅一个元素。



下例展示 对一个自定义java对象数据集和Tuple数据集上通过key－selector函数来执行join操作，以及如何使用一个用户自定义的join函数：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// some POJO
public class Rating {
  public String name;
  public String category;
  public int points;
}

// Join function that joins a custom POJO with a Tuple
public class PointWeighter
         implements JoinFunction<Rating, Tuple2<String, Double>, Tuple2<String, Double>> {

  @Override
  public Tuple2<String, Double> join(Rating rating, Tuple2<String, Double> weight) {
    // multiply the points and rating and construct a new output tuple
    return new Tuple2<String, Double>(rating.name, rating.points * weight.f1);
  }
}

DataSet<Rating> ratings = // [...]
DataSet<Tuple2<String, Double>> weights = // [...]
DataSet<Tuple2<String, Double>>
            weightedRatings =
            ratings.join(weights)

                   // key of the first input
                   .where("category")

                   // key of the second input
                   .equalTo("f0")

                   // applying the JoinFunction on joining pairs
                   .with(new PointWeighter());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
case class Rating(name: String, category: String, points: Int)

val ratings: DataSet[Ratings] = // [...]
val weights: DataSet[(String, Double)] = // [...]

val weightedRatings = ratings.join(weights).where("category").equalTo(0) {
  (rating, weight) => (rating.name, rating.points * weight._2)
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 class PointWeighter(JoinFunction):
   def join(self, rating, weight):
     return (rating[0], rating[1] * weight[1])
       if value1[3]:

 weightedRatings =
   ratings.join(weights).where(0).equal_to(0). \
   with(new PointWeighter());
~~~

</div>
</div>

#### Join with Flat-Join Function
用flat－join函数进行join

类似Map and FlatMap, FlatJoin 行为类似join，但返回不是一个元素，可以返回0个，1个，或多个元素。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class PointWeighter
         implements FlatJoinFunction<Rating, Tuple2<String, Double>, Tuple2<String, Double>> {
  @Override
  public void join(Rating rating, Tuple2<String, Double> weight,
	  Collector<Tuple2<String, Double>> out) {
	if (weight.f1 > 0.1) {
		out.collect(new Tuple2<String, Double>(rating.name, rating.points * weight.f1));
	}
  }
}

DataSet<Tuple2<String, Double>>
            weightedRatings =
            ratings.join(weights) // [...]
~~~

#### Join with Projection (Java/Python Only)
用投射(projection)进行join（仅仅java／python）， 意味着返回集是经过挑选的字段。

下例将展示使用projection join：

~~~java
DataSet<Tuple3<Integer, Byte, String>> input1 = // [...]
DataSet<Tuple2<Integer, Double>> input2 = // [...]
DataSet<Tuple4<Integer, String, Double, Byte>
            result =
            input1.join(input2)
                  // key definition on first DataSet using a field position key
                  .where(0)
                  // key definition of second DataSet using a field position key
                  .equalTo(0)
                  // select and reorder fields of matching tuples
                  .projectFirst(0,2).projectSecond(1).projectFirst(1);
~~~


`projectFirst(int...)` 和 `projectSecond(int...)` 挑选第一个数据集的字段以及joined第二个数据集的字段， 他们都会被打包入输出tuple。 index的顺序定义了输出tuple中字段的顺序。
The join projection works also for non-Tuple DataSets. In this case, `projectFirst()` or `projectSecond()` must be called without arguments to add a joined element to the output Tuple.
对一个非Tuple的数据集， join projection同样可以工作， 这种情况下， 必须无参数调用`projectFirst()` or `projectSecond()`来增加一个joined 元素到输出tuple里面。 

</div>
<div data-lang="scala" markdown="1">

~~~scala
case class Rating(name: String, category: String, points: Int)

val ratings: DataSet[Ratings] = // [...]
val weights: DataSet[(String, Double)] = // [...]

val weightedRatings = ratings.join(weights).where("category").equalTo(0) {
  (rating, weight, out: Collector[(String, Double)] =>
    if (weight._2 > 0.1) out.collect(left.name, left.points * right._2)
}

~~~

</div>
<div data-lang="python" markdown="1">

#### Join with Projection (Java/Python Only)


用投射(projection)进行join（仅仅java／python）， 意味着返回集是经过挑选的字段。

下例将展示使用projection join：

~~~python
 result = input1.join(input2).where(0).equal_to(0) \
  .project_first(0,2).project_second(1).project_first(1);
~~~


`projectFirst(int...)` 和 `projectSecond(int...)` 挑选第一个数据集的字段以及joined第二个数据集的字段， 他们都会被打包入输出tuple。 index的顺序定义了输出tuple中字段的顺序。
The join projection works also for non-Tuple DataSets. In this case, `projectFirst()` or `projectSecond()` must be called without arguments to add a joined element to the output Tuple.
对一个非Tuple的数据集， join projection同样可以工作， 这种情况下， 必须无参数调用`projectFirst()` or `projectSecond()`来增加一个joined 元素到输出tuple里面。 


</div>
</div>

#### Join with DataSet Size Hint
带数据集大小提示的join

为了帮助优化器选择正确的执行策略， 用户可以提示join的数据集的大小：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>>
            result1 =
            // hint that the second DataSet is very small
            input1.joinWithTiny(input2)
                  .where(0)
                  .equalTo(0);

DataSet<Tuple2<Tuple2<Integer, String>, Tuple2<Integer, String>>>
            result2 =
            // hint that the second DataSet is very large
            input1.joinWithHuge(input2)
                  .where(0)
                  .equalTo(0);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[(Int, String)] = // [...]
val input2: DataSet[(Int, String)] = // [...]

// hint that the second DataSet is very small
val result1 = input1.joinWithTiny(input2).where(0).equalTo(0)

// hint that the second DataSet is very large
val result1 = input1.joinWithHuge(input2).where(0).equalTo(0)

~~~

</div>
<div data-lang="python" markdown="1">

~~~python

 #hint that the second DataSet is very small
 result1 = input1.join_with_tiny(input2).where(0).equal_to(0)

 #hint that the second DataSet is very large
 result1 = input1.join_with_huge(input2).where(0).equal_to(0)

~~~

</div>
</div>

#### Join Algorithm Hints
join算法提示


flink runtime可以用很多种方式进行join， 每种方式在不同的环境下结果不一样。 系统会自动选择一种合理的方式， 
但允许用户手动选择一种策略， 这种情况下， 用户可以选择一种指定的方式进行join。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<SomeType> input1 = // [...]
DataSet<AnotherType> input2 = // [...]

DataSet<Tuple2<SomeType, AnotherType> result =
      input1.join(input2, JoinHint.BROADCAST_HASH_FIRST)
            .where("id").equalTo("key");
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[SomeType] = // [...]
val input2: DataSet[AnotherType] = // [...]

// hint that the second DataSet is very small
val result1 = input1.join(input2, JoinHint.BROADCAST_HASH_FIRST).where("id").equalTo("key")

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

有如下提示：

* `OPTIMIZER_CHOOSES`: 等价于没有提示， 让系统做选择。

* `BROADCAST_HASH_FIRST`: 广播第一个数据集并根据它来创建一张hash table， 这个hash table 会被第二个数据集来调查（probe）。 这种策略适合第一个数据集比较小。

* `BROADCAST_HASH_SECOND`: 广播第二个数据集并根据它来创建一张hash table， 这个hash table 会被第一个数据集来调查（probe）。 这种策略适合第二个数据集比较小。

* `REPARTITION_HASH_FIRST`: 系统分区（shuffles）每一个输入（除非这个输入已经被分区）并根据第一个数据集创建一张hash table。 
  这种策略适合第一个数据集比第二个数据集小，但2个数据集仍然很大。
  *注意*： 这是默认的fallback策略， 当无法评估大小时并且没有预先存在的parition和可以重复使用的sort－order。

* `REPARTITION_HASH_SECOND`: 系统分区（shuffles）每一个输入（除非这个输入已经被分区）并根据第二个数据集创建一张hash table。 
  这种策略适合第一个数据集比第二个数据集大，但2个数据集仍然很大。

* `REPARTITION_SORT_MERGE`: 系统分区（shuffles）每一个输入（除非这个输入已经被分区）并对每个数据集进行排序（除非它已经排序了）
  2个数据集通过merge 排序的数据集， 这种策略非常适合，当一个或2个数据集都已经排好序。 


### OuterJoin



Outerjoin 转换对2个数据集执行一个left／righ／full outer join。 外部join类似普通的inner join， 并创建在key上相等的所有pair。 除此之外， “outer” 边（left／right／在full时的2边）的纪录被保留当没有找到另外一边对应的key时。 匹配的pair(或者一个有元素，另外一个为`null`)输入给`JoinFunction` 函数，
这个函数将元素pair转化为一个元素，对于`FlatJoinFunction`函数， 它则转化pair为任意多个元素。

通过下面的方式来选择key：

- 一个或多个field 位置keys(仅仅Tuple DataSets)
- 一个key-selector函数
- 一个key表达式
- Case Class 字段

**仅仅Java and Scala DataSet API 支持OuterJoins.**


#### OuterJoin with Join Function
带join函数的outerjoin

OuterJoin 转换调用用户自定义的join函数来处理joining的tuples。
这个join的函数接受第一个数据集的一个元素和第二个数据集的一个元素， 然后仅仅返回一个元素。 
依赖outer join的类型（left, right, full）, 2个输入元素之一可以是`null`。


下例对一个自定义java对象的数据集 left outer join一个tuple数据集， 用key－selector函数来选择key， 并展示如何使用用户自定义的join函数：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// some POJO
public class Rating {
  public String name;
  public String category;
  public int points;
}

// Join function that joins a custom POJO with a Tuple
public class PointAssigner
         implements JoinFunction<Tuple2<String, String>, Rating, Tuple2<String, Integer>> {

  @Override
  public Tuple2<String, Integer> join(Tuple2<String, String> movie, Rating rating) {
    // Assigns the rating points to the movie.
    // NOTE: rating might be null
    return new Tuple2<String, Double>(movie.f0, rating == null ? -1 : rating.points;
  }
}

DataSet<Tuple2<String, String>> movies = // [...]
DataSet<Rating> ratings = // [...]
DataSet<Tuple2<String, Integer>>
            moviesWithPoints =
            movies.leftOuterJoin(ratings)

                   // key of the first input
                   .where("f0")

                   // key of the second input
                   .equalTo("name")

                   // applying the JoinFunction on joining pairs
                   .with(new PointAssigner());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
case class Rating(name: String, category: String, points: Int)

val movies: DataSet[(String, String)] = // [...]
val ratings: DataSet[Ratings] = // [...]

val moviesWithPoints = movies.leftOuterJoin(ratings).where(0).equalTo("name") {
  (movie, rating) => (movie._1, if (rating == null) -1 else rating.points)
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

#### OuterJoin with Flat-Join Function

类似Map and FlatMap, 带flat－join函数的outerjoin 类似outjoin转换， 但返回的不是一个元素， 它可以返回0个，1个或多个元素。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class PointAssigner
         implements FlatJoinFunction<Tuple2<String, String>, Rating, Tuple2<String, Integer>> {
  @Override
  public void join(Tuple2<String, String> movie, Rating rating
    Collector<Tuple2<String, Integer>> out) {
  if (rating == null ) {
    out.collect(new Tuple2<String, Integer>(movie.f0, -1));
  } else if (rating.points < 10) {
    out.collect(new Tuple2<String, Integer>(movie.f0, rating.points));
  } else {
    // do not emit
  }
}

DataSet<Tuple2<String, Integer>>
            moviesWithPoints =
            movies.leftOuterJoin(ratings) // [...]
~~~

#### Join Algorithm Hints


flink runtime可以用很多种方式进行join， 每种方式在不同的情况下结果不一样。 系统会自动选择一种合理的方式， 
但允许用户手动选择一种策略， 这种情况下， 用户可以选择一种指定的方式进行join。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<SomeType> input1 = // [...]
DataSet<AnotherType> input2 = // [...]

DataSet<Tuple2<SomeType, AnotherType> result1 =
      input1.leftOuterJoin(input2, JoinHint.REPARTITION_SORT_MERGE)
            .where("id").equalTo("key");

DataSet<Tuple2<SomeType, AnotherType> result2 =
      input1.rightOuterJoin(input2, JoinHint.BROADCAST_HASH_FIRST)
            .where("id").equalTo("key");
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[SomeType] = // [...]
val input2: DataSet[AnotherType] = // [...]

// hint that the second DataSet is very small
val result1 = input1.leftOuterJoin(input2, JoinHint.REPARTITION_SORT_MERGE).where("id").equalTo("key")

val result2 = input1.rightOuterJoin(input2, JoinHint.BROADCAST_HASH_FIRST).where("id").equalTo("key")

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

The following hints are available.


* `OPTIMIZER_CHOOSES`: 等价于没有提示， 让系统做选择。

* `BROADCAST_HASH_FIRST`: 广播第一个数据集并根据它来创建一张hash table， 这个hash table 会被第二个数据集来调查（probe）。 这种策略适合第一个数据集比较小。

* `BROADCAST_HASH_SECOND`: 广播第二个数据集并根据它来创建一张hash table， 这个hash table 会被第一个数据集来调查（probe）。 这种策略适合第二个数据集比较小。

* `REPARTITION_HASH_FIRST`: 系统分区（shuffles）每一个输入（除非这个输入已经被分区）并根据第一个数据集创建一张hash table。 
  这种策略适合第一个数据集比第二个数据集小，但2个数据集仍然很大。

* `REPARTITION_HASH_SECOND`: 系统分区（shuffles）每一个输入（除非这个输入已经被分区）并根据第二个数据集创建一张hash table。 
  这种策略适合第一个数据集比第二个数据集大，但2个数据集仍然很大。

* `REPARTITION_SORT_MERGE`: 系统分区（shuffles）每一个输入（除非这个输入已经被分区）并对每个数据集进行排序（除非它已经排序了）
  2个数据集通过merge 排序的数据集， 这种策略非常适合，当一个或2个数据集都已经排好序。 

** 注意 ** : 每一种join类型并不是支持所有的执行策略：

* `LeftOuterJoin` supports:
  * `OPTIMIZER_CHOOSES`
  * `BROADCAST_HASH_SECOND`
  * `REPARTITION_HASH_SECOND`
  * `REPARTITION_SORT_MERGE`

* `RightOuterJoin` supports:
  * `OPTIMIZER_CHOOSES`
  * `BROADCAST_HASH_FIRST`
  * `REPARTITION_HASH_FIRST`
  * `REPARTITION_SORT_MERGE`

* `FullOuterJoin` supports:
  * `OPTIMIZER_CHOOSES`
  * `REPARTITION_SORT_MERGE`


### Cross

交叉转换（Croass）合并2个数据集到一个数据集。 它建立所有的成对的2个数据集元素的combination， 实际上就是创建一个笛卡尔集。
cross转换或者调用一个用户自定义的cross函数在每一对元素上或者输出Tuple2。

**注意：** cross转换是一个*非常*耗cpu的操作， 它会冲击一个即使很大的集群。

#### Cross with User-Defined Function
调用用户自定义函数的cross转换。


cross转换可以调用一个用户自定义的cross函数， 这个cross函数接受2个元素， 其中一个元素来自第一个数据集，第二个元素来自第二个数据集，并返回一个结果元素。

下例展示调用用户自定义函数的cross转换：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
public class Coord {
  public int id;
  public int x;
  public int y;
}

// CrossFunction computes the Euclidean distance between two Coord objects.
public class EuclideanDistComputer
         implements CrossFunction<Coord, Coord, Tuple3<Integer, Integer, Double>> {

  @Override
  public Tuple3<Integer, Integer, Double> cross(Coord c1, Coord c2) {
    // compute Euclidean distance of coordinates
    double dist = sqrt(pow(c1.x - c2.x, 2) + pow(c1.y - c2.y, 2));
    return new Tuple3<Integer, Integer, Double>(c1.id, c2.id, dist);
  }
}

DataSet<Coord> coords1 = // [...]
DataSet<Coord> coords2 = // [...]
DataSet<Tuple3<Integer, Integer, Double>>
            distances =
            coords1.cross(coords2)
                   // apply CrossFunction
                   .with(new EuclideanDistComputer());
~~~

#### Cross with Projection

cross转换可以通过投射（projection）来构建结果tuple， 如下所示：

~~~java
DataSet<Tuple3<Integer, Byte, String>> input1 = // [...]
DataSet<Tuple2<Integer, Double>> input2 = // [...]
DataSet<Tuple4<Integer, Byte, Integer, Double>
            result =
            input1.cross(input2)
                  // select and reorder fields of matching tuples
                  .projectSecond(0).projectFirst(1,0).projectSecond(1);
~~~

在一个cross 投射（projection）里面字段选择方式和join projection 是一样的。

</div>
<div data-lang="scala" markdown="1">

~~~scala
case class Coord(id: Int, x: Int, y: Int)

val coords1: DataSet[Coord] = // [...]
val coords2: DataSet[Coord] = // [...]

val distances = coords1.cross(coords2) {
  (c1, c2) =>
    val dist = sqrt(pow(c1.x - c2.x, 2) + pow(c1.y - c2.y, 2))
    (c1.id, c2.id, dist)
}
~~~


</div>
<div data-lang="python" markdown="1">

~~~python
 class Euclid(CrossFunction):
   def cross(self, c1, c2):
     return (c1[0], c2[0], sqrt(pow(c1[1] - c2.[1], 2) + pow(c1[2] - c2[2], 2)))

 distances = coords1.cross(coords2).using(Euclid())
~~~

#### Cross with Projection

cross转换可以通过投射（projection）来构建结果tuple， 如下所示：

~~~python
result = input1.cross(input2).projectFirst(1,0).projectSecond(0,1);
~~~

在一个cross 投射（projection）里面字段选择方式和join projection 是一样的。

</div>
</div>

#### Cross with DataSet Size Hint

为了帮助优化器选择正确的执行策略， 可以提示数据集的大小给cross函数， 如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<Integer, String>> input1 = // [...]
DataSet<Tuple2<Integer, String>> input2 = // [...]

DataSet<Tuple4<Integer, String, Integer, String>>
            udfResult =
                  // hint that the second DataSet is very small
            input1.crossWithTiny(input2)
                  // apply any Cross function (or projection)
                  .with(new MyCrosser());

DataSet<Tuple3<Integer, Integer, String>>
            projectResult =
                  // hint that the second DataSet is very large
            input1.crossWithHuge(input2)
                  // apply a projection (or any Cross function)
                  .projectFirst(0,1).projectSecond(1);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val input1: DataSet[(Int, String)] = // [...]
val input2: DataSet[(Int, String)] = // [...]

// hint that the second DataSet is very small
val result1 = input1.crossWithTiny(input2)

// hint that the second DataSet is very large
val result1 = input1.crossWithHuge(input2)

~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 #hint that the second DataSet is very small
 result1 = input1.cross_with_tiny(input2)

 #hint that the second DataSet is very large
 result1 = input1.cross_with_huge(input2)

~~~

</div>
</div>

### CoGroup

CoGroup转换共同处理2个数据集的groups。 2个数据集都是在一个定义好的key上进行分组（grouped）。 2个数据集上共享相同key的分组会输入个用户自定义的co-group函数。
如果一个指定的key仅仅在一个数据集上有group， 则输入这个group和一个空group给co－group函数。
co－group函数可以独立在各自group上进行迭代，并返回一个任意大小的result 元素。

类似Reduce, GroupReduce, and Join， 可以使用不同的key-selection函数来选择key。


#### CoGroup on DataSets

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

下例展示通过filed position（仅仅tuple数据集）来做group。 用户可以同哟pojo类型和key表达式做同样事情。

~~~java
// Some CoGroupFunction definition
class MyCoGrouper
         implements CoGroupFunction<Tuple2<String, Integer>, Tuple2<String, Double>, Double> {

  @Override
  public void coGroup(Iterable<Tuple2<String, Integer>> iVals,
                      Iterable<Tuple2<String, Double>> dVals,
                      Collector<Double> out) {

    Set<Integer> ints = new HashSet<Integer>();

    // add all Integer values in group to set
    for (Tuple2<String, Integer>> val : iVals) {
      ints.add(val.f1);
    }

    // multiply each Double value with each unique Integer values of group
    for (Tuple2<String, Double> val : dVals) {
      for (Integer i : ints) {
        out.collect(val.f1 * i);
      }
    }
  }
}

// [...]
DataSet<Tuple2<String, Integer>> iVals = // [...]
DataSet<Tuple2<String, Double>> dVals = // [...]
DataSet<Double> output = iVals.coGroup(dVals)
                         // group first DataSet on first tuple field
                         .where(0)
                         // group second DataSet on first tuple field
                         .equalTo(0)
                         // apply CoGroup function on each pair of groups
                         .with(new MyCoGrouper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val iVals: DataSet[(String, Int)] = // [...]
val dVals: DataSet[(String, Double)] = // [...]

val output = iVals.coGroup(dVals).where(0).equalTo(0) {
  (iVals, dVals, out: Collector[Double]) =>
    val ints = iVals map { _._2 } toSet

    for (dVal <- dVals) {
      for (i <- ints) {
        out.collect(dVal._2 * i)
      }
    }
}
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 class CoGroup(CoGroupFunction):
   def co_group(self, ivals, dvals, collector):
     ints = dict()
     # add all Integer values in group to set
     for value in ivals:
       ints[value[1]] = 1
     # multiply each Double value with each unique Integer values of group
     for value in dvals:
       for i in ints.keys():
         collector.collect(value[1] * i)


 output = ivals.co_group(dvals).where(0).equal_to(0).using(CoGroup())
~~~

</div>
</div>


### Union


执行2个数据集上的union， 这2个数据集必须类型相同。 超过2个的数据集union可以调用union函数多次， 如下所示：

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> vals1 = // [...]
DataSet<Tuple2<String, Integer>> vals2 = // [...]
DataSet<Tuple2<String, Integer>> vals3 = // [...]
DataSet<Tuple2<String, Integer>> unioned = vals1.union(vals2).union(vals3);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val vals1: DataSet[(String, Int)] = // [...]
val vals2: DataSet[(String, Int)] = // [...]
val vals3: DataSet[(String, Int)] = // [...]

val unioned = vals1.union(vals2).union(vals3)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
 unioned = vals1.union(vals2).union(vals3)
~~~

</div>
</div>

### Rebalance
均匀rebalance一个多分区的数据集科研用来解决数据倾斜。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<String> in = // [...]
// rebalance DataSet and apply a Map transformation.
DataSet<Tuple2<String, String>> out = in.rebalance()
                                        .map(new Mapper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[String] = // [...]
// rebalance DataSet and apply a Map transformation.
val out = in.rebalance().map { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>


### Hash-Partition


在一个给定key上对一个数据集进行hash-partition.
可以用position key， key表达式， key－selector函数来选择key（查看[Reduce examples](#reduce-on-grouped-dataset)）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> in = // [...]
// hash-partition DataSet by String value and apply a MapPartition transformation.
DataSet<Tuple2<String, String>> out = in.partitionByHash(0)
                                        .mapPartition(new PartitionMapper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[(String, Int)] = // [...]
// hash-partition DataSet by String value and apply a MapPartition transformation.
val out = in.partitionByHash(0).mapPartition { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>

### Range-Partition


在一个给定key上对一个数据集进行range-partition.
可以用position key， key表达式， key－selector函数来选择key（查看[Reduce examples](#reduce-on-grouped-dataset)）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> in = // [...]
// range-partition DataSet by String value and apply a MapPartition transformation.
DataSet<Tuple2<String, String>> out = in.partitionByRange(0)
                                        .mapPartition(new PartitionMapper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[(String, Int)] = // [...]
// range-partition DataSet by String value and apply a MapPartition transformation.
val out = in.partitionByRange(0).mapPartition { ... }
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>


### Sort Partition

基于一个指定的字段按照指定的顺序本地sort一个数据集的所有分区。
可以用position key， key表达式， key－selector函数来选择key（查看[Reduce examples](#reduce-on-grouped-dataset)）。
可以通过链接`sortPartition()`来基于多个字段进行parition 排序。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> in = // [...]
// Locally sort partitions in ascending order on the second String field and
// in descending order on the first String field.
// Apply a MapPartition transformation on the sorted partitions.
DataSet<Tuple2<String, String>> out = in.sortPartition(1, Order.ASCENDING)
                                          .sortPartition(0, Order.DESCENDING)
                                        .mapPartition(new PartitionMapper());
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[(String, Int)] = // [...]
// Locally sort partitions in ascending order on the second String field and
// in descending order on the first String field.
// Apply a MapPartition transformation on the sorted partitions.
val out = in.sortPartition(1, Order.ASCENDING)
              .sortPartition(0, Order.DESCENDING)
            .mapPartition { ... }
~~~

</div>
</div>

### First-n


返回一个数据集的签名n 个元素。 first－n可以对一个常见数据集或grouped数据集或grouped 排序的数据集操作。 
可以用position key， key表达式， key－selector函数来选择key（查看[Reduce examples](#reduce-on-grouped-dataset)）。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
DataSet<Tuple2<String, Integer>> in = // [...]
// Return the first five (arbitrary) elements of the DataSet
DataSet<Tuple2<String, Integer>> out1 = in.first(5);

// Return the first two (arbitrary) elements of each String group
DataSet<Tuple2<String, Integer>> out2 = in.groupBy(0)
                                          .first(2);

// Return the first three elements of each String group ordered by the Integer field
DataSet<Tuple2<String, Integer>> out3 = in.groupBy(0)
                                          .sortGroup(1, Order.ASCENDING)
                                          .first(3);
~~~

</div>
<div data-lang="scala" markdown="1">

~~~scala
val in: DataSet[(String, Int)] = // [...]
// Return the first five (arbitrary) elements of the DataSet
val out1 = in.first(5)

// Return the first two (arbitrary) elements of each String group
val out2 = in.groupBy(0).first(2)

// Return the first three elements of each String group ordered by the Integer field
val out3 = in.groupBy(0).sortGroup(1, Order.ASCENDING).first(3)
~~~

</div>
<div data-lang="python" markdown="1">

~~~python
Not supported.
~~~

</div>
</div>
