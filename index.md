---
title: "概览"
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

本文档是针对 Apache Flink {{ site.version }} 版本的。

Apache Flink 是一个开源的分布式流处理和批处理系统。Flink 的核心是在数据流上提供了数据分发、通信、具备容错的分布式计算。同时，Flink 在流处理引擎上构建了批处理引擎，原生支持了迭代计算、内存管理和程序优化。

## 第一步

- **快速起步**: 在你的本地机器上 [运行一个实例](quickstart/setup_quickstart.html) 或者 [编写一个实例](quickstart/run_example_quickstart.html)。

- **安装**: [本地]({{ site.baseurl }}/setup/local_setup.html), [集群](setup/cluster_setup.html), 和 [谷歌云](setup/gce_setup.html) 安装指南为你展示了如何部署 Flink。

- **编程指南**: 你可以查阅 [基础概念](apis/common/index.html) 和 [DataStream API 指南](apis/streaming/index.html) 或者 [DataSet API 指南](apis/batch/index.html) 学习如何编写第一个 Flink 程序。

- **迁移指南**: 如果你要从 Flink 0.10.x 升级到 1.0 ，请查阅 [0.10 到 1.0 迁移指南](https://cwiki.apache.org/confluence/display/FLINK/Migration+Guide%3A+0.10.x+to+1.0.x)。

## 栈

这是 Flink 技术栈的一个总览。点击任意一个组件可以查看相应的文档页面。

<center>
  <img src="../fig/stack.png" width="700px" alt="Apache Flink: Stack" usemap="#overview-stack">
</center>

<map name="overview-stack">
<area id="lib-datastream-cep" title="CEP: Complex Event Processing" href="{{ site.baseurl }}/apis/streaming/libs/cep.html" shape="rect" coords="63,0,143,177" />
<area id="lib-datastream-table" title="Table: Relational DataStreams" href="{{ site.baseurl }}/apis/batch/libs/table.html" shape="rect" coords="143,0,223,177" />
<area id="lib-dataset-ml" title="FlinkML: Machine Learning" href="{{ site.baseurl }}/apis/batch/libs/ml/index.html" shape="rect" coords="382,2,462,176" />
<area id="lib-dataset-gelly" title="Gelly: Graph Processing" href="{{ site.baseurl }}/apis/batch/libs/gelly.html" shape="rect" coords="461,0,541,177" />
<area id="lib-dataset-table" title="Table: Relational DataSets" href="{{ site.baseurl }}/apis/batch/libs/table.html" shape="rect" coords="544,0,624,177" />
<area id="datastream" title="DataStream API" href="{{ site.baseurl }}/apis/streaming/index.html" shape="rect" coords="64,177,379,255" />
<area id="dataset" title="DataSet API" href="{{ site.baseurl }}/apis/batch/index.html" shape="rect" coords="382,177,697,255" />
<area id="runtime" title="Runtime" href="{{ site.baseurl }}/internals/general_arch.html" shape="rect" coords="63,257,700,335" />
<area id="local" title="Local" href="{{ site.baseurl }}/setup/local_setup.html" shape="rect" coords="62,337,275,414" />
<area id="cluster" title="Cluster" href="{{ site.baseurl }}/setup/cluster_setup.html" shape="rect" coords="273,336,486,413" />
<area id="cloud" title="Cloud" href="{{ site.baseurl }}/setup/gce_setup.html" shape="rect" coords="485,336,700,414" />
</map>

