---
title: nsq使用记录
date: 2017-03-31 11:03:14
tags: [nsq,消息队列,mq,消息,队列]
categories: 技术
---
NSQ是一个基于Go语言的分布式实时消息平台，它基于MIT开源协议发布，代码托管在[GitHub](https://github.com/nsqio/nsq)。
# 组件
## nsqd
`nsqd`是一个守护进程，负责接收，排队，投递消息给客户端。
它可以独立运行，不过通常它是由`nsqlookupd`实例所在集群配置的。
## nsqlookupd
`nsqlookupd`是一个coordinator。客户端通过查询`nsqlookupd`来发现指定`topic`的生产者，和`nsqd`节点的`topic`和`channel`信息。
## nsqadmin
`nsqadmin`是一个web服务器，用来汇集集群的实时统计，并执行不同的管理任务。
<!--more-->
# nsqd
nsqd被设计成可以同时处理多个数据流的模式。
在nsqd中，数据流被称为`topic`，每个`topic`有1个或多个`channel`,每个`channel`都拥有一份`topic`中所有消息的拷贝。
每个`topic`和每个`channel`的缓存相互独立。
## topic

`topic`由第一次发布或第一次订阅创建。
## channel

`channel`由第一次订阅创建。
消费者订阅`topic`时要指定所订阅的`channel`。
当有消息被发布时,每个`channel`随机选择一个订阅该`channel`的消费者进行数据推送。
如果所有的消费者都订阅了相同的`channel`就成了`队列模式`。

# 上码

`pub`:
```go
func main() {
	logs.EnableFuncCallDepth(true)
	logs.SetLogFuncCallDepth(3)
	log := logs.GetLogger()
	if len(os.Args) < 3 {
		log.Fatal("args error")
	}
	producer, err := nsq.NewProducer("172.27.31.156:4150", nsq.NewConfig())
	if nil != err {
		log.Fatal(err)
	}
	defer producer.Stop()
	producer.SetLogger(log, nsq.LogLevelError)
	err = producer.Ping()
	if nil != err {
		log.Fatal(err)
	}
	producer.Publish(os.Args[1], []byte(os.Args[2]))
}
```

`sub`:
```go
func main() {
	logs.EnableFuncCallDepth(true)
	logs.SetLogFuncCallDepth(3)
	log := logs.GetLogger()
	if len(os.Args) < 3 {
		log.Fatal("args error")
	}
	consumer, err := nsq.NewConsumer(os.Args[1], os.Args[2], nsq.NewConfig())
	if nil != err {
		log.Println(err)
		return
	}
	defer consumer.Stop()
	consumer.SetLogger(log, nsq.LogLevelError)
	consumer.AddHandler(&sub{})
	err = consumer.ConnectToNSQD("172.27.31.156:4150")
	if nil != err {
		log.Println(err)
		return
	}
	sigch := make(chan os.Signal)
	signal.Notify(sigch, syscall.SIGTERM, syscall.SIGINT, syscall.SIGKILL)
	<-sigch
}

type sub struct {
}

func (s *sub) HandleMessage(message *nsq.Message) error {
	logs.Debug(string(message.Body))
	return nil
}

```
