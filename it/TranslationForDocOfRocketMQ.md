---
title: 为开源社区尽一份力，翻译RocketMQ官方文档
date: 2017/12/20
categories: 技术博客
---

正如在上一篇文章中写道：“据我所知，现在RocketMQ还没有中文文档。我打算自己试着在github上开一个项目，自行翻译。”我这几天抽空翻译了文档的前3个小节，发现翻译真的不是一件容易的事情，即便只是翻译这种语言复杂程度较低的技术文档。

<!-- more -->
翻译文档有很多好处，首先一点就是可以提升阅读英文文档的水平。现在很多开源项目都是国外的，即便是国内的开源项目，为了国际化影响也是先出英文文档，眼前的例子就是RocketMQ。然后，就是翻译文档分享出来，能够帮助一些英文水平还有待提高的开发者朋友们，让他们也尽快能读到阅读体验更好的项目文档。

翻译要求三个字，信、达、雅。我的翻译还远远没有到这个程度，纯属抛砖引玉。官方文档链接为http://rocketmq.apache.org/docs/motivation，请对照阅读。


---
## 为什么是RocketMQ

### 动机
在早期阶段，我们在ActiveMQ 5.x(早于5.3)的基础上构建我们的分布式消息中间件。我们的跨国业务使用它来实现异步通信、检索、社交网络活动流、数据管道，甚至在交易过程中也在使用。随着我们的交易业务量增加，来自消息集群的压力与日俱增，亟需解决。

### 为什么是RocketMQ
根据我们的研究，随着使用中的队列越来越长、虚拟主题越来越多，ActiveMQ的IO模型会到达一个瓶颈。我们尽力地试图通过节流、断路器或者降级这些手段来解决这个问题，但是都没有很好的效果。因此，我们开始注意到当时非常流行的消息解决方案，Kafka。不走运的是，Kafka并不能满足我们的需求，尤其是在低延迟和高可用这两点上，点击链接进一步了解细节。

在这样的情况下，我们决定写一个全新的消息引擎来处理这一类用途更广泛的使用案例，囊括范围从传统的发布/订阅情景到高流量实时零差错交易系统。我们相信这个解决方案能带来好处，所以我们非常愿意把这个项目向社区开源。今时今日，已经有超过100家企业在他们的业务里采用开源版本的RocketMQ。我们也在阿里云平台发布了一个基于RocketMQ的商业版PaaS产品。

下面这张表格对比了RocketMQ、ActiveMQ和Kafaka(Apache里最受欢迎的基于java的消息解决方案)
### RocketMQ vs. ActiveMQ vs. Kafka
(译者注：markdown要完整实现这个表格比较复杂，暂且搁置，延后处理)

## 快速开始
快速开始指引由详细的命令组成，告诉你如何在本机配置RocketMQ消息投递系统并且收发消息。

### 前期必要的准备
以下软件必须安装：
1. 64位操作系统，linux/Unit/Mac（推荐）；
2. 64位JDK，1.8+；
3. Maven 3.2.x
4. Git

### 克隆代码仓库和构建程序
```
  > git clone -b develop https://github.com/apache/rocketmq.git
  > cd rocketmq
  > mvn -Prelease-all -DskipTests clean install -U
  > cd distribution/target/apache-rocketmq
```
(译者注：命令集合无需翻译。下同。)

### 启动Name Server
```
  > nohup sh bin/mqnamesrv &
  > tail -f ~/logs/rocketmqlogs/namesrv.log
  The Name Server boot success...
```

### 启动Broker
```
  > nohup sh bin/mqbroker -n localhost:9876 &
  > tail -f ~/logs/rocketmqlogs/broker.log
  The broker[%s, 172.30.30.233:10911] boot success...
```

### 发送和接收消息
在发送或接收消息之前，我们需要通知客户端name servers的位置。RocketMQ提供多种实现方式。为简单起见，我们现在展示环境变量NAMESRV_ADDR的用法
```
 > export NAMESRV_ADDR=localhost:9876
 > sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
 SendResult [sendStatus=SEND_OK, msgId= ...

 > sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
 ConsumeMessageThread_%d Receive New Messages: [MessageExt...
```

