---
title: RocketMQ官方文档-用户指引
date: 2017/12/29
categories: 文档翻译
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

## 简单示例
### 简单的消息示例
使用RocketMQ发送消息的3种方法：可靠同步发送、可靠异步发送和单向发送

This page exemplifies these three message-sending ways. Checkout the notes along with the example to figure out which way to use for your specific use case.

本页文档展示这3种消息发送方式。检出这些带有注释的示例代码，可以让你知道每一个用例对应哪一种发送方式。

#### 可靠同步发送
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

#### 可靠异步发送
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

#### 单向发送
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

## 顺序消息示例
### 顺序消息
RocketMQ提供使用先进先出算法的顺序消息实现。

以下示例展示了如何发送/接收全局顺序消息和分区顺序消息。

#### 发送消息示例代码
```
public class OrderedProducer {
    public static void main(String[] args) throws Exception {
        //Instantiate with a producer group name.
        MQProducer producer = new DefaultMQProducer("example_group_name");
        //Launch the instance.
        producer.start();
        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 100; i++) {
            int orderId = i % 10;
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("TopicTestjjj", tags[i % tags.length], "KEY" + i,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
            @Override
            public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                Integer id = (Integer) arg;
                int index = id % mqs.size();
                return mqs.get(index);
            }
            }, orderId);

            System.out.printf("%s%n", sendResult);
        }
        //server shutdown
        producer.shutdown();
    }
}
```

#### 订阅消息示例代码
```
public class OrderedConsumer {
    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("example_group_name");

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("TopicTest", "TagA || TagC || TagD");

        consumer.registerMessageListener(new MessageListenerOrderly() {

            AtomicLong consumeTimes = new AtomicLong(0);
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                       ConsumeOrderlyContext context) {
                context.setAutoCommit(false);
                System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                this.consumeTimes.incrementAndGet();
                if ((this.consumeTimes.get() % 2) == 0) {
                    return ConsumeOrderlyStatus.SUCCESS;
                } else if ((this.consumeTimes.get() % 3) == 0) {
                    return ConsumeOrderlyStatus.ROLLBACK;
                } else if ((this.consumeTimes.get() % 4) == 0) {
                    return ConsumeOrderlyStatus.COMMIT;
                } else if ((this.consumeTimes.get() % 5) == 0) {
                    context.setSuspendCurrentQueueTimeMillis(3000);
                    return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                }
                return ConsumeOrderlyStatus.SUCCESS;

            }
        });

        consumer.start();

        System.out.printf("Consumer Started.%n");
    }
}
```

## 广播示例
### 什么是广播
广播就是向一个主题的所有订阅者发送同一条消息。如果你想让一个主题的所有订阅者收到消息，广播是一个很好的选择。

#### 生产者示例
```
public class BroadcastProducer {
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
        producer.start();

        for (int i = 0; i < 100; i++){
            Message msg = new Message("TopicTest",
                "TagA",
                "OrderID188",
                "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(msg);
            System.out.printf("%s%n", sendResult);
        }
        producer.shutdown();
    }
}
```

#### 消费者示例
```
public class BroadcastConsumer {
    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("example_group_name");

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        //set to broadcast mode
        consumer.setMessageModel(MessageModel.BROADCASTING);

        consumer.subscribe("TopicTest", "TagA || TagC || TagD");

        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {
                System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        consumer.start();
        System.out.printf("Broadcast Consumer Started.%n");
    }
}
```

## 延时消息示例
### 什么是延时消息？
延时消息提供了一种不同于普通消息的实现形式——它们只会在设定的时限到了之后才被递送出去。


### 应用
1. 启动消费者，等待即将接收的订阅消息
```
 import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
 import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
 import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
 import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
 import org.apache.rocketmq.common.message.MessageExt;
 import java.util.List;

 public class ScheduledMessageConsumer {

     public static void main(String[] args) throws Exception {
         // Instantiate message consumer
         DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("ExampleConsumer");
         // Subscribe topics
         consumer.subscribe("TestTopic", "*");
         // Register message listener
         consumer.registerMessageListener(new MessageListenerConcurrently() {
             @Override
             public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> messages, ConsumeConcurrentlyContext context) {
                 for (MessageExt message : messages) {
                     // Print approximate delay time period
                     System.out.println("Receive message[msgId=" + message.getMsgId() + "] "
                             + (System.currentTimeMillis() - message.getStoreTimestamp()) + "ms later");
                 }
                 return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
             }
         });
         // Launch consumer
         consumer.start();
     }
 }
```

