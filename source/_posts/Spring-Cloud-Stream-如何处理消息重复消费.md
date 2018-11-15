---
title: Spring Cloud Stream 如何处理消息重复消费
date: 2018-11-15 13:23:26
tags: 
  - Spring
  - Spring Boot
  - Spring Cloud
  - Spring Cloud Stream
  - RabbitMQ
  - Kafka
---

# 问题重现

## 构建消息消费端

- 创建绑定接口，绑定`example-topic`输入通道（默认情况下，会绑定到RabbitMQ的同名Exchange或Kafaka的同名Topic）。

```java
interface ExampleBinder {

    String NAME = "example-topic";

    @Input(NAME)
    SubscribableChannel input();

}
```
- 对上述输入通道创建监听与处理逻辑。

```java
@EnableBinding(ExampleBinder.class)
public class ExampleReceiver {

    private static Logger logger = LoggerFactory.getLogger(ExampleReceiver.class);

    @StreamListener(ExampleBinder.NAME)
    public void receive(String payload) {
        logger.info("Received: " + payload);
    }

}
```
- 创建应用主类和配置文件

```java
@SpringBootApplication
public class ExampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(ExampleApplication.class, args);
    }

}
```

```
spring.application.name=stream-consumer-group
server.port=0
```

> 这里设置server.port=0，以方便在本地启动多实例来重现问题。

## 构建消息生产端

比较简单，需要注意的是，使用@Output创建一个同名的输出绑定，这样发出的消息才能被上述启动的实例接收到。具体实现如下：

```java
@RunWith(SpringRunner.class)
@EnableBinding(value = {ExampleApplicationTests.ExampleBinder.class})
public class ExampleApplicationTests {

	@Autowired
	private ExampleBinder exampleBinder;

	@Test
	public void exampleBinderTester() {
        exampleBinder.output().send(MessageBuilder.withPayload("Produce a message from : http://blog.didispace.com").build());
	}

	public interface ExampleBinder {

		String NAME = "example-topic";

		@Output(NAME)
		MessageChannel output();

	}

}
```
启动上述测试用例之后，可以发现之前启动的两个实例都收到的消息，并在日志中打印了：Received: Produce a message from : http://blog.didispace.com。消息重复消费的问题成功重现！

# 使用消费组解决问题

如何解决上述消息重复消费的问题呢？我们只需要在配置文件中增加如下配置即可：

```
spring.cloud.stream.bindings.example-topic.group=aaa
```
当我们指定了某个绑定所指向的消费组之后，往当前主题发送的消息在每个订阅消费组中，只会有一个订阅者接收和消费，从而实现了对消息的负载均衡。只所以之前会出现重复消费的问题，是由于默认情况下，任何订阅都会产生一个匿名消费组，所以每个订阅实例都会有自己的消费组，从而当有消息发送的时候，就形成了广播的模式。


另外，需要注意上述配置中`example-topic`是在代码中`@Output`和`@Input`中传入的名字。



