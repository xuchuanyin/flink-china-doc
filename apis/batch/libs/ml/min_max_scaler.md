---
mathjax: include
title: MinMax标准化

# Sub navigation
sub-nav-group: batch
sub-nav-parent: flinkml
sub-nav-title: MinMax标准化
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

## 描述

 MinMax标准化通过对给定数据集进行缩放，使得所有值都落在指定的区间[min,max]内。
 如果用户没有指定区间的最大和最小值，则MinMax标准化将会把输入特征缩放到[0,1]区间内。
 给定输入数据集：$x_1, x_2,... x_n$，其中最小值为：

 $$x_{min} = min({x_1, x_2,..., x_n})$$

 最大值为：

 $$x_{max} = max({x_1, x_2,..., x_n})$$

 则经过缩放的数据集$z_1, z_2,...,z_n$为：

 $$z_{i}= \frac{x_{i} - x_{min}}{x_{max} - x_{min}} \left ( max - min \right ) + min$$

 其中$\textit{min}$和$\textit{max}$为用户指定的最小值和最大值。

## 操作

`MinMaxScaler`是`Transformer`，因此支持训练和转换操作。

### 训练

MinMaxScaler可以在所有`Vector`或`LabeledVector`的子类型上进行训练：

* `fit[T <: Vector]: DataSet[T] => Unit`
* `fit: DataSet[LabeledVector] => Unit`

### 转换

MinMaxScaler将`Vector`或`LabeledVector`的子类型数据集转换到对应的相同类型的数据集：

* `transform[T <: Vector]: DataSet[T] => DataSet[T]`
* `transform: DataSet[LabeledVector] => DataSet[LabeledVector]`

## 参数

MinMax标准化可由下列参数进行控制：

 <table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left" style="width: 20%">参数</th>
      <th class="text-center">说明</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <td><strong>Min</strong></td>
      <td>
        <p>
        目标缩放区间的最小值
        (默认值：<strong>0.0</strong>)
        </p>
      </td>
    </tr>
    <tr>
      <td><strong>Max</strong></td>
      <td>
        <p>
        目标缩放区间的最大值
        (默认值：<strong>1.0</strong>)
        </p>
      </td>
    </tr>
  </tbody>
</table>

## 代码示例

{% highlight scala %}
// Create MinMax scaler transformer
val minMaxscaler = MinMaxScaler()
  .setMin(-1.0)

// Obtain data set to be scaled
val dataSet: DataSet[Vector] = ...

// Learn the minimum and maximum values of the training data
minMaxscaler.fit(dataSet)

// Scale the provided data set to have min=-1.0 and max=1.0
val scaledDS = minMaxscaler.transform(dataSet)
{% endhighlight %}
