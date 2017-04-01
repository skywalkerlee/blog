---
title: 实现一个分布式消息队列(ohmq)
date: 2017-04-01 11:29:22
tags: [mq,消息队列,raft,etcd,分布式,ohmq,ohmyqueue]
categories: 技术
---

>ohmq is a distributed message queue written in golang

`ohmq`(ohmyqueue)是一个使用`etcd`做coordinator的分布式消息队列，[源码地址](https://github.com/skywalkerlee/ohmyqueue)。

# 架构图
![](/images/arch.png)

<!--more-->

# Features
* 使用gRPC进行通信,protobuf作为数据交换格式,支持TLS
* 消息具有生命周期,过期数据会被清理,在生命周期内可以多次被消费
* 支持水平扩展
* 支持故障恢复

# DONE list
* Pub/Sub
* 过期数据清理
* backend storage使用rocks
* 水平扩展

# TODO list
* consumer group (load balance)
* backend storage支持bitcask
* cluster manager
* 重构数据结构
* 更改客户端连接逻辑

# etcd
`etcd`是一个基于`raft`算法的高可用键值存储系统,主要用于共享配置和服务发现。
`ohmq`使用`etcd`做服务发现，服务协调和选主等。

# 实现思路
## 服务启动
节点启动时去`etcd`注册自己的相关信息。
## topic
`topic`创建需要在`etcd`注册，每个节点都监控`topic`，如果发现有新`topic`注册或`topicleader`消失则当前服务的`topic`数量少于`topic`平均数量的节点去竞选为`topicleader`。每个`topicleader`对该`topic`进行读写服务，并把数据发送到其他非`leader`节点进行备份。

关键代码：
```go
func (broker *Broker) watchTopics() {
	resp, _ := broker.Client.Get(context.TODO(), "topicname", clientv3.WithPrefix())
	for _, v := range resp.Kvs {
		broker.topics.AddTopic(string(v.Key[9:]))
		tlresp, _ := broker.Client.Get(context.TODO(), "topicleader"+string(v.Key[9:]))
		if tlresp.Count == 0 {
			go broker.voteTopicleader(string(v.Key[9:]))
		}
	}
	wch := broker.Client.Watch(context.TODO(), "topicname", clientv3.WithPrefix())
	for wresp := range wch {
		for _, ev := range wresp.Events {
			switch ev.Type.String() {
			case "PUT":
				broker.topics.AddTopic(string(ev.Kv.Key[9:]))
				resp, _ := broker.Client.Get(context.TODO(), "brokerleader", clientv3.WithPrefix())
				var sum int
				for _, v := range resp.Kvs {
					tmp, _ := strconv.Atoi(string(v.Value))
					sum += tmp
				}
				if len(broker.leaders) <= sum/int(resp.Count) {
					go broker.voteTopicleader(string(ev.Kv.Key[9:]))
				}
			}
		}
	}
}
```

## producer
`producer`查询`etcd`获得`topicleader`地址，连接该地址进行发布。发布成功后ohmq会更改该`topic`在`etcd`中的状态。

## consumer
`producer`查询`etcd`获得`topicleader`地址，连接该地址并对`topic`状态进行监控，`topic`发生改变则去`ohmq`进行拉取。