### 关闭所有服务器
```
> sh bin/mqshutdown broker
The mqbroker(36695) is running...
Send shutdown request to mqbroker(36695) OK

> sh bin/mqshutdown namesrv
The mqnamesrv(36664) is running...
Send shutdown request to mqnamesrv(36664) OK
```

## 简单的消息示例
使用RocketMQ发送消息的3种方法：可靠同步发送、可靠异步发送和单向发送

本页文档展示这3种消息发送方式。检出这些带有注释的示例代码，可以让你知道每一个用例对应哪一种发送方式。

### 可靠同步发送
应用：可靠同步发送在众多场景中被使用，例如重要的通知消息、短信通知、短信营销系统，等等。
```
public class SyncProducer {
    public static void main(String[] args) throws Exception {
        //Instantiate with a producer group name.
        DefaultMQProducer producer = new
            DefaultMQProducer("please_rename_unique_group_name");
        //Launch the instance.
        producer.start();
        for (int i = 0; i < 100; i++) {
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTest" /* Topic */,
                "TagA" /* Tag */,
                ("Hello RocketMQ " +
                    i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );
            //Call send message to deliver message to one of brokers.
            SendResult sendResult = producer.send(msg);
            System.out.printf("%s%n", sendResult);
        }
        //Shut down once the producer instance is not longer in use.
        producer.shutdown();
    }
}
```

### 可靠异步发送
Application: asynchronous transmission is generally used in response time sensitive business scenarios.
应用：异步发送通常被用于对响应时间敏感的业务场景
```
public class AsyncProducer {
    public static void main(String[] args) throws Exception {
        //Instantiate with a producer group name.
        DefaultMQProducer producer = new DefaultMQProducer("ExampleProducerGroup");
        //Launch the instance.
        producer.start();
        producer.setRetryTimesWhenSendAsyncFailed(0);
        for (int i = 0; i < 100; i++) {
                final int index = i;
                //Create a message instance, specifying topic, tag and message body.
                Message msg = new Message("TopicTest",
                    "TagA",
                    "OrderID188",
                    "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                producer.send(msg, new SendCallback() {
                    @Override
                    public void onSuccess(SendResult sendResult) {
                        System.out.printf("%-10d OK %s %n", index,
                            sendResult.getMsgId());
                    }
                    @Override
                    public void onException(Throwable e) {
                        System.out.printf("%-10d Exception %s %n", index, e);
                        e.printStackTrace();
                    }
                });
        }
        //Shut down once the producer instance is not longer in use.
        producer.shutdown();
    }
}
```

### 单向发送
Application: One-way transmission is used for cases requiring moderate reliability, such as log collection.
应用：单向发送用于要求一定可靠性的场景，例如日志收集。
```
public class OnewayProducer {
    public static void main(String[] args) throws Exception{
        //Instantiate with a producer group name.
        DefaultMQProducer producer = new DefaultMQProducer("ExampleProducerGroup");
        //Launch the instance.
        producer.start();
        for (int i = 0; i < 100; i++) {
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTest" /* Topic */,
                "TagA" /* Tag */,
                ("Hello RocketMQ " +
                    i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );
            //Call send message to deliver message to one of brokers.
            producer.sendOneway(msg);

        }
        //Shut down once the producer instance is not longer in use.
        producer.shutdown();
    }
}
```
---

## 最后说点什么

翻译文档的地址在https://github.com/levenYes/RocketMQ-ChineseDoc.

感兴趣的朋友可以关注一下，给个Star。我会陆陆续续把文档翻译好，上传上去。如果有志同道合的朋友想参与进来，可以试着翻译一两节，然后新建一个Pull request。或者直接发邮件给到我的邮箱levenyes@icloud.com，互相沟通交流学习。
