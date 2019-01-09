---
layout: post
title:  "Compass Switch"
categories: Arducopter Compass
tags: APM Compass
author: Byte
---

* content
{:toc}

由于飞行器罗盘容易挂掉，故现在开始研究APM 罗盘切换。

## 如何做：

检测到罗盘里的时间轴不更新，则开始执行切换罗盘，切换罗盘就是切换主罗盘，那么就是切换那个主标志：_primary = 0 (用第一罗盘)/ _primary = 1（用第二罗盘）。

COMPASS_PRIMARY: Choose primary compass
Note: This parameter is for advanced users
If more than one compass is available, this selects which compass is the primary. When external compasses are connected, they will be ordered first. NOTE: If no external compass is attached, this parameter is ignored.



## 代码
目前有两个个调用readMagData()(此函数为切换罗盘的函数)：
- calcQuatAndFieldStates();
- SelectMagFusion();

目前有两种方案，获得那个timeout值：
- 通过构建虚函数 uint8_t maxCount = _ahrs->get_compass()->get_count(); 然后在readMagData();里调用此虚函数，即可得到超时标志。
- 或者直接在5983驱动那边用两个全局变量，计算出超时标志，然后直接传过来进行切换即可。
- 若是想救多次，那么还得在那个3100中加入这样的判断。若是，5983超时标志已经触发，那么切换到3100后，发现3100也超时，若是5983标志正常，那么再次切换到5983，哈哈哈，就这样愉快地决定了。
```js
yawAlignComplete;          // true when yaw alignment is complete
```

```js
    // If we are using the compass and the magnetometer has been unhealthy for too long we declare a timeout
    if (magHealth) {
        magTimeout = false;
        lastHealthyMagTime_ms = imuSampleTime_ms;
         printf("magTimeout = false\n");
         printf("imuSampleTime_ms=%d lastHealthyMagTime_ms=%d use_compass()=%d\n",imuSampleTime_ms, lastHealthyMagTime_ms, use_compass());
    } else if ((imuSampleTime_ms - lastHealthyMagTime_ms) > frontend->magFailTimeLimit_ms && use_compass()) {
        magTimeout = true;
        printf("magTimeout = true\n");
    }

    // imuSampleTime_ms 在 readIMUData() 里 imuSampleTime_ms = AP_HAL::millis();更新。
    // magFailTimeLimit_ms = 10000; number of msec before a magnetometer failing innovation consistency checks is declared failed (msec) 大于10000毫秒 这样判断显然错误。
    // use_compass() return true if the compass should be used for yaw calculations.

```



```js
/**
 * 获取剩余时间
 * @param  {Number} endTime    截止时间
 * @param  {Number} deviceTime 设备时间
 * @param  {Number} serverTime 服务端时间
 * @return {Object}            剩余时间对象
 */
let getRemainTime = (endTime, deviceTime, serverTime) => {
    let t = endTime - Date.parse(new Date()) - serverTime + deviceTime
    let seconds = Math.floor((t / 1000) % 60)
    let minutes = Math.floor((t / 1000 / 60) % 60)
    let hours = Math.floor((t / (1000 * 60 * 60)) % 24)
    let days = Math.floor(t / (1000 * 60 * 60 * 24))
    return {
        'total': t,
        'days': days,
        'hours': hours,
        'minutes': minutes,
        'seconds': seconds
    }
}
```

<del>获取服务器时间可以使用 mtop 接口 `mtop.common.getTimestamp` </del>

然后可以通过下面的方式来使用：

```js
// 获取服务端时间（获取服务端时间代码略）
getServerTime((serverTime) => {

    //设置定时器
    let intervalTimer = setInterval(() => {

        // 得到剩余时间
        let remainTime = getRemainTime(endTime, deviceTime, serverTime)

        // 倒计时到两个小时内
        if (remainTime.total <= 7200000 && remainTime.total > 0) {
            // do something

        //倒计时结束
        } else if (remainTime.total <= 0) {
            clearInterval(intervalTimer);
            // do something
        }
    }, 1000)
})
```

这样的的写法也可以做到准确倒计时，同时也比较简洁。不需要隔段时间再去同步一次服务端时间。

## 补充

在写倒计时的时候遇到了一个坑这里记录一下。

**千万别在倒计时结束的时候请求接口**。会让服务端瞬间 QPS 峰值达到非常高。

![](https://img.alicdn.com/tfs/TB1LBzjOpXXXXcnXpXXXXXXXXXX-154-71.png)

如果在倒计时结束的时候要使用新的数据渲染页面，正确的做法是：

在倒计时结束前的一段时间里，先请求好数据，倒计时结束后，再渲染页面。

关于倒计时，如果你有什么更好的解决方案，欢迎评论交流。