2. 发送延时消息
```
 import org.apache.rocketmq.client.producer.DefaultMQProducer;
 import org.apache.rocketmq.common.message.Message;

 public class ScheduledMessageProducer {

     public static void main(String[] args) throws Exception {
         // Instantiate a producer to send scheduled messages
         DefaultMQProducer producer = new DefaultMQProducer("ExampleProducerGroup");
         // Launch producer
         producer.start();
         int totalMessagesToSend = 100;
         for (int i = 0; i < totalMessagesToSend; i++) {
             Message message = new Message("TestTopic", ("Hello scheduled message " + i).getBytes());
             // This message will be delivered to consumer 10 seconds later.
             message.setDelayTimeLevel(3);
             // Send the message
             producer.send(message);
         }

         // Shutdown producer after use.
         producer.shutdown();
     }

 }
```

3. 验证
你应该会在消息被存储之后10秒钟看到它们被消费。

## 批量消息示例
### 为什么要采用批量消息？
批量发送消息可以提升投递小内存消息时的性能。

### 使用限制
同一批消息必须满足以下条件：相同的主题、相同的waitStoreMsgOK变量设置，而且都不支持延时发送

另外，一个批量消息的大小最好不要大于1MiB。

### 如何使用批量消息
如果你一次发送的消息总大小不超过1MB，使用批量消息就很简单
```
String topic = "BatchTest";
List<Message> messages = new ArrayList<>();
messages.add(new Message(topic, "TagA", "OrderID001", "Hello world 0".getBytes()));
messages.add(new Message(topic, "TagA", "OrderID002", "Hello world 1".getBytes()));
messages.add(new Message(topic, "TagA", "OrderID003", "Hello world 2".getBytes()));
try {
    producer.send(messages);
} catch (Exception e) {
    e.printStackTrace();
    //handle the error
}
```

### 切分后用List保存
只有在你发送大内存批量消息而且不确定是否达到大小限制（1MiB）的时候，才会变得复杂。

这时候，你应该把它们切分，然后用List保存
```
public class ListSplitter implements Iterator<List<Message>> {
    private final int SIZE_LIMIT = 1000 * 1000;
    private final List<Message> messages;
    private int currIndex;
    public ListSplitter(List<Message> messages) {
            this.messages = messages;
    }
    @Override public boolean hasNext() {
        return currIndex < messages.size();
    }
    @Override public List<Message> next() {
        int nextIndex = currIndex;
        int totalSize = 0;
        for (; nextIndex < messages.size(); nextIndex++) {
            Message message = messages.get(nextIndex);
            int tmpSize = message.getTopic().length() + message.getBody().length;
            Map<String, String> properties = message.getProperties();
            for (Map.Entry<String, String> entry : properties.entrySet()) {
                tmpSize += entry.getKey().length() + entry.getValue().length();
            }
            tmpSize = tmpSize + 20; //for log overhead
            if (tmpSize > SIZE_LIMIT) {
                //it is unexpected that single message exceeds the SIZE_LIMIT
                //here just let it go, otherwise it will block the splitting process
                if (nextIndex - currIndex == 0) {
                   //if the next sublist has no element, add this one and then break, otherwise just break
                   nextIndex++;  
                }
                break;
            }
            if (tmpSize + totalSize > SIZE_LIMIT) {
                break;
            } else {
                totalSize += tmpSize;
            }

        }
        List<Message> subList = messages.subList(currIndex, nextIndex);
        currIndex = nextIndex;
        return subList;
    }
}
//then you could split the large list into small ones:
ListSplitter splitter = new ListSplitter(messages);
while (splitter.hasNext()) {
   try {
       List<Message>  listItem = splitter.next();
       producer.send(listItem);
   } catch (Exception e) {
       e.printStackTrace();
       //handle the error
   }
}
```

## 消息过滤器示例
在大多数情况下，标签是一种帮助你选择所需消息的简单有用的设计。例如：
```
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("CID_EXAMPLE");
consumer.subscribe("TOPIC", "TAGA || TAGB || TAGC");
```

该消费者会接收含有TAGA、TAGB或TAGC标签的消息。但是，因为有一条消息只能打一个标签的限制，在复杂的场景下可能会失效。在这种情况下，你可以用SQL表达式来过滤消息。

