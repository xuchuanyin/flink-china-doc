---
mathjax: include
title: 多项式特征转换
# Sub navigation
sub-nav-group: batch
sub-nav-parent: flinkml
sub-nav-title: 多项式特征转换
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

多项式特征转换将一个向量映射到 $d$ 次的多项式特征空间中，其中输入向量的维数决定了多项式的次数，每个多项式特征因子对应了特定维的输入向量。

给定向量：$(x, y, z, \ldots)^T$，经过转换后的特征向量可表示为：

$$\left(x, y, z, x^2, xy, y^2, yz, z^2, x^3, x^2y, x^2z, xy^2, xyz, xz^2, y^3, \ldots\right)^T$$


Flink 的实现会将多项式按降次进行排列。

给定输入向量：$\left(3,2\right)^T$，3次的多项式特征向量可表示为：
 
 $$\left(3^3, 3^2\cdot2, 3\cdot2^2, 2^3, 3^2, 3\cdot2, 2^2, 3, 2\right)^T$$


任何需要输入为`LabeledVector`或`Vector`的子类型的 `Transformer`或`Predictor`实现，都可以前置多项式特征转换。

## 操作

`PolynomialFeatures` 是 `Transformer`，因此支持训练和转换操作。 

### 训练

PolynomialFeatures 并不基于输入数据进行训练，因此支持所有类型的输入。

### Transform

PolynomialFeatures 将 `Vector` 或 `LabeledVector` 转换成对应的 DataSet： 

* `transform[T <: Vector]: DataSet[T] => DataSet[T]`
* `transform: DataSet[LabeledVector] => DataSet[LabeledVector]`

## Parameters

多项式特征转换可由以下参数来控制：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">参数</th>
        <th class="text-center">说明</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>多项式次数</strong></td>
        <td>
          <p>
            多项式的最大次数 
            (默认值：<strong>10</strong>)
          </p>
        </td>
      </tr>
    </tbody>
  </table>

## 示例代码

{% highlight scala %}
// 获取训练数据集
val trainingDS: DataSet[LabeledVector] = ...

// 将多项式次数设置为3
val polyFeatures = PolynomialFeatures()
.setDegree(3)

// 创建多元线性回归 learner
val mlr = MultipleLinearRegression()

// 通过 parameter map 设置 leaner 的参数
val parameters = ParameterMap()
.add(MultipleLinearRegression.Iterations, 20)
.add(MultipleLinearRegression.Stepsize, 0.5)

// 创建 PolynomialFeatures 至 MultipleLinearRegression 的 pipeline
val pipeline = polyFeatures.chainPredictor(mlr)

// 训练模型
pipeline.fit(trainingDS)
{% endhighlight %}
