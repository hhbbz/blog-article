---
title: SpringBoot+RocketMQ做分布式消息队列
date: 2018-03-26 11:01:46
updated: 2018-03-26 19:07:11
categories:
- 后端
tags:
- MQ
- 分布式
- Java
- Spring
---

# 项目地址

[spring-boot-aliRocketMQ-starter](https://github.com/hhbbz/spring-boot-aliRocketMQ-starter)

# application.yml

```yml
   mq-config:
   producerId: PID_*
   consumerId: CID_*
   accessKey: *
   secretKey: *
   onsAddr: *
   topic: *
```

# 添加启动类用于初始化消费者

```java
@Component
public class RocketMQRunner implements CommandLineRunner {

    private static final Logger logger = LoggerFactory.getLogger(RocketMQRunner.class);

    @Autowired
    private MQConfig mqConfig;
    @Autowired
    @Qualifier("orderConsumer")
    private OrderConsumer orderConsumer;
    /**消息监听器**/
    @Autowired
    private ConsumerHandler consumerHandler;

    /**
     * 初始化订阅者，生产者信息，启动
     * @param args
     * @throws Exception
     */
    @Override
    public void run(String... args) throws Exception {
        orderConsumer.subscribe(mqConfig.getTopic(), mqConfig.getTag(), consumerHandler);
        orderConsumer.start();
    }
}
```

# 生产消息

```java
@Service
public class SysMessageProducer {
    private static final Logger logger = LoggerFactory.getLogger(SysMessageProducer.class);
    @Autowired
    private MQHelper<SysMessage> mqHelper;
    @Autowired
    private MQConfig mqConfig;

    /**
     * 此方法用于在消息中心创建消息后调用推送消息
     *
     * @param sysMessage
     */
    public void PushMessageWhenCreate(SysMessage sysMessage) {
        if (AssertValue.isNotNull(sysMessage)) {
            //发送消息到队列中
            try {
                ProducerMessage<SysMessage> producerMessage = new ProducerMessage<SysMessage>()
                        .setTopic(mqConfig.getTopic())
                        .setTags("middle")
                        .setName(MQHandlerType.PUSH_APPMESSAGE_NEWS.getTypeName())
                        .setKey(MQHandlerType.PUSH_APPMESSAGE_NEWS.toString())
                        .setBody(sysMessage)
                        .setShardingKey(String.valueOf(sysMessage.getId()))
                        .setState("none");

                //设置立刻发送消息
                producerMessage.setType(RocketMQServiceConstant.SYNCHRONOUS_ORDER_MESSAGE);
                producerMessage.setStartDeliveryTime(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()));
                //创建消息
                Message message = mqHelper
                        .generateMessage(producerMessage);
                //发送消息
                SendResult sendResult = mqHelper.sendMessage(message, producerMessage);
                logger.info(new Date() + " 发送成功! Topic:" + mqConfig.getTopic() + " msgId: " + sendResult.getMessageId());
            } catch (Exception e) {
                e.printStackTrace();
                logger.error("消息生产失败,将进行两次重试", e.getMessage());
                logger.info("发送失败");
                throw new RestInternalServerErrorException(ExceptionEnumeration.SYS_NEWS_PUSH_FAIL, "平台消息推送失败");
            }
        } else {
            throw new RestInternalServerErrorException(ExceptionEnumeration.SYS_NEWS_SELECT_FAIL, "找不到该平台消息");
        }
    }
}
```

# 监听器消费消息

```java
@Component
public class ConsumerHandler implements MessageOrderListener {

    private static final Logger logger = LoggerFactory.getLogger(ConsumerHandler.class);

    /**
     * 消费消息 handler
     *
     * @param message
     * @param context
     * @return
     */
    @Override
    public OrderAction consume(Message message, ConsumeOrderContext context) {

    }
}
```
