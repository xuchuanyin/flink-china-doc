---
title:  "Bundled Examples"

# Sub-level navigation
sub-nav-group: batch
sub-nav-pos: 5
sub-nav-title: Examples
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

>注：本节未经校验，如有问题欢迎提issue

本页介绍的example展示不同的flink应用， 从简单的wordcount到图算法。 展示了如何使用[Flink's API](index.html)。


完整代码可以查看源码下面__flink-examples-batch__ or __flink-examples-streaming__模块。

* This will be replaced by the TOC
{:toc}


## Running an example

在运行example前， 得安装一套flink环境， 参考导航栏"Setup"了解如何启动flink。

最简单的方式是运行`./bin/start-local.sh`， 它将启动一个本地JobManager.

每一个发行版flink都会自带example 目录， 含有本页列出的所有example。

To run the WordCount example, issue the following command:

~~~bash
./bin/flink run ./examples/batch/WordCount.jar
~~~

The other examples can be started in a similar way.

注意， 有些example 运行时不需要输入参数，它们使用自带数据。想基于真实数据运行wordcount， 得传入数据的路径：

~~~bash
./bin/flink run ./examples/batch/WordCount.jar --input /path/to/some/text/data --output /path/to/result
~~~

注意：非本地文件系统会要求一个schema头，比如`hdfs://`.

## Word Count
wordcount是大数据领域的"hello world". 它计算文本中word出现的频率。 这个算法分为2步： 文本被切分为独立的word， 第二步， 单词被grouped并count。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

DataSet<String> text = env.readTextFile("/path/to/file");

DataSet<Tuple2<String, Integer>> counts =
        // split up the lines in pairs (2-tuples) containing: (word,1)
        text.flatMap(new Tokenizer())
        // group by the tuple field "0" and sum up tuple field "1"
        .groupBy(0)
        .sum(1);

counts.writeAsCsv(outputPath, "\n", " ");

// User-defined functions
public static class Tokenizer implements FlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
        // normalize and split the line
        String[] tokens = value.toLowerCase().split("\\W+");

        // emit the pairs
        for (String token : tokens) {
            if (token.length() > 0) {
                out.collect(new Tuple2<String, Integer>(token, 1));
            }   
        }
    }
}
~~~

The {% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/wordcount/WordCount.java  "WordCount example" %} 实现了上面的算法，但可以输入参数 `--input <path> --output <path>`. 任何text文本都可以做测试.

</div>
<div data-lang="scala" markdown="1">

~~~scala
val env = ExecutionEnvironment.getExecutionEnvironment

// get input data
val text = env.readTextFile("/path/to/file")

val counts = text.flatMap { _.toLowerCase.split("\\W+") filter { _.nonEmpty } }
  .map { (_, 1) }
  .groupBy(0)
  .sum(1)

counts.writeAsCsv(outputPath, "\n", " ")
~~~

The {% gh_link /flink-examples/flink-exampls-batch/src/main/scala/org/apache/flink/examples/scala/wordcount/WordCount.scala  "WordCount example" %} 实现了上面的算法，但可以输入参数 `--input <path> --output <path>`. 任何text文本都可以做测试.


</div>
</div>

## Page Rank



pagerank 算法 计算图中页面的“重要性”，通过一个页面指向另一个页面的link。它是一个迭代图算法， 它意味着重复相同的计算。 在每一次迭代中， 每个页面发布它的当前rank 给它所有邻居， 并计算它的新rank，通过对它接收到邻居rank 求和。 google让pagerank算法流行起来， 它用页面的重要性对查询结果进行排序。  

In this simple example, PageRank is implemented with a [bulk iteration](iterations.html) and a fixed number of iterations.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

// read the pages and initial ranks by parsing a CSV file
DataSet<Tuple2<Long, Double>> pagesWithRanks = env.readCsvFile(pagesInputPath)
						   .types(Long.class, Double.class)

// the links are encoded as an adjacency list: (page-id, Array(neighbor-ids))
DataSet<Tuple2<Long, Long[]>> pageLinkLists = getLinksDataSet(env);

// set iterative data set
IterativeDataSet<Tuple2<Long, Double>> iteration = pagesWithRanks.iterate(maxIterations);

DataSet<Tuple2<Long, Double>> newRanks = iteration
        // join pages with outgoing edges and distribute rank
        .join(pageLinkLists).where(0).equalTo(0).flatMap(new JoinVertexWithEdgesMatch())
        // collect and sum ranks
        .groupBy(0).sum(1)
        // apply dampening factor
        .map(new Dampener(DAMPENING_FACTOR, numPages));

DataSet<Tuple2<Long, Double>> finalPageRanks = iteration.closeWith(
        newRanks,
        newRanks.join(iteration).where(0).equalTo(0)
        // termination condition
        .filter(new EpsilonFilter()));

