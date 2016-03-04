---
title: "快速起步: 安装"
# Top navigation
top-nav-group: quickstart
top-nav-pos: 1
top-nav-title: 安装
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

只需要简单几步就能获取 Flink 并运行。

## 环境需求

Flink 可以运行在 **Linux**，**Mac OS X** 和 **Windows** 上，仅要求 **Java 7.x** 或更高版本。 对于 Window 用户，请参考 [Flink on Windows]({{ site.baseurl }}/setup/local_setup.html#flink-on-windows)。

## 下载

下载所需的二进制包。如果你想要Flink和Hadoop一起用（如HDFS或者HBase），请**选择一个与你的Hadoop版本相匹配的Flink包**。当你不确定或者只是想运行在本地文件系统上，请选择Hadoop 1.2.x对应的包。

<ul class="nav nav-tabs">
  <li class="active"><a href="#bin-hadoop1" data-toggle="tab">Hadoop 1.2</a></li>
  <li><a href="#bin-hadoop2" data-toggle="tab">Hadoop 2 (YARN)</a></li>
</ul>
<p>
<div class="tab-content text-center">
  <div class="tab-pane active" id="bin-hadoop1">
    <a class="btn btn-info btn-lg" onclick="_gaq.push(['_trackEvent','Action','download-quickstart-setup-1',this.href]);" href="{{site.FLINK_DOWNLOAD_URL_HADOOP1_STABLE}}"><i class="icon-download"> </i> Download Flink for Hadoop 1.2</a>
  </div>
  <div class="tab-pane" id="bin-hadoop2">
    <a class="btn btn-info btn-lg" onclick="_gaq.push(['_trackEvent','Action','download-quickstart-setup-2',this.href]);" href="{{site.FLINK_DOWNLOAD_URL_HADOOP2_STABLE}}"><i class="icon-download"> </i> Download Flink for Hadoop 2</a>
  </div>
</div>
</p>

## 启动

1. 进入下载的目录。
2. 解压下载的压缩包。
3. 运行 Flink 。


```bash
$ cd ~/Downloads        # 进入下载的目录
$ tar xzf flink-*.tgz   # 解压下载的压缩包
$ cd flink-0.10.2
$ bin/start-local.sh    # 运行 Flink
```

打 UI [http://localhost:8081](http://localhost:8081) 检查 Jobmanager 和其他组件.
如果想打开流优化模式，可以调用 `bin/start-local-streaming.sh` 来替换 `bin/start-local.sh`。

## 运行例子

运行 **Word Count** 例子，了解 Flink 运行机制。

- **下载测试数据集**：
  
  ```bash
  $ wget -O hamlet.txt http://www.gutenberg.org/cache/epub/1787/pg1787.txt
  ```
- 在你当前目录中会有一个名叫 *hamlet.txt* 的文本文件。
- **运行样例程序**：
  
  ~~~bash
  $ bin/flink run ./examples/batch/WordCount.jar --input file://`pwd`/hamlet.txt --output file://`pwd`/wordcount-result.txt
  ~~~

- 在当前目录下你会找到一个名叫 *wordcount-result.txt* 的文件。

如果想要运行复杂例子，可以参考 [Kmeans Example]({{ site.baseurl }}/quickstart/run_example_quickstart.html)。

## 停止

当你要停止 Flink 时，只需要运行：

~~~bash
$ bin/stop-local.sh
~~~

## 集群安装

**在集群上运行 Flink** 是和在本地运行一样简单的。需要先配置好 [SSH 免密码登录]({{ site.baseurl }}/setup/cluster_setup.html#passwordless_ssh) 和保证所有节点的**目录结构是一致的**，这是保证我们的脚本能正确控制任务启停的关键。

1. 在每台节点上，复制解压出来的 **flink** 目录到同样的路径下。
2. 选择一个 **master 节点** (JobManager) 然后在 `conf/flink-conf.yaml` 中设置 `jobmanager.rpc.address` 配置项为该节点的 IP 或者主机名。确保所有节点有有一样的 `jobmanager.rpc.address` 配置。
3. 将所有的 **worker 节点** （TaskManager）的 IP 或者主机名（一行一个）填入 `conf/slaves` 文件中。

现在，你可以在 master 节点上**启动集群**：`bin/start-cluster.sh` 。

下面的**例子**阐述了三个节点的集群部署（IP地址从 _10.0.0.1_ 到 _10.0.0.3_，主机名分别为 _master_, _worker1_, _worker2_）。并且展示了配置文件，以及所有机器上一致的可访问的安装路径。


<div class="row">
  <div class="col-md-6 text-center">
    <img src="{{ site.baseurl }}/page/img/quickstart_cluster.png" style="width: 85%">
  </div>
<div class="col-md-6">
  <div class="row">
    <p class="lead text-center">
      /path/to/<strong>flink/conf/<br>flink-conf.yaml</strong>
    <pre>jobmanager.rpc.address: 10.0.0.1</pre>
    </p>
  </div>
<div class="row" style="margin-top: 1em;">
  <p class="lead text-center">
    /path/to/<strong>flink/<br>conf/slaves</strong>
  <pre>
10.0.0.2
10.0.0.3</pre>
  </p>
</div>
</div>
</div>


访问文档的[配置章节]({{ site.baseurl }}/setup/config.html)查看更多可用的配置项。为了使 Flink 更高效的运行，还需要设置一些配置项。

尤其是，

- TaskManager 总共能使用的内存大小（`taskmanager.heap.mb`）
- 每一台机器上能使用的 CPU 个数（`taskmanager.numberOfTaskSlots`）
- 集群中的总 CPU 个数（`parallelism.default`）
- 临时目录（`taskmanager.tmp.dirs`）

是非常重要的配置项。

## Flink on YARN

你可以很方便地部署 Flink 在现有的 **YARN 集群**上。

1. 下载  __Flink Hadoop2 包__: [Flink with Hadoop 2]({{site.FLINK_DOWNLOAD_URL_HADOOP2_STABLE}})
2. 确保你的 __HADOOP_HOME__ (或 _YARN_CONF_DIR_ 或 _HADOOP_CONF_DIR_) __环境变量__设置成你的 YARN 和 HDFS 配置。
3. 运行 **YARN 客户端**：`./bin/yarn-session.sh` 。你可以带参数运行客户端 `-n 10 -tm 8192` 表示分配 10 个 TaskManager，每个拥有 8 GB 的内存。

更多详细的操作指南，请查阅编程指南和范例。