### 规则
根据你在发送消息时设定的属性，SQL特性可以做相应的运算。遵循RocketMQ预设的语法，你可以实现一些有趣的逻辑。这里有一个例子：
![将文字转为图片](http://p0zshkwk1.bkt.clouddn.com/17-12-21/54128752.jpg)

### 语法
RocketMQ只预设了一些基本的语法来支持这个特性。你也可以很容易地扩展它。
1. Numeric comparison, like >, >=, <, <=, BETWEEN, =;
2. Character comparison, like =, <>, IN;
3. IS NULL or IS NOT NULL;
4. Logical AND, OR, NOT;

以下是基本类型：
1. Numeric, like 123, 3.1415;
2. Character, like ‘abc’, must be made with single quotes;
3. NULL, special constant;
4. Boolean, TRUE or FALSE;

### 使用限制
只有push类型的消费者可以通过SQL92标准的语句来筛选消息。接口如下：
```
public void subscribe(final String topic, final MessageSelector messageSelector)
```

### 生产者示例
在发送消息时，你可以使用putUserProperty()方法为消息设置属性。
```
DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
producer.start();

Message msg = new Message("TopicTest",
    tag,
    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET)
);
// Set some properties.
msg.putUserProperty("a", String.valueOf(i));

SendResult sendResult = producer.send(msg);

producer.shutdown();
```

### 消费者示例
在消费消息时，使用MessageSelector.bySql()方法和遵循SQL92标准的语句来筛选消息
```
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");

// only subsribe messages have property a, also a >=0 and a <= 3
consumer.subscribe("TopicTest", MessageSelector.bySql("a between 0 and 3");

consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
consumer.start();
```

## 日志插件管理示例
RocketMQ日志插件管理提供了log4j插件、log4j2插件和logback插件，都可以在业务中使用，以下是如何设置的示例，

### log4j
log4j配置文件如下.
```
log4j.appender.mq=org.apache.rocketmq.logappender.log4j.RocketmqLog4jAppender
log4j.appender.mq.Tag=yourTag
log4j.appender.mq.Topic=yourLogTopic
log4j.appender.mq.ProducerGroup=yourLogGroup
log4j.appender.mq.NameServerAddress=yourRocketmqNameserverAddress
log4j.appender.mq.layout=org.apache.log4j.PatternLayout
log4j.appender.mq.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-4r [%t] (%F:%L) %-5p
```

使用xml文件配置log4j时，如下配置并添加一个异步插件：
```
<appender name="mqAppender1" class="org.apache.rocketmq.logappender.log4j.RocketmqLog4jAppender">
    <param name="Tag" value="yourTag" />
    <param name="Topic" value="yourLogTopic" />
    <param name="ProducerGroup" value="yourLogGroup" />
    <param name="NameServerAddress" value="yourRocketmqNameserverAddress"/>
    <layout class="org.apache.log4j.PatternLayout">
        <param name="ConversionPattern" value="%d{yyyy-MM-dd HH:mm:ss}-%p %t %c - %m%n" />
    </layout>
</appender>

<appender name="mqAsyncAppender1" class="org.apache.log4j.AsyncAppender">
    <param name="BufferSize" value="1024" />
    <param name="Blocking" value="false" />
    <appender-ref ref="mqAppender1"/>
</appender>
```

### log4j2
log4j2配置如下。如果你希望隐藏，就配置一个异步插件作为引用。【译者注：有问题】
```
<RocketMQ name="rocketmqAppender" producerGroup="yourLogGroup" nameServerAddress="yourRocketmqNameserverAddress"
     topic="yourLogTopic" tag="yourTag">
    <PatternLayout pattern="%d [%p] hahahah %c %m%n"/>
</RocketMQ>
```

### logback
使用logback，异步插件也是必需的。  
```
<appender name="mqAppender1" class="org.apache.rocketmq.logappender.logback.RocketmqLogbackAppender">
    <tag>yourTag</tag>
    <topic>yourLogTopic</topic>
    <producerGroup>yourLogGroup</producerGroup>
    <nameServerAddress>yourRocketmqNameserverAddress</nameServerAddress>
    <layout>
        <pattern>%date %p %t - %m%n</pattern>
    </layout>
</appender>

<appender name="mqAsyncAppender1" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>1024</queueSize>
    <discardingThreshold>80</discardingThreshold>
    <maxFlushTime>2000</maxFlushTime>
    <neverBlock>true</neverBlock>
    <appender-ref ref="mqAppender1"/>
</appender>
```

## OpenMessaging兼容示例
OpenMessaging力图建立行业准则和消息分发、流计算领域标准，为金融、电子商务、物联网和大数据领域提供通用框架。设计原则包括，面向云、简单易用、灵活度高、独立于编程语言，以及能在分布式异构环境中使用。这些标准的一致性将会使得开发一款跨所有主流平台和操作系统的异构消息分发应用成为可能。

RocketMQ提供了关于OpenMessaging 0.1.0-alpha版本的部分实现，以下例子展示了如何按照OpenMessaging规范接入RocketMQ.

### OMSProducer
下面这个例子展示了如何向RocketMQ broker用同步、异步、单向发送的方式发送消息。
```
public class OMSProducer {
    public static void main(String[] args) {
        final MessagingAccessPoint messagingAccessPoint = MessagingAccessPointFactory
            .getMessagingAccessPoint("openmessaging:rocketmq://IP1:9876,IP2:9876/namespace");

        final Producer producer = messagingAccessPoint.createProducer();

        messagingAccessPoint.startup();
        System.out.printf("MessagingAccessPoint startup OK%n");

        producer.startup();
        System.out.printf("Producer startup OK%n");

        {
            Message message = producer.createBytesMessageToTopic("OMS_HELLO_TOPIC", "OMS_HELLO_BODY".getBytes(Charset.forName("UTF-8")));
            SendResult sendResult = producer.send(message);
            System.out.printf("Send sync message OK, msgId: %s%n", sendResult.messageId());
        }

        {
            final Promise<SendResult> result = producer.sendAsync(producer.createBytesMessageToTopic("OMS_HELLO_TOPIC", "OMS_HELLO_BODY".getBytes(Charset.forName("UTF-8"))));
            result.addListener(new PromiseListener<SendResult>() {
                @Override
                public void operationCompleted(Promise<SendResult> promise) {
                    System.out.printf("Send async message OK, msgId: %s%n", promise.get().messageId());
                }

                @Override
                public void operationFailed(Promise<SendResult> promise) {
                    System.out.printf("Send async message Failed, error: %s%n", promise.getThrowable().getMessage());
                }
            });
        }

        {
            producer.sendOneway(producer.createBytesMessageToTopic("OMS_HELLO_TOPIC", "OMS_HELLO_BODY".getBytes(Charset.forName("UTF-8"))));
            System.out.printf("Send oneway message OK%n");
        }

        producer.shutdown();
        messagingAccessPoint.shutdown();
    }
}
```

### OMSPullConsumer
使用OMS PullConsumer从特定队列中拉取消息。
```
public class OMSPullConsumer {
    public static void main(String[] args) {
        final MessagingAccessPoint messagingAccessPoint = MessagingAccessPointFactory
            .getMessagingAccessPoint("openmessaging:rocketmq://IP1:9876,IP2:9876/namespace");

        final PullConsumer consumer = messagingAccessPoint.createPullConsumer("OMS_HELLO_TOPIC",
            OMS.newKeyValue().put(NonStandardKeys.CONSUMER_GROUP, "OMS_CONSUMER"));

        messagingAccessPoint.startup();
        System.out.printf("MessagingAccessPoint startup OK%n");

        consumer.startup();
        System.out.printf("Consumer startup OK%n");

        Message message = consumer.poll();
        if (message != null) {
            String msgId = message.headers().getString(MessageHeader.MESSAGE_ID);
            System.out.printf("Received one message: %s%n", msgId);
            consumer.ack(msgId);
        }

        consumer.shutdown();
        messagingAccessPoint.shutdown();
    }
}
```

### OMSPushConsumer
把OMS PushConsumer和某一个特定的队列绑定在一起，通过MessageListener消费消息。
```
public class OMSPushConsumer {
    public static void main(String[] args) {
        final MessagingAccessPoint messagingAccessPoint = MessagingAccessPointFactory
            .getMessagingAccessPoint("openmessaging:rocketmq://IP1:9876,IP2:9876/namespace");

        final PushConsumer consumer = messagingAccessPoint.
            createPushConsumer(OMS.newKeyValue().put(NonStandardKeys.CONSUMER_GROUP, "OMS_CONSUMER"));

        messagingAccessPoint.startup();
        System.out.printf("MessagingAccessPoint startup OK%n");

        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            @Override
            public void run() {
                consumer.shutdown();
                messagingAccessPoint.shutdown();
            }
        }));

        consumer.attachQueue("OMS_HELLO_TOPIC", new MessageListener() {
            @Override
            public void onMessage(final Message message, final ReceivedMessageContext context) {
                System.out.printf("Received one message: %s%n", message.headers().getString(MessageHeader.MESSAGE_ID));
                context.ack();
            }
        });

    }
}
```

## 常见问题
以下是一般情况下关于RocketMQ项目的常见问题。

### 一般问题
1. 为什么我们要开发RocketMQ项目，而不是选择其他的产品？
请参考《为什么是RocketMQ》。

2. 如果要使用RocketMQ，我必须安装其他的软件吗，例如Zookeeper？
不需要。RocketMQ可以独立运行。

### 使用问题
1. 新建消费者ID从哪里开始消费消息？
  1. 如果消息是该主题在3天之内发送的，那么消费者会从保存在服务端的第一条消息开始消费。
  2. 如果消息是主题在3天之前发送的，那么消费者会从保存在服务端的最后一条消息开始消费，换句话来说，从消息队列的队尾开始。
  3. 如果消费者重启，那么它会从重启前最后一个消费位置开始。

2. 如果消费失败，要如何重新消费消息？
  1. 集群消费模式。如果一条消息消费失败，消费业务逻辑代码会返回Action.ReconsumerLater、NULL或者抛出一个异常，将会重新尝试消费，尝试次数最多可达16次。在此之后，如果还是消费失败，这条消息将会被丢弃。
  2. 广播消费模式。广播消费仍然会确保一条消息至少被消费一次，但是不提供重新发送选项。

3. 如果发生消费失败，要如何查询消费失败的消息？
  1. 根据时间使用主题查询，你可以查询到在一段时间内消费失败的消息。
  2. 使用主题和消息ID可以准确查询到该消息。
  3. 使用主题和消息键可以准确地查询同一类有相同消息键的消息。

4. 消息送达是否有且仅有一次？
RocketMQ确保所有的消息至少送达一次。在绝大多数情况下，消息不会重复。

5. 如何添加一个新boker？
  1. 启动一个新的boker，然后在相应的name servers所在的list上注册。
  2. 默认情况下，只有内部系统主题和消费者群组会自动被创建。如果你想在新节点上使用你自己的业务主题和消费者群组，请从已有的broker上复制它们。RocketMQ提供了Admin管理工具和命令行来满足这些需求。

### 配置相关问题
以下回答都是基于默认值不变的情况，默认值可以通过修改配置文件进行更改。

1. 消息会在服务器上保存多久？
存储的消息会最多保存3天，如果3天之后还没有被消费则会被删除。

2. 消息体的大小限制是？
通常是256KB。

3. 如何设置消费者线程的数量？
启动消费者服务时，设置ConsumeThreadNums属性，举例如下：
```
consumer.setConsumeThreadMin(20);
consumer.setConsumeThreadMax(20);
```

### 错误
1. 如果你启动一个生产者服务或消费者服务时失败了，错误提示是生产者群组重复或消费者群组重复，要如何处理？
原因：使用相同的生产者、消费者群组在同一个JVM初始化多个生产者/消费者示例可能会导致客户端启动失败。
解决方法：确保一个JVM对应一个生产者/消费者群组只启动一个生产者/消费者实例。

2. 如果在广播模式，消费者在开始加载JSON文件时失败，要如何处理？
原因：Fastjson的版本太低，以至于无法允许广播消费者加载offsets.json文件，导致该消费者启动失败。损坏的fastjson文件也会导致同样的问题。
解决方法：Fastjson的版本要升级到rocketmq客户端依赖版本，以确保本地offsets.json文件可以被正常加载。默认条件下offsets.json文件存放位置是/home/{user}/.rocketmq_offsets。或者检查一下fastjson是否完整。

3. broker崩溃有什么影响？
Master崩溃
如果你有另外一个broker集合可用，那么消息就可以不再发送到这个已经崩溃的boker集合。目前，消息仍然可以被发送到原来的主题。消息仍然可以通过slaves被消费。

部分salve崩溃
只要还有另外一个正在运作的slave，就不会对发送消息有任何影响。消费消息也不会受到影响，除非消费者群组设置了优先消费该slave的消息。默认情况下，消费者群组优先消费来自master的消息。

所有slaves崩溃
发送消息到master不会有任何影响，但是，如果master设置了SYNC_MASTER，那么生产者将会收到一个SLAVE_NOT_AVAILABLE的提醒，提醒说该消息未能发送到任意一个slaves。消费消息也不会受到影响，除非消费者群组设置了优先消费该slave的消息。默认情况下，消费者群组优先消费来自master的消息。

4. 生产者报告“没有主题路由信息”，如何判断哪里出了问题？
这种问题会发生在当你尝试发送消息到一个主题而这个主题的路由信息对该生产者不可用的时候。
  1. 确定生产者可以连接到name server，而且能够从该处获取路由元信息。
  2. 确定name servers的确保存了主题的路由元信息。通过使用admin工具或者web控制台，你可以根据topicRoute查询name server的路由元信息。
  3. 确定你的brokers正在发送heartbeats给同一list上的name servers，而且你的生产者正在跟它们连接着。
  4. 确定该主题的权限是6(rw-)，或者至少是2(-w-).

如果你找不到这个主题，那就通过admin工具命令updateTopic或者web控制台在一个boker上创建它。
