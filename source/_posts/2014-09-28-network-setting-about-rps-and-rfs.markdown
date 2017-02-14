---
layout: post
title: "服务器CPU0 占用偏高分析(RPS&RFS)"
date: 2014-09-28 17:23:15 +0800
comments: true
categories: tcp 
---
##背景
线上Linux 服务器，使用`mpstat -P ALL 1`看每个核的CPU使用，总是发现CPU0使用率比其他核高很多。对于多核CPU来说，CPU0是相当关键的，如果CPU0负载高，那么会影响其他核的性能，因为CPU各个核之间是需要调度的，这个调度工作就靠CPU0。

##分析
深入观察了一下，发现CPU0负载较重的服务器，往往也是网络负载，特别是inbound负载较大的。于是怀疑和网络收包处理有关。
<!--more-->
对于outbound，发送数据包，因为是应用程序跑在多个核上，分别调用系统write写数据到NIC，可以很好的利用多核。

![outbound 多核使用](https://raw.githubusercontent.com/xinheli/xinheli.github.com/master/images/RPS_RFS_1.png)

对于inbound，系统默认是使用CPU0来接受数据，处理各种软弱中断，将处理的结果在发送给用户态使用。如果是多条接收队列和多重中断线路的NIC可以帮助数据包并行分发。这就需要RPS和RFS来帮忙了。

###RPS
RPS全称是Receive Packet Steering, 这是Google工程师Tom Herbert提交的内核补丁, 在2.6.35进入Linux内核. 这个patch采用软件模拟的方式，实现了多队列网卡所提供的功能，分散了在多CPU系统上数据接收时的负载, 把软中断分到各个CPU处理，而不需要硬件支持，大大提高了网络性能。

RPS实现了数据流的hash归类，它通过数据包相关的信息（比如IP地址和端口号）来创建CPU核分配的hash表项，当一个数据包从NIC转到内核网络子系统时就从该hash表内获取其对应分配的CPU核（首次会创建表项）。并把软中断的负载均衡分到各个cpu，实现了类似多队列网卡的功能。

![inbound 多核使用](https://raw.githubusercontent.com/xinheli/xinheli.github.com/master/images/RPS_RFS_2.png)

###RFS
RFS 全称是 Receive Flow Steering, 这也是Tom提交的内核补丁，它是用来配合RPS补丁使用的，是RPS补丁的扩展补丁，它把接收的数据包送达应用所在的CPU上，提高cache的命中率。

这两个补丁往往都是一起设置，来达到最好的优化效果, 主要是针对单队列网卡多CPU环境(多队列多重中断的网卡也可以使用该补丁的功能，但多队列多重中断网卡有更好的选择:SMP IRQ affinity。

###如何设置
[官方](https://www.kernel.org/doc/Documentation/networking/scaling.txt)给出了详细的建议设置参数。

####设置RPS 
修改`/sys/class/net/<dev>/queues/rx-<n>/rps_cpus` 该数值是一个bitmap，需要几号CPU负责收包，就设置那一位为1。默认是0，就是说使用CPU0。如果设置了ffff(系统是16核),说明16个CPU都会参与到收包。当然也可以具体设置某一个核，这样可以实现不同队列绑定不同的CPU。不过一般不需要这么复杂，全部修改为绑定所有的CPU就好。

####设置RFS
修改`/proc/sys/net/core/rps_sock_flow_entries` 为32768。这个数具体[来历](https://www.kernel.org/doc/Documentation/networking/scaling.txt) `We have found that a value of 32768 for rps_sock_flow_entries works fairly well on a moderately loaded server.` 

修改`/sys/class/net/<dev>/queues/rx-<n>/rps_flow_cnt` 具体的数值就是rps_sock_flow_entries / N。N是rx的数目。

###实现效果
具体效果如下，更多在[这里](http://wenku.baidu.com/view/315d2c8571fe910ef12df838.html).

![TCP测试](https://raw.githubusercontent.com/xinheli/xinheli.github.com/master/images/RPS_RFS_3.png)

![UDP测试](https://raw.githubusercontent.com/xinheli/xinheli.github.com/master/images/RPS_RFS_4.png)

特别说明，在UDP使用环境下，不要关联到单独CPU上，具体原因比较复杂，参考[这里](http://wenku.baidu.com/view/315d2c8571fe910ef12df838.html) 