finalPageRanks.writeAsCsv(outputPath, "\n", " ");

// User-defined functions

public static final class JoinVertexWithEdgesMatch
                    implements FlatJoinFunction<Tuple2<Long, Double>, Tuple2<Long, Long[]>,
                                            Tuple2<Long, Double>> {

    @Override
    public void join(<Tuple2<Long, Double> page, Tuple2<Long, Long[]> adj,
                        Collector<Tuple2<Long, Double>> out) {
        Long[] neigbors = adj.f1;
        double rank = page.f1;
        double rankToDistribute = rank / ((double) neigbors.length);

        for (int i = 0; i < neigbors.length; i++) {
            out.collect(new Tuple2<Long, Double>(neighbors[i], rankToDistribute));
        }
    }
}

public static final class Dampener implements MapFunction<Tuple2<Long,Double>, Tuple2<Long,Double>> {
    private final double dampening, randomJump;

    public Dampener(double dampening, double numVertices) {
        this.dampening = dampening;
        this.randomJump = (1 - dampening) / numVertices;
    }

    @Override
    public Tuple2<Long, Double> map(Tuple2<Long, Double> value) {
        value.f1 = (value.f1 * dampening) + randomJump;
        return value;
    }
}

public static final class EpsilonFilter
                implements FilterFunction<Tuple2<Tuple2<Long, Double>, Tuple2<Long, Double>>> {

    @Override
    public boolean filter(Tuple2<Tuple2<Long, Double>, Tuple2<Long, Double>> value) {
        return Math.abs(value.f0.f1 - value.f1.f1) > EPSILON;
    }
}
~~~

The {% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/graph/PageRank.java "PageRank program" %} 实现了上面的算法.
但它要求输入一些参数： `--pages <path> --links <path> --output <path> --numPages <n> --iterations <n>`.

</div>
<div data-lang="scala" markdown="1">

~~~scala
// User-defined types
case class Link(sourceId: Long, targetId: Long)
case class Page(pageId: Long, rank: Double)
case class AdjacencyList(sourceId: Long, targetIds: Array[Long])

// set up execution environment
val env = ExecutionEnvironment.getExecutionEnvironment

// read the pages and initial ranks by parsing a CSV file
val pages = env.readCsvFile[Page](pagesInputPath)

// the links are encoded as an adjacency list: (page-id, Array(neighbor-ids))
val links = env.readCsvFile[Link](linksInputPath)

// assign initial ranks to pages
val pagesWithRanks = pages.map(p => Page(p, 1.0 / numPages))

// build adjacency list from link input
val adjacencyLists = links
  // initialize lists
  .map(e => AdjacencyList(e.sourceId, Array(e.targetId)))
  // concatenate lists
  .groupBy("sourceId").reduce {
  (l1, l2) => AdjacencyList(l1.sourceId, l1.targetIds ++ l2.targetIds)
  }

// start iteration
val finalRanks = pagesWithRanks.iterateWithTermination(maxIterations) {
  currentRanks =>
    val newRanks = currentRanks
      // distribute ranks to target pages
      .join(adjacencyLists).where("pageId").equalTo("sourceId") {
        (page, adjacent, out: Collector[Page]) =>
        for (targetId <- adjacent.targetIds) {
          out.collect(Page(targetId, page.rank / adjacent.targetIds.length))
        }
      }
      // collect ranks and sum them up
      .groupBy("pageId").aggregate(SUM, "rank")
      // apply dampening factor
      .map { p =>
        Page(p.pageId, (p.rank * DAMPENING_FACTOR) + ((1 - DAMPENING_FACTOR) / numPages))
      }

    // terminate if no rank update was significant
    val termination = currentRanks.join(newRanks).where("pageId").equalTo("pageId") {
      (current, next, out: Collector[Int]) =>
        // check for significant update
        if (math.abs(current.rank - next.rank) > EPSILON) out.collect(1)
    }

    (newRanks, termination)
}

val result = finalRanks

// emit result
result.writeAsCsv(outputPath, "\n", " ")
~~~

he {% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/graph/PageRankBasic.scala "PageRank program" %} 实现了上面的例子.
但它要求输入一些参数： `--pages <path> --links <path> --output <path> --numPages <n> --iterations <n>`.
</div>
</div>

输入的文件是plain text 文件，并且按如下格式：
- 页面由long 的id 来表示， id由new-line 来切分。
    * For example `"1\n2\n12\n42\n63\n"` gives five pages with IDs 1, 2, 12, 42, and 63.
- link 由 pageid 对来表示， pageid 中间有空格， link之间由new-line来切分
    * For example `"1 2\n2 12\n1 12\n42 63\n"` gives four (directed) links (1)->(2), (2)->(12),

在这个例子中， 它要求每个页面至少有个incoming的link 和一个outgoing 的link， 页面可以指向页面自己。

