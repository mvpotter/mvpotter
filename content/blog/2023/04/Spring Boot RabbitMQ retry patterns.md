+++
title = "Spring Boot RabbitMQ retry patterns"
date = "2023-04-12T21:53:20+04:00"
tags = ["Spring Boot", "RabbitMQ"]
draft = false
+++

People tend to think optimistically and working with message queues is not an exception. When events are delivered and handled appropriately - developers are happy. But what options do we have when errors happen?

> The code can be found [here](https://github.com/mvpotter/spring-boot-rabbit-retry)

## Default behaviour

Spring AMQP's default behaviour is to requeue failed message. It is useful if there is some temporal issue that prevents message to be processed, or issue affects only a subset of consumers. Message would be requeued until it reaches consumer that might process it.  However, if we have a bug in consumer's logic or there are some infra issues (e.g. network split that makes database unreachable) consumers would fail message processing and it might cause an infinite loop that would create excessive load on consumers.

To avoid infinite loop there are at least two options:

1. Turn off requeue completely by setting:

    ```yaml
    spring:
        rabbitmq:
            listener.simple.default-requeue-rejected: false
    ```

2. Throw **AmqpRejectAndDontRequeueException** from consumer.

In both cases the message would be just discarded and there won't be any opportunity to move it back to queue or analyze message content to figure out why consumer failed. 

![image](/blog/2023/04/Spring-Boot-RabbitMQ-retry-patterns-1.png)

## RetryInterceptor

Another approach is to apply retry interceptor to consumer. Consumer would be able to make a number of attempts with specified delay between them. It would increase the probability of message to be handled. However, here we either block a consumer or stop message from processing by other consumers unit retry logic finished. If consumer has a bug in its logic the message would be discarded in the end. Or if requeueing is turned on, message might continue redelivering in a loop. However, the load on consumers would be less than in case of default behaviour.

To achieve it, it is necessary to define `RetryOperationsInterceptor` and set it to listener container factory.

```java
@Bean  
public RetryOperationsInterceptor retryInterceptor() {  
return RetryInterceptorBuilder.stateless()  
                              .maxAttempts(5)  
                              .backOffOptions(2000, 2.0, 100000)  
                              .build();  
}  
  
@Bean  
public DirectRabbitListenerContainerFactory rabbitListenerContainerFactory(  
    final CachingConnectionFactory cachingConnectionFactory,  
    final DirectRabbitListenerContainerFactoryConfigurer configurer  
) {  
    final DirectRabbitListenerContainerFactory factory = new DirectRabbitListenerContainerFactory();  
    configurer.configure(factory, cachingConnectionFactory);  
    factory.setAcknowledgeMode(AcknowledgeMode.AUTO);  
    factory.setAdviceChain(retryInterceptor());  
    factory.setDefaultRequeueRejected(false);  
    return factory;  
}
```

![image](/blog/2023/04/Spring-Boot-RabbitMQ-retry-patterns-2.png)

## DLQ

Dead letter queue (DLQ) is a queue that holds undelivered or failed messages. We can bind our queue with appropriate DQL and if message fails it would be moved to DLQ. That behavior is not that useful, as we need some processing logic for messages in DQL. However, there is a nice pattern to implement repeatable event with timeout. The topology is quite simple:

1. Let's suppose we have a queue named **queue**
2. Set dead letter exchange (DLX) for **queue**.
3. Bind DLQ (let's name it **queue.dql**) to DLX.
4. For **queue.dql** we set default exchange as DLX and set dead letter routing key to **queue**.
5. Now the only thing that is left is set message TTL to **queue.dql**.

Our flow would work the following way:

1. Some message is being routed to **queue**.
2. If consumer fails to handle it, it is being routed to **queue.dql**.
3. After TTL reached message being routed to default exchange with routing key **queue**. So the message would be routed back to original **queue**. 

![image](/blog/2023/04/Spring-Boot-RabbitMQ-retry-patterns-3.png)

This approach is a bit better scalable than with retry interceptor, as consumers fail fast and message can be routed to another consumer after its TTL is reached in DLQ.  However, we still might face an infinite loop if message cannot be processed for any reason.

## DLQ + Parking lot

To get rid of the infinite loop described above parking lot can be used. Parking lot is a simple queue where we route a message after a number of consumers fails. E.g. we can check failed count header value and if it reached 5 attempts, consumer just route the message to parking lot. There messages can be checked manually and either moved back to original queue, discarded, or processed in some other way. 

![image](/blog/2023/04/Spring-Boot-RabbitMQ-retry-patterns-4.png)
