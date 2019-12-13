---
layout: post
title: kube-proxy的ipvs模式"no route to host"问题
tags: kube-proxy ipvs "no route to host"
---

我们在使用腾讯云容器服务(tke)的过程中，在使用kube-proxy的ipvs模式时，遇到"no route to host"问题，以下对该问题的分析来自tke团队的roc，这里记录为运维日志。

## 环境

- tke 1.12.4(1.14.3) 托管集群.
- 操作系统：ubuntu16.04.1 LTSx86_64
- kube-proxy ipvs模式 /usr/bin/kube-proxy --proxy-mode=ipvs --ipvs-min-sync-period=1s --ipvs-sync-period=5s --ipvs-scheduler=rr --masquerade-all=true --kubeconfig=/etc/kubernetes/kubeproxy-kubeconfig --hostname-override=172.21.128.111 --v=2
- 运行时：Docker version 18.06.3-ce, build d7080c1


## 排查过程

1.14也压测了一样有这个问题。已经找到根因，这个问题在最新版仍然存在，还在 Open 状态的 issue 81775 描述的正是这个问题: https://github.com/kubernetes/kubernetes/issues/81775


观察到现象是服务滚动更新后，新的 pod ip:port 加入 rs 列表，权重为1，旧的 rs 对应 pod ip:port 权重变为0，并且通过`ipvsadm -lnc | grep <old-podip> `发现大量 TIME_WAIT 状态连接。当旧的 rs 完全没有连接后才会被踢掉(ActiveConn+InactiveConn=0)，这个是 kube-proxy 的逻辑，通过这里的代码可以确认：https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/graceful_termination.go#L170

kube-proxy 没有立即踢掉旧 rs 的原因是 ipvs 模式支持连接的 graceful termination，等存量连接完全 close 了才踢掉 rs。

问题奇怪的地方在于只要这个rs没有被踢掉，即便权重为0，后面新的连接仍有可能被转发到这个rs上。我们怀疑是 ipvs 内核模块的问题，找到相关同事一起研究这块的代码逻辑，在 ipvs 调度函数里没有看到会将新连接调度到权重为0的rs，然后又看调度之前的逻辑，对应 net/netfilter/ipvs/ip_vs_core.c 中的 ip_vs_in 函数，发现会先在转发连接表查是否已经有连接了（匹配五元组），如果有连接说明不是新连接就不会调度，直接发给这个连接对应的之前已经调度过的rs(也不会判断权重)，如果没匹配到说明是新的连接，就会走到调度这里(rr, wrr 等调度策略)，这个逻辑也没问题。

另外我们发现它还有个重新调度的逻辑，也就是即便匹配到了连接，也可能不会转发到这个连接对应的rs，在满足重新调度的所有条件的情况下会将此连接做一次重新调度，在判断是否需要重新调度这里，加一个权重的判断，如果检测到匹配到了要转发的rs并且这个rs的权重为0，就重新调度(重新选择rs，会过滤掉权重为0的rs)

重新编译加载ipvs内核模块后，再次压测，发现没有之前的问题了。猜想：之前权重被置为0后，后面的新连接实际匹配到了ipvs连接转发表里的旧连接，也就是没有认为它是一个新连接，而是当作旧连接处理，但旧连接对应的 pod 早已销毁，转发过去就会发生 "no route to host"。

那为什么还会匹配到旧连接呢？猜测是：被压测的服务又作为client发起很多请求调用另一个服务，client副本少，它收到和发起的请求量就巨大，而且是短连接，源端口很快接近耗尽，而client发的短连接关闭后会进去 TIME_WAIT 状态，等待 2MSL 时长（2分钟）才会被清理，当源端口不够用了先释放 TIME_WAIT 连接的源端口给 client 用，但 TIME_WAIT 状态的连接仍然存在与 ipvs 的连接转发表中，由于新连接使用了 TIME_WAIT 状态连接使用的源端口，源IP始终是client ip，这样端口复用的报文的五元组就可以跟 ipvs 连接转发表中的 TIME_WAIT 连接匹配上，然后认为这个"新连接"是旧连接，将转发给这个旧连接之前的rs，而之前的rs对应的pod早已销毁，转发过去就 "no route to host" 了。 我这里压测环境比较极端，client 和 server 的副本都只有一个。

后面搜索发现这实际是一个已知并且未解决的问题，找到了个这个 issue: https://github.com/kubernetes/kubernetes/issues/81775 ，并且根据 issue 讨论也证实猜想。

社区的解决思路主要是优化 ipvs graceful termination 的逻辑，有一个 issue 讨论是否不给非活跃连接支持 graceful termination: https://github.com/kubernetes/kubernetes/issues/81308 还有一个讨论是否支持自定义 ipvs 的超时时间 https://github.com/kubernetes/kubernetes/pull/85517 (但这个对 TIME_WAIT 不生效，TIME_WAIT 取决于 MSL)

## 总结

这个问题通常只会在集群中client高并发访问service发生，虽然通过改ipvs内核逻辑做实验能规避，但应该会引入其它问题，那个修改只是为了缩小排查范围，“暴力”的将匹配到连接但权重为0的rs重新调度了。社区暂时还没有一个完美的解决方案，这里只能建议做一些规避措施: 集群中的 client 副本可以多增加一些，并且加反亲和避免调度到同一个节点，这样client的并发量就分散到不同节点了，避免源端口重用导致命中 ipvs 的旧连接，从而规避这个问题。

## 其它找到的一些有用的文章

- https://engineering.dollarshaveclub.com/kubernetes-fixing-delayed-service-endpoint-updates-fd4d0a31852c
- https://fuckcloudnative.io/posts/kubernetes-fixing-delayed-service-endpoint-updates/ 中文版本
- 五元组：源IP地址，源端口，目的IP地址，目的端口，和传输层协议这五个量组成的一个集合