## Connected Components

连接组件算法 识别出一张大图中， 哪些点是是连起来的。 连接组件也是一个迭代算法。 在每一次迭代中， 每个点传播它的component id给它的所有邻居。假设邻居传来的componentid 比自己的component id还要小， 它就接收邻居传来的componentid。

这个实现使用[delta iteration](iterations.html)： 没有修改component id的点不回参与下一次迭代。 计算越到后越快， 因为参与计算的点越来越少。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// read vertex and edge data
DataSet<Long> vertices = getVertexDataSet(env);
DataSet<Tuple2<Long, Long>> edges = getEdgeDataSet(env).flatMap(new UndirectEdge());

// assign the initial component IDs (equal to the vertex ID)
DataSet<Tuple2<Long, Long>> verticesWithInitialId = vertices.map(new DuplicateValue<Long>());

// open a delta iteration
DeltaIteration<Tuple2<Long, Long>, Tuple2<Long, Long>> iteration =
        verticesWithInitialId.iterateDelta(verticesWithInitialId, maxIterations, 0);

// apply the step logic:
DataSet<Tuple2<Long, Long>> changes = iteration.getWorkset()
        // join with the edges
        .join(edges).where(0).equalTo(0).with(new NeighborWithComponentIDJoin())
        // select the minimum neighbor component ID
        .groupBy(0).aggregate(Aggregations.MIN, 1)
        // update if the component ID of the candidate is smaller
        .join(iteration.getSolutionSet()).where(0).equalTo(0)
        .flatMap(new ComponentIdFilter());

// close the delta iteration (delta and new workset are identical)
DataSet<Tuple2<Long, Long>> result = iteration.closeWith(changes, changes);

// emit result
result.writeAsCsv(outputPath, "\n", " ");

// User-defined functions

public static final class DuplicateValue<T> implements MapFunction<T, Tuple2<T, T>> {

    @Override
    public Tuple2<T, T> map(T vertex) {
        return new Tuple2<T, T>(vertex, vertex);
    }
}

public static final class UndirectEdge
                    implements FlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {
    Tuple2<Long, Long> invertedEdge = new Tuple2<Long, Long>();

    @Override
    public void flatMap(Tuple2<Long, Long> edge, Collector<Tuple2<Long, Long>> out) {
        invertedEdge.f0 = edge.f1;
        invertedEdge.f1 = edge.f0;
        out.collect(edge);
        out.collect(invertedEdge);
    }
}

public static final class NeighborWithComponentIDJoin
                implements JoinFunction<Tuple2<Long, Long>, Tuple2<Long, Long>, Tuple2<Long, Long>> {

    @Override
    public Tuple2<Long, Long> join(Tuple2<Long, Long> vertexWithComponent, Tuple2<Long, Long> edge) {
        return new Tuple2<Long, Long>(edge.f1, vertexWithComponent.f1);
    }
}

public static final class ComponentIdFilter
                    implements FlatMapFunction<Tuple2<Tuple2<Long, Long>, Tuple2<Long, Long>>,
                                            Tuple2<Long, Long>> {

    @Override
    public void flatMap(Tuple2<Tuple2<Long, Long>, Tuple2<Long, Long>> value,
                        Collector<Tuple2<Long, Long>> out) {
        if (value.f0.f1 < value.f1.f1) {
            out.collect(value.f0);
        }
    }
}
~~~

The {% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/graph/ConnectedComponents.java "ConnectedComponents program" %} implements the above example. It requires the following parameters to run: `--vertices <path> --edges <path> --output <path> --iterations <n>`.

</div>
<div data-lang="scala" markdown="1">

~~~scala
// set up execution environment
val env = ExecutionEnvironment.getExecutionEnvironment

// read vertex and edge data
// assign the initial components (equal to the vertex id)
val vertices = getVerticesDataSet(env).map { id => (id, id) }

// undirected edges by emitting for each input edge the input edges itself and an inverted
// version
val edges = getEdgesDataSet(env).flatMap { edge => Seq(edge, (edge._2, edge._1)) }

// open a delta iteration
val verticesWithComponents = vertices.iterateDelta(vertices, maxIterations, Array(0)) {
  (s, ws) =>

    // apply the step logic: join with the edges
    val allNeighbors = ws.join(edges).where(0).equalTo(0) { (vertex, edge) =>
      (edge._2, vertex._2)
    }

    // select the minimum neighbor
    val minNeighbors = allNeighbors.groupBy(0).min(1)

    // update if the component of the candidate is smaller
    val updatedComponents = minNeighbors.join(s).where(0).equalTo(0) {
      (newVertex, oldVertex, out: Collector[(Long, Long)]) =>
        if (newVertex._2 < oldVertex._2) out.collect(newVertex)
    }

    // delta and new workset are identical
    (updatedComponents, updatedComponents)
}

