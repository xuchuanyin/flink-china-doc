---
title: "Zipping Elements in a DataSet"
# Sub-level navigation
sub-nav-group: batch
sub-nav-parent: dataset_api
sub-nav-pos: 2
sub-nav-title: Zipping Elements
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


在一些特定的算法中， 用户可能需要分配一个唯一的id给data set元素。
本文档展示如何使用{% gh_link /flink-java/src/main/java/org/apache/flink/api/java/utils/DataSetUtils.java "DataSetUtils" %}来达到这个目的

* This will be replaced by the TOC
{:toc}

### Zip with a Dense Index


想要分配连续的label给元素， 将调用`zipWithIndex` 函数。 它接受一个data set， 返回一个带唯一id， 初始值的新的data set

For example, the following code:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(1);
DataSet<String> in = env.fromElements("A", "B", "C", "D", "E", "F");

DataSet<Tuple2<Long, String>> result = DataSetUtils.zipWithIndex(in);

result.writeAsCsv(resultPath, "\n", ",");
env.execute();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
val input: DataSet[String] = env.fromElements("A", "B", "C", "D", "E", "F")

val result: DataSet[(Long, String)] = input.zipWithIndex

result.writeAsCsv(resultPath, "\n", ",")
env.execute()
{% endhighlight %}
</div>

</div>

tuple 字段将是: (0,A), (1,B), (2,C), (3,D), (4,E), (5,F)

[Back to top](#top)

### Zip with an Unique Identifier


在一些情况下， 用户可能不需要分配连续的labels。 `zipWIthUniqueId`将以pipeline形式来执行， 加速分配label的过程。 
这个函数接收一个data set， 返回一个带唯一id， 初始值 tuple的新的data set。

For example, the following code:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(1);
DataSet<String> in = env.fromElements("A", "B", "C", "D", "E", "F");

DataSet<Tuple2<Long, String>> result = DataSetUtils.zipWithUniqueId(in);

result.writeAsCsv(resultPath, "\n", ",");
env.execute();
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
import org.apache.flink.api.scala._

val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
val input: DataSet[String] = env.fromElements("A", "B", "C", "D", "E", "F")

val result: DataSet[(Long, String)] = input.zipWithUniqueId

result.writeAsCsv(resultPath, "\n", ",")
env.execute()
{% endhighlight %}
</div>

</div>

will yield the tuples: (0,A), (2,B), (4,C), (6,D), (8,E), (10,F)

[Back to top](#top)
