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
<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/Loiter_adjust_velocity.png"  >

接着进入调整速度函数：

<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/adjust_velocity.png"  >

再进入真正避障速度调整：

<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/adjust_velocity_p.png"  >

最关键的是：

<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/1.png" >
<img src="https://raw.githubusercontent.com/GKbytes/gkbytes.github.io/master/photos/2.png"  >

## 目前存在问题
* 前后走动会引起左右避障;
* 速度过快时，提前避障幅度太大。（提前避障做的不是很好）


## 后续
* LOIT_SPEED : 最大水平速度，单位为cm/s(i.e. 1250 = 12.5m/s)。【默认：1250】（老版本默认5m/s）
* LOIT_ACC_MAX : 最大加速度，单位为cm/s/s。较高的值会使直升机加速并更快地停止。【500】（值越大，加速和停止性能越好）为
* LOIT_ANG_MAX : 以厘米为单位的最大倾斜角（即3000 = 30度）。默认情况下，此值为零，这会导致使用ANGLE_MAX参数的值。【0】
* LOIT_BRK_ACCEL: 制动时的最大加速度（cm/s/s）（即飞行员将杆移动到中心位置）。较高的值将更快地停止车辆。【250】
* LOIT_BRK_DELAY: 一旦飞行员将手杖放在中心位置，制动开始前的延迟时间。【1】
* LOIT_BRK_JERK: 制动时加速度的最大变化，单位为cm/s/s/s。较高的数字将使车辆更快地达到最大制动角度，较低的数字将使制动更平稳【500】
* PSC_POSXY_P : （在上面的屏幕截图的右上方显示为“位置XY（距离速度）”将水平位置误差（即所需位置和实际位置之间的差值）转换为朝向目标位置的所需速度。通常不需要调整它。【0.8】
* PSC_VELXY_P  （显示为“Velocity XY（Vel to Accel）”）将所需速度朝向目标转换为所需的加速度。得到的所需加速度变为倾斜角，然后传递到稳定模式使用的相同角度控制器。通常不需要调整它。【1.6】
