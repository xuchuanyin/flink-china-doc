---
mathjax: include
title: 距离度量

# Sub navigation
sub-nav-group: batch
sub-nav-parent: flinkml
sub-nav-title: 距离度量
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

不同的距离度量方法在不同的分析类型中非常有用，Flink ML内置了许多标准的距离度量实现。
此外，用户也可以通过实现`DistanceMetric`这个trait来创建自定义的距离度量。

## 内置实现

目前，Flink ML支持以下距离度量实现：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 20%">度量实现</th>
        <th class="text-center">说明</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td><strong>欧氏距离(Euclidean Distance)</strong></td>
        <td>
          $$d(\x, \y) = \sqrt{\sum_{i=1}^n \left(x_i - y_i \right)^2}$$
        </td>
      </tr>
      <tr>
        <td><strong>欧氏距离平方(Squared Euclidean Distance)</strong></td>
        <td>
          $$d(\x, \y) = \sum_{i=1}^n \left(x_i - y_i \right)^2$$
        </td>
      </tr>
      <tr>
        <td><strong>余弦相似度(Cosine Similarity)</strong></td>
        <td>
          $$d(\x, \y) = 1 - \frac{\x^T \y}{\Vert \x \Vert \Vert \y \Vert}$$
        </td>
      </tr>
      <tr>
        <td><strong>切比雪夫距离(Chebyshev Distance)</strong></td>
        <td>
          $$d(\x, \y) = \max_{i}\left(\left \vert x_i - y_i \right\vert \right)$$
        </td>
      </tr>
      <tr>
        <td><strong>曼哈顿距离(Manhattan Distance)</strong></td>
        <td>
          $$d(\x, \y) = \sum_{i=1}^n \left\vert x_i - y_i \right\vert$$
        </td>
      </tr>
      <tr>
        <td><strong>明氏距离(Minkowski Distance)</strong></td>
        <td>
          $$d(\x, \y) = \left( \sum_{i=1}^{n} \left( x_i - y_i \right)^p \right)^{\rfrac{1}{p}}$$
        </td>
      </tr>
      <tr>
        <td><strong>Tanimoto距离(Tanimoto Distance)</strong></td>
        <td>
          $$d(\x, \y) = 1 - \frac{\x^T\y}{\Vert \x \Vert^2 + \Vert \y \Vert^2 - \x^T\y}$$ 
          其中$\x$和$\y$为位矢量(bit-vector)
        </td>
      </tr>
    </tbody>
  </table>

## 自定义实现

用户也可以通过实现`DistanceMetric`这个trait来创建自定义的距离度量。

{% highlight scala %}
class MyDistance extends DistanceMetric {
  override def distance(a: Vector, b: Vector) = ... // your implementation for distance metric
}

object MyDistance {
  def apply() = new MyDistance()
}

val myMetric = MyDistance()
{% endhighlight %}
