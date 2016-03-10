---
title: "Streaming Connectors"

# Sub-level navigation
sub-nav-group: streaming
sub-nav-id: connectors
sub-nav-pos: 6
sub-nav-title: Connectors
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

连接器（connector）提供了对接各种第三方系统的代码。

当前支持以下这些系统：

 * [Apache Kafka](https://kafka.apache.org/) (sink/source)
 * [Elasticsearch](https://elastic.co/) (sink)
 * [Hadoop FileSystem](http://hadoop.apache.org) (sink)
 * [RabbitMQ](http://www.rabbitmq.com/) (sink/source)
 * [Twitter Streaming API](https://dev.twitter.com/docs/streaming-apis) (source)

要使用这些连接器来运行应用程序，一般都需要先安装并启动第三方的组件，例如消息队列服务器。你可以在对应的章节中找到更详细的指引。