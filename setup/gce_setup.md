---
title:  "Google Compute Engine Setup"
top-nav-group: deployment
top-nav-title: Google Compute Engine
top-nav-pos: 4
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


本文介绍了如何在  [Google Compute Engine](https://cloud.google.com/compute/) 集群上基于 Hadoop 1 或者 Hadoop 2 自动部署 Flink 。Google's [bdutil](https://cloud.google.com/hadoop/bdutil) 工具可以启动一个集群并基于 Hadoop 部署 Flink 。根据下列步骤开始吧。

* This will be replaced by the TOC
{:toc}

# 前提条件

## 安装 Google Cloud SDK

请根据下列教程开始使用 [Google Cloud SDK](https://cloud.google.com/sdk/)。 需要特别注意的是使用下列命令确保 Google Cloud 验证成功：

    gcloud auth login

## 安装 bdutil

当前 bdutil 发布版本中并不包含 Flink 扩展。不过，你可以从 [GitHub](https://github.com/GoogleCloudPlatform/bdutil) 获得最新版本 bdutil ，master分支已经支持 Flink :

    git clone https://github.com/GoogleCloudPlatform/bdutil.git

在源码下载完成之后，进入新创建的 `bdutil` 目录，继续下一个步骤。

# 在 Google Compute Engine 之上部署 Flink

## 设置一个 bucket

如果没有的话，需要创建一个 bucket 用于配置 bdutil 和 staging 文件。gsutil 可以创建一个新的 bucket ：

    gsutil mb gs://<bucket_name>

## 适配 bdutil 配置

使用 bdutil 部署 Flink ，在 bdutil_env.sh 中至少需要配置下列参数。

    CONFIGBUCKET="<bucket_name>"
    PROJECT="<compute_engine_project_name>"
    NUM_WORKERS=<number_of_workers>

    # set this to 'n1-standard-2' if you're using the free trial
    GCE_MACHINE_TYPE="<gce_machine_type>"

    # for example: "europe-west1-d"
    GCE_ZONE="<gce_zone>"

## 适配 Flink 配置

bdutil 的 Flink 扩展为你处理好 Flink 配置。你可以在 `extensions/flink/flink_env.sh` 中添加配置参数。如果想进一步了解配置参数，请见 [configuring Flink](config.html) 。在修改配置之后需要使用`bin/stop-cluster` 和 `bin/start-cluster`重启Flink。

## 启动一个 Flink 集群

在 Google Compute Engine 上启动一个 Flink 集群，执行命令：

    ./bdutil -e extensions/flink/flink_env.sh deploy

## 运行 Flink 示例程序

    ./bdutil shell
    cd /home/hadoop/flink-install/bin
    ./flink run ../examples/batch/WordCount.jar gs://dataflow-samples/shakespeare/othello.txt gs://<bucket_name>/output

## 关闭集群

关闭一个 Flink 集群只需执行

    ./bdutil -e extensions/flink/flink_env.sh delete
