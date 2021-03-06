# 背景介绍

### 前言

这个案例来源自一个真实的拉新推广活动。已注册用户成功邀请一个新用户注册，即可获得一定的积分奖励，每邀请10个新用户获取的积分，可以兑换一张话费充值卡。活动上线之后，吸引了大量的黑产团伙，造成了推广资金的大量浪费。

黑产团伙手里会控制大量的手机号和身份证号，然后利用安卓模拟器，或者用群控系统操作真实手机设备，从服务器下发控制指令，自动化完成邀请、注册、兑换等流程。

对于黑产团伙的检测有多种:

* 基于前端数据的机器行为检测，比如模拟器检测，或者通过搜集手机上的一些信息，手势轨迹等判断其真人操作等概率
* 基于后端数据等机器行为检测，主要通过用户访问各个业务界面的频率、顺序等特征，判断其为真人的概率

薅羊毛与反薅羊毛，本质上是一个成本与效益的博弈，不存在绝对有效的防御方法，只要给的奖励够多，黑产团伙完全可以人工，用真人身份来赚奖励。所有黑产团伙检测的最终目标并不是消灭黑产，而是想办法提高他们**大规模**薅取奖励的成本。

站在黑产团伙的角度来说，他们的首要目标就是尽可能降低自己薅羊毛的成本，比如使用模拟器、比如在同一个手机上登陆更多的账号等。

一个比较简单、直觉的想法，可能是去统计每个独立设备登陆的账户数，每个IP上登陆的账户数等。

本案例利用账户与账户之间的邀请行为，以及设备关联关系，构建图模型，希望能找到一个有效的方式检测图中可能属于黑产团伙的账号。

### 数据预览

数据来源于两张表，**积分兑换表**和**邀请注册表**。简单期间，我删除了不必要的字段，这两张表的字段说明如下:

{% code title="积分兑换表" %}
```bash
order_id          订单id
order_date        订单时间
recvr_phone       奖品接收人手机号
recvr_imei        奖品接收人手机IMEI
recvr_ip          奖品接收人IP
recvr_reg_time    奖品接收人注册时间
sendr_phone       积分兑换人手机号
sendr_imei        积分兑换人手机IMEI
sendr_ip          积分兑换人IP
```
{% endcode %}

其业务逻辑是 `sendr` 消耗了自己的积分给 `recvr` 的手机充了话费。

{% code title="邀请注册表" %}
```bash
recvr_phone       被邀请人手机号
recvr_reg_time    被邀请人注册时间
recvr_imei        被邀请人手机IMEI
recvr_ip          被邀请人IP
sendr_phone       邀请人手机号
sendr_reg_time    邀请人注册时间
sendr_imei        邀请人手机IMEI
sendr_ip          邀请人IP
```
{% endcode %}

其业务逻辑是`sendr` 成功邀请了 `recvr` 注册， `sendr` 获取了积分奖励。

