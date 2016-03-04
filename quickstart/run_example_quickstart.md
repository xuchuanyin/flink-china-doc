---
title: "快速启动: 运行 K-Means 实例"
# Top navigation
top-nav-group: quickstart
top-nav-pos: 2
top-nav-title: 运行实例
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

* This will be replaced by the TOC
{:toc}
本文主要介绍在flink上运行实例([K-Means clustering](http://en.wikipedia.org/wiki/K-means_clustering))需要操作的一系列步骤。另一方面，你能观察实例运行过程中的可
视化界面、优化策略以及跟踪实例执行进度。


## 安装 Flink
请查看 [快速安装](setup_quickstart.html) 来安装flink，并进入flink安装的根目录。


## 生成输入数据
Flink 为 K-Means 封装了数据生产器

~~~bash
# 假设你已经位于flink的安装根目录
mkdir kmeans
cd kmeans
# 运行数据生成器
java -cp ../examples/batch/KMeans.jar:../lib/flink-dist-{{ site.version }}.jar \
  org.apache.flink.examples.java.clustering.util.KMeansDataGenerator \
  -points 500 -k 10 -stddev 0.08 -output `pwd`
~~~

生成器需要以下参数（位于`[]`的参数是可选项）：

~~~bash
-points <num> -k <num clusters> [-output <output-path>] [-stddev <relative stddev>] [-range <centroid range>] [-seed <seed>]
~~~

-stddev 是一个有趣的调整参数，它决定了随机生成的数据点于中心的接近程度。

`kmeans/` 目录下包含两个文件: `centers` 和 `points`. `points` 包含了集群上所有的数据点， `centers` 包含了集群初始化后的所有中心。


## 检测数据
使用 `plotPoints.py` 工具来检查之前产生的数据。 [Download Python Script](plotPoints.py)

~~~ bash
python plotPoints.py points ./points input
~~~ 

备注: 你可能需要安装 [matplotlib](http://matplotlib.org/) (在Ubuntn系统上是`python-matplotlib`) 来跑Python脚本.

你可以打开 `input-plot.pdf` 来查看输入数据, 比如Evince相关的数据就存放在 (`evince input-plot.pdf`)文件上.

以下是对 -stddev 设置不同值情况下输入数据点分布图的展示。

|relative stddev = 0.03|relative stddev = 0.08|relative stddev = 0.15|
|:--------------------:|:--------------------:|:--------------------:|
|<img src="{{ site.baseurl }}/page/img/quickstart-example/kmeans003.png" alt="example1" style="width: 275px;"/>|<img src="{{ site.baseurl }}/page/img/quickstart-example/kmeans008.png" alt="example2" style="width: 275px;"/>|<img src="{{ site.baseurl }}/page/img/quickstart-example/kmeans015.png" alt="example3" style="width: 275px;"/>|


## 启动 Flink
在你本机上启动 Flink 和 web端任务提交客户端。

~~~ bash
# 返回到 Flink 的根目录
cd ..
# 启动 Flink
./bin/start-local.sh
~~~

## 运行 K-Means 实例
Flink web 界面上允许用户交互去提交Flink任务。

<div class="row" style="padding-top:15px">
	<div class="col-md-6">
		<a data-lightbox="compiler" href="{{ site.baseurl }}/page/img/quickstart-example/jobmanager_kmeans_submit.png" data-lightbox="example-1"><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-example/jobmanager_kmeans_submit.png" /></a>
	</div>
	<div class="col-md-6">
		1. 打开web界面 <a href="http://localhost:8081">localhost:8081</a> <br>
		2. 在菜单上选择 "Submit new Job"  <br>
		3. 通过点击"Add New"， 选择<code>examples/batch</code> 目录下的 <code>KMeans.jar</code>，之后点击 "Upload"。 <br>
		4. 在一列任务中选择 <code>KMeans.jar</code> <br>
		5. 在弹出框中输入参数及配置: <br>
		    维持 <i>Entry Class</i> 和 <i>Parallelism</i> 表格不变<br>
		    为该实例输入以下参数: <br>
		    (KMeans 期望的输入参数是: <code>--points &lt;path&gt; --centroids &lt;path&gt; --output &lt;path&gt; --iterations &lt;n&gt;</code>
			{% highlight bash %}--points /tmp/kmeans/points --centroids /tmp/kmeans/centers --output /tmp/kmeans/result --iterations 10{% endhighlight %}<br>
		6. 点击 <b>Submit</b> 来启动任务
	</div>
</div>
<hr>
<div class="row" style="padding-top:15px">
	<div class="col-md-6">
		<a data-lightbox="compiler" href="{{ site.baseurl }}/page/img/quickstart-example/jobmanager_kmeans_execute.png" data-lightbox="example-1"><img class="img-responsive" src="{{ site.baseurl }}/page/img/quickstart-example/jobmanager_kmeans_execute.png" /></a>
	</div>

	<div class="col-md-6">
		查看任务执行过程。
	</div>
</div>


## 停止 Flink
停止 Flink命令.

~~~ bash
# 停止 Flink
./bin/stop-local.sh
~~~

## 分析结果
可以再次运行 [Python Script](plotPoints.py) 脚本来可视化展示结果.

~~~bash
cd kmeans
python plotPoints.py result ./result clusters
~~~

以下三幅图展示了对之前输入数据点集采样统计的结果。你可以多次调整参数（迭代次数、集群数量）来查看这些参数如何影响数据点的分布。


|relative stddev = 0.03|relative stddev = 0.08|relative stddev = 0.15|
|:--------------------:|:--------------------:|:--------------------:|
|<img src="{{ site.baseurl }}/page/img/quickstart-example/result003.png" alt="example1" style="width: 275px;"/>|<img src="{{ site.baseurl }}/page/img/quickstart-example/result008.png" alt="example2" style="width: 275px;"/>|<img src="{{ site.baseurl }}/page/img/quickstart-example/result015.png" alt="example3" style="width: 275px;"/>|

