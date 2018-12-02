---
layout: post
title:  "arducopter避障分析"
categories: arducopter
tags:  UAV APM arducopter sf40 避障  
author: Byte
---

* content
{:toc}


## 前言

最近在研究Arducopter的避障功能，发现其避障碍效果做的很差，速度控制时候出了问题，最近也在研究怎么优化此状况。

##  内容
在留待模式（loiter mode）下，找到调用此函数地方就算开启避障的大门：
<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/Loiter_adjust_velocity.png"  height="100" width="500">
接着进入调整速度函数：
<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/adjust_velocity.png"  height="400" width="500">
再进入真正避障速度调整：
<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/adjust_velocity_p.png"  height="300" width="500">
最关键的是：
<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/1.png"  height="600" width="500">
<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/2.png"  height="450" width="500">

## 目前存在问题
* 前后走动会引起左右避障;
* 速度过快时，提前避障速度太快。（提前避障做的不是很好）
