---
title:  "Connectors"

# Sub-level navigation
sub-nav-group: batch
sub-nav-pos: 4
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

* TOC
{:toc}

>注：本节未经校验，如有问题欢迎提issue

## Reading from file systems

Flink 内建支持如下的文件系统:

| Filesystem                            | Scheme       | Notes  |
| ------------------------------------- |--------------| ------ |
| Hadoop Distributed File System (HDFS) &nbsp; | `hdfs://`    | All HDFS versions are supported |
| Amazon S3                             | `s3://`      | Support through Hadoop file system implementation (see below) | 
| MapR file system                      | `maprfs://`  | The user has to manually place the required jar files in the `lib/` dir |
| Tachyon                               | `tachyon://` &nbsp; | Support through Hadoop file system implementation (see below) |



### Using Hadoop file system implementations


flink 允许用户使用任何实现了`org.apache.hadoop.fs.FileSystem`接口的文件系统。 

- [S3](https://aws.amazon.com/s3/) (tested)
- [Google Cloud Storage Connector for Hadoop](https://cloud.google.com/hadoop/google-cloud-storage-connector) (tested)
- [Tachyon](http://tachyon-project.org/) (tested)
- [XtreemFS](http://www.xtreemfs.org/) (tested)
- FTP via [Hftp](http://hadoop.apache.org/docs/r1.2.1/hftp.html) (not tested)
- and many more.

如果想要在flink中使用hadoop 文件系统， 请确定：

- `flink-conf.yaml` 设置`fs.hdfs.hadoopconf` 到正确的hadoop 配置文件目录。
- hadoop 配置中必须有要求文件系统的入口点， 可以参考下面的s3和tachyon。
- 对应文件系统需要的class 必须已经安装在flink安装目录的`lib／`中， 并且是每台机器上。 如果不能放到这个目录下， 那必须设置`HADOOP_CLASSPATH`环境变量， flink会把这个目录下的jar加载到classpath中

#### Amazon S3

For Amazon S3 support add the following entries into the `core-site.xml` file:

~~~xml
<!-- configure the file system implementation -->
<property>
  <name>fs.s3.impl</name>
  <value>org.apache.hadoop.fs.s3native.NativeS3FileSystem</value>
</property>

<!-- set your AWS ID -->
<property>
  <name>fs.s3.awsAccessKeyId</name>
  <value>putKeyHere</value>
</property>

<!-- set your AWS access key -->
<property>
  <name>fs.s3.awsSecretAccessKey</name>
  <value>putSecretHere</value>
</property>
~~~

#### Tachyon

For Tachyon support add the following entry into the `core-site.xml` file:

~~~xml
<property>
  <name>fs.tachyon.impl</name>
  <value>tachyon.hadoop.TFS</value>
</property>
~~~


## Connecting to other systems using Input/OutputFormat wrappers for Hadoop

连接其他封装input/outputformat的文件系统。


flink允许用户使用很多不同的系统作为source或sink。 为了让系统很容易扩展， 类似hadoop，flink也有`InputFormat`s 和 `OutputFormat`的概念


`HadoopInputFormat`就是实现了`InputFormat`。 允许用户使用现有hadoop 所有的input format。

[Read more about Hadoop compatibility in Flink]({{ site.baseurl }}/apis/batch/hadoop_compatibility.html) 会给出一些连接其他系统的example。


## Avro support in Flink

flink 内建扩展支持[Apache Avro](http://avro.apache.org/)。 这样很容易从avro文件中读取数据。 同样， 序列化框架可以处理从avro schema产生的class

如果想要读取avro的文件， 用户必须设置`AvroInputFormat`

**Example**:

~~~java
AvroInputFormat<User> users = new AvroInputFormat<User>(in, User.class);
DataSet<User> usersDS = env.createInput(users);
~~~

`User` 是一个由avro生成的pojo。 flink 可以在这些pojo上执行string基础的key选择， 比如：

~~~java
usersDS.groupBy("name")
~~~



`GenericData.Record` 可以在flink中使用，但不推荐。 因为record中包含所有的schema， 会有点慢。


flink的pojo字段选择对avro生成的pojo同样有效。 但仅仅当字段类型被正确写入到生产的类中。 如果一个字段是某种类型object, 用户就不能用这个字段作为join或grouping的key。
在avro中确定一个字段类似`{"name": "type_double_test", "type": "double"},`是ok的， 然后确定一个字段为union-type 但只有一个filed(`{"name": "type_double_test", "type": ["double"]},`)， 这样会生成一个类型`Object`字段。 注意， 确定nullable类型(`{"name": "type_double_test", "type": ["null", "double"]},`) 是可以的



### Access Microsoft Azure Table Storage

_Note: This example works starting from Flink 0.6-incubating_

这个例子使用`HadoopInputFormat` 封装来使用一种现存的hadoop input format实现访问[Azure's Table Storage](https://azure.microsoft.com/en-us/documentation/articles/storage-introduction/).

1. 下载并编译`azure-tables-hadoop`工程。 这个项目开发的input format 目前在中央maven库中还不能被直接使用， 因此用户得自己编译项目。

   ~~~bash
   git clone https://github.com/mooso/azure-tables-hadoop.git
   cd azure-tables-hadoop
   mvn clean install
   ~~~

2. 用quickstart来快速创建一个flink项目：

   ~~~bash
   curl https://flink.apache.org/q/quickstart.sh | bash
   ~~~

3. 在`pom.xml`中加入下面的依赖：

   ~~~xml
   <dependency>
       <groupId>org.apache.flink</groupId>
       <artifactId>flink-hadoop-compatibility{{ site.scala_version_suffix }}</artifactId>
       <version>{{site.version}}</version>
   </dependency>
   <dependency>
     <groupId>com.microsoft.hadoop</groupId>
     <artifactId>microsoft-hadoop-azure</artifactId>
     <version>0.0.4</version>
   </dependency>
   ~~~

   `flink-hadoop-compatibility` 是一个提供hadoop input format 封装的package。
   把`microsoft-hadoop-azure`加到项目中去。 

建议把项目import到ide中，比如eclipse或intellij。 查看代码`Job.java`， 它是一个flink job的空框架


Paste the following code into it:

~~~java
import java.util.Map;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.hadoopcompatibility.mapreduce.HadoopInputFormat;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import com.microsoft.hadoop.azure.AzureTableConfiguration;
import com.microsoft.hadoop.azure.AzureTableInputFormat;
import com.microsoft.hadoop.azure.WritableEntity;
import com.microsoft.windowsazure.storage.table.EntityProperty;

public class AzureTableExample {

  public static void main(String[] args) throws Exception {
    // set up the execution environment
    final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
    
    // create a  AzureTableInputFormat, using a Hadoop input format wrapper
    HadoopInputFormat<Text, WritableEntity> hdIf = new HadoopInputFormat<Text, WritableEntity>(new AzureTableInputFormat(), Text.class, WritableEntity.class, new Job());

    // set the Account URI, something like: https://apacheflink.table.core.windows.net
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.ACCOUNT_URI.getKey(), "TODO"); 
    // set the secret storage key here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.STORAGE_KEY.getKey(), "TODO");
    // set the table name here
    hdIf.getConfiguration().set(AzureTableConfiguration.Keys.TABLE_NAME.getKey(), "TODO");
    
    DataSet<Tuple2<Text, WritableEntity>> input = env.createInput(hdIf);
    // a little example how to use the data in a mapper.
    DataSet<String> fin = input.map(new MapFunction<Tuple2<Text,WritableEntity>, String>() {
      @Override
      public String map(Tuple2<Text, WritableEntity> arg0) throws Exception {
        System.err.println("--------------------------------\nKey = "+arg0.f0);
        WritableEntity we = arg0.f1;

        for(Map.Entry<String, EntityProperty> prop : we.getProperties().entrySet()) {
          System.err.println("key="+prop.getKey() + " ; value (asString)="+prop.getValue().getValueAsString());
        }

        return arg0.f0.toString();
      }
    });

    // emit result (this works only locally)
    fin.print();

    // execute program
    env.execute("Azure Example");
  }
}
~~~

这个例子展示如何访问azure的table并把数据加载到`DataSet`

## Access MongoDB

This [GitHub repository documents how to use MongoDB with Apache Flink (starting from 0.7-incubating)](https://github.com/okkam-it/flink-mongodb-test).


