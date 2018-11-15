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

创建绑定接口，绑定`example-topic`输入通道（默认情况下，会绑定到RabbitMQ的同名Exchange或Kafaka的同名Topic）。

```java
interface ExampleBinder {

    String NAME = "example-topic";

    @Input(NAME)
    SubscribableChannel input();

}
```
