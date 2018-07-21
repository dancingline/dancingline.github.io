---
layout: post
title: "python用igraph实现复杂网络社团检测"
date: 2018-07-21
excerpt: 
tags: [python, igraph, 聚类, 数据挖掘]
feature: 
comments: false
---

前几天有个作业，要求在karate数据集上做一个社团检测，其实就是一个聚类。简单介绍一下数据集，空手道俱乐部（karate）数据集包含34个节点，由于管理不善，分裂成了两个社团，更多有关数据集和复杂网络的内容参见这里：[Mark Newman](http://www-personal.umich.edu/~mejn/)

由于数据集是gml格式的，搜了一下python可以用igraph这个库来读取，详细了解了一下发现这个包还带了社团检测的函数，所以直接调包了，顺便研究了一下网上的代码做了个可视的结果，把聚类结果直接画出来。

#####**1、先期准备**

我在Windows下的python版本是3.5，写这个需要装两个包，[igraph](https://pypi.org/project/python-igraph/0.6.5/)和cairo，igraph用来读图和社团检测，cairo用来进行绘图，比较不凑巧的是这两个包在Windows下都不能正常pip，需要找别人编译好的[第三方包](https://www.lfd.uci.edu/~gohlke/pythonlibs/)，（官方推荐），找到对应版本的包下载下来然后pip。

#####**2、社团检测**

社团检测这里用的是igraph的`community_label_propagation()`，这个函数我追踪源码进不去，查了一下用的是**标签传播算法（LPA）**。

LPA的思路比较简单，这里简单说明一下：初始所有节点都有一个不同的标签，每轮迭代针对单个节点考虑与它直接相连的节点，将出现次数最多的标签赋给这个节点，持续这个过程直到所有节点的标签都不再变化。

LPA感觉非常像KNN，由于不需要计算到所有点的距离，复杂度应该不会太高，不过这个算法不够稳定，具有一定的随机性，这一点在下面会再次提到。

#####**3、聚类结果绘制**

绘图使用cairo，它的plot函数可以接收一个map类型的参数，用来传递图里一组节点的标签、颜色、大小等设置，map里的键是属性名，值是一个存储了各个节点对应属性值的列表，基本就是下面这样：

```python
visual_style = {}
visual_style["vertex_label"] = g.vs["name"]
visual_style["vertex_color"] = ['' for i in range(g.vcount())]
```

下面贴两张绘制出来的结果，聚类结果不太稳定，在1~4个类之间波动，大部分是2、3个的情况。

![](http://dle.oss-cn-beijing.aliyuncs.com/18-7-21/29144028.jpg)

![](http://dle.oss-cn-beijing.aliyuncs.com/18-7-21/84723159.jpg)

![](http://dle.oss-cn-beijing.aliyuncs.com/18-7-21/60766601.jpg)

![](http://dle.oss-cn-beijing.aliyuncs.com/18-7-21/60710099.jpg)

最后附代码：

```python
from igraph import *

# 预置部分颜色用于绘图
colors_type = ["yellow", "red", "green", "coral", "alice blue", "cyan", "pink"]
if __name__ == '__main__':
    g = Graph.Read_GML("karate.gml")    # 读图
    g.vs["name"] = [str(i+1) for i in range(g.vcount())]
    visual_style = {}
    # 设置节点显示的名字
    visual_style["vertex_label"] = g.vs["name"]
    # 使用标签传递算法进行聚类
    v = g.community_label_propagation()
    print(v)
    # 根据聚类结果对节点进行染色
    visual_style["vertex_color"] = ['' for i in range(g.vcount())]
    for i in range(len(v)):
        for j in v[i]:
            visual_style["vertex_color"][j] = colors_type[i]
    # 可视化聚类结果 打印到屏幕
    plot(g, None, **visual_style)
```