verticesWithComponents.writeAsCsv(outputPath, "\n", " ")

~~~

The {% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/graph/ConnectedComponents.scala "ConnectedComponents program" %} implements the above example. It requires the following parameters to run: `--vertices <path> --edges <path> --output <path> --iterations <n>`.
</div>
</div>

Input files are plain text files and must be formatted as follows:
- Vertices represented as IDs and separated by new-line characters.
    * For example `"1\n2\n12\n42\n63\n"` gives five vertices with (1), (2), (12), (42), and (63).
- Edges are represented as pairs for vertex IDs which are separated by space characters. Edges are separated by new-line characters:
    * For example `"1 2\n2 12\n1 12\n42 63\n"` gives four (undirected) links (1)-(2), (2)-(12), (1)-(12), and (42)-(63).

## Relational Query


关联查询 假设2张表， 一张是`orders`， 另外一张是`lineitems` 由[TPC-H decision support benchmark](http://www.tpc.org/tpch/)定义的。 tpc-h是数据库领域一个标准benchamark。 下面展示如何生成输入数据。

The example implements the following SQL query.

~~~sql
SELECT l_orderkey, o_shippriority, sum(l_extendedprice) as revenue
    FROM orders, lineitem
WHERE l_orderkey = o_orderkey
    AND o_orderstatus = "F"
    AND YEAR(o_orderdate) > 1993
    AND o_orderpriority LIKE "5%"
GROUP BY l_orderkey, o_shippriority;
~~~

The Flink program, which implements the above query looks as follows.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
// get orders data set: (orderkey, orderstatus, orderdate, orderpriority, shippriority)
DataSet<Tuple5<Integer, String, String, String, Integer>> orders = getOrdersDataSet(env);
// get lineitem data set: (orderkey, extendedprice)
DataSet<Tuple2<Integer, Double>> lineitems = getLineitemDataSet(env);

// orders filtered by year: (orderkey, custkey)
DataSet<Tuple2<Integer, Integer>> ordersFilteredByYear =
        // filter orders
        orders.filter(
            new FilterFunction<Tuple5<Integer, String, String, String, Integer>>() {
                @Override
                public boolean filter(Tuple5<Integer, String, String, String, Integer> t) {
                    // status filter
                    if(!t.f1.equals(STATUS_FILTER)) {
                        return false;
                    // year filter
                    } else if(Integer.parseInt(t.f2.substring(0, 4)) <= YEAR_FILTER) {
                        return false;
                    // order priority filter
                    } else if(!t.f3.startsWith(OPRIO_FILTER)) {
                        return false;
                    }
                    return true;
                }
            })
        // project fields out that are no longer required
        .project(0,4).types(Integer.class, Integer.class);

// join orders with lineitems: (orderkey, shippriority, extendedprice)
DataSet<Tuple3<Integer, Integer, Double>> lineitemsOfOrders =
        ordersFilteredByYear.joinWithHuge(lineitems)
                            .where(0).equalTo(0)
                            .projectFirst(0,1).projectSecond(1)
                            .types(Integer.class, Integer.class, Double.class);

// extendedprice sums: (orderkey, shippriority, sum(extendedprice))
DataSet<Tuple3<Integer, Integer, Double>> priceSums =
        // group by order and sum extendedprice
        lineitemsOfOrders.groupBy(0,1).aggregate(Aggregations.SUM, 2);

// emit result
priceSums.writeAsCsv(outputPath);
~~~

The {% gh_link /flink-examples/flink-examples-batch/src/main/java/org/apache/flink/examples/java/relational/TPCHQuery10.java "Relational Query program" %} implements the above query. It requires the following parameters to run: `--orders <path> --lineitem <path> --output <path>`.

</div>
<div data-lang="scala" markdown="1">
Coming soon...

The {% gh_link /flink-examples/flink-examples-batch/src/main/scala/org/apache/flink/examples/scala/relational/TPCHQuery3.scala "Relational Query program" %} implements the above query. It requires the following parameters to run: `--orders <path> --lineitem <path> --output <path>`.

</div>
</div>

The orders and lineitem files can be generated using the [TPC-H benchmark](http://www.tpc.org/tpch/) suite's data generator tool (DBGEN).
Take the following steps to generate arbitrary large input files for the provided Flink programs:

1.  Download and unpack DBGEN
2.  Make a copy of *makefile.suite* called *Makefile* and perform the following changes:

~~~bash
DATABASE = DB2
MACHINE  = LINUX
WORKLOAD = TPCH
CC       = gcc
~~~

1.  Build DBGEN using *make*
2.  Generate lineitem and orders relations using dbgen. A scale factor
    (-s) of 1 results in a generated data set with about 1 GB size.

~~~bash
./dbgen -T o -s 1
~~~
