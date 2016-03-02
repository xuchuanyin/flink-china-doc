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

本文档是针对 Apache Flink {{ site.version }} 版本的，当前还是开发版本，这也是 Apache Flink 下一个即将到来的主版本。

Apache Flink 是一个开源的分布式流处理和批处理系统。不过 Flink 核心是一个流处理引擎，提供了分布式，高可用， 具备容错性的流处理平台。同时，Flink 在流处理引擎上构建了批处理引擎，原生支持了迭代计算、内存管理和程序优化。

如果你想要开始编写自己的第一个程序，请前往快速起步，然后查阅 [DataSet API 指南](apis/batch/index.html) 或是 [DataStream API 指南](apis/streaming/index.html)。

## 栈

这是 Flink 技术栈的一个总览。点击任意一个组件可以查看相应的文档页面。

<img src="fig/overview-stack-0.9.png" width="893" height="450" alt="Stack" usemap="#overview-stack">

<map name="overview-stack">
  <area shape="rect" coords="188,0,263,200" alt="Graph API: Gelly" href="libs/gelly_guide.html">
  <area shape="rect" coords="268,0,343,200" alt="Flink ML" href="libs/ml/">
  <area shape="rect" coords="348,0,423,200" alt="Table" href="libs/table.html">

  <area shape="rect" coords="188,205,538,260" alt="DataSet API (Java/Scala)" href="apis/batch/index.html">
  <area shape="rect" coords="543,205,893,260" alt="DataStream API (Java/Scala)" href="apis/streaming/index.html">

  <!-- <area shape="rect" coords="188,275,538,330" alt="Optimizer" href="optimizer.html"> -->
  <!-- <area shape="rect" coords="543,275,893,330" alt="Stream Builder" href="streambuilder.html"> -->

  <area shape="rect" coords="188,335,893,385" alt="Flink Runtime" href="internals/general_arch.html">

  <area shape="rect" coords="188,405,328,455" alt="Local" href="apis/local_execution.html">
  <area shape="rect" coords="333,405,473,455" alt="Remote" href="apis/cluster_execution.html">
  <area shape="rect" coords="478,405,638,455" alt="Embedded" href="apis/local_execution.html">
  <area shape="rect" coords="643,405,765,455" alt="YARN" href="setup/yarn_setup.html">
</map>
