+++
title = "Beyond layered architecture"
date = "2019-03-30T16:04:11+07:00"
tags = ["architecture"]
categories = []
description = ""
menu = ""
banner = ""
images = []
draft = false
+++

I have taken part in development of tens enterprise project during my career. Usually, the main approach of building them is quite similar: layered architecture with Spring framework and JPA. It is quite easy to understand the [layered architecture](https://en.wikipedia.org/wiki/Multitier_architecture) principles and get a vision how to separate layers vertically. However, developers usually have issues when they need to make a decision concerning horizontal separation of concerns and how exactly business logic should be separated.

I took part in a number of discussions with different teams on how to do it. Of course, details were different each time, however, main approaches were the same. Sometimes, correct decisions were made from the very beginning of the project, sometimes we reached them when faced architectural issues. I want to describe the approaches we used. It does not mean that it is the only good way of building applications, but it reduced cognitive complexity and saved us a lot of time when we worked on enterprise applications.

##### How to create an optimal subset of services that implement required business logic?

One of the popular issues that I found in projects is big service classes (e.g. 1000 lines of code). For example, we are developing an online store and need to implement the scenario of forming an order. It is required to fill in a customer, a set of items to buy, delivery and payment information. Simple tutorials usually explain how to deal with simple entities, e.g. we create a service for each entity: ```OrderService```, ```ItemService```, ```CustomerService```. They can make CRUD operations and allow to search information about specified entities. However, when we need to set an order's customer or add a new item, one of the most obvious ways is to extend ```OrderService``` with required methods.

```java
public interface ComplexOrderService {
  Order create();
  Order find(UUID id);
  Order save(Order order);
  void delete(UUID id);
  Order setCustomer(UUID orderId, UUID customerId);
  Order addItem(UUID orderId, UUID itemId);
  Order removeItem(UUID orderId, UUID itemId);
  ...
}
```

This approach will lead to the situation when ```OrderService``` becomes a God service that makes almost everything. And it will expand when new business requirements need to be implemented. To make service responsibility more separated we can move necessary operations to special services that should manage the relation. To manage items in an order we can create ```OrderItemService```, to manage order customer ```OrderCustomerService``` and so on.

```java
public interface SimpleOrderService {
  Order create();
  Order find(UUID id);
  Order save(Order order);
  void delete(UUID id);
}
```

```java
public interface OrderItemService {
  Order addItem(UUID orderId, UUID itemId);
  Order removeItem(UUID orderId, UUID itemId);
}
```

```java
public interface OrderCustomerService {
  Order setCustomer(UUID orderId, UUID customerId);
}
```

Services become smaller and manageable.

##### Can service read entities of another one using its repository or entity manager?

Another quite common idea is to inject any repository in any service to work with required entities. Anyway we have the same database and it is easy to get what you need without writing additional code. Moreover, it does not break layered architecture principle. Some developers even say that repository is redundant abstraction and we can inject entity manager right into the service and do everything we want. Having entity manager injected in ```OrderItemService``` we can extract required order ```em.find(Order.class, orderId)``` and item ```em.find(Item.class, itemId)```.

I agree that you do not need to think and do much to work with required entities, however, these approaches raise quite complex issues in a long run. In my opinion, the best approach for a service to manage only its entity, interaction with others should be done using appropriate services. Because, only service knows how to create, find and update its entities appropriately. Sometimes, you have to not only get an entity from a database, but also make some optimizations or fill in some properties by invoking external service. Or you can have a need to perform some specific modifications before saving an entity. If you do not incapsulate the way entity is managed, you can mistakenly miss some critical data when trying to update it using repository or entity manager instead of a service.

```java
public class BadOrderItemService {
  private OrderRepository orderRepository;
  private ItemRepository itemRepository;
  ...
}
```

```java
public class GoodOrderItemService {
  private OrderService orderService;
  private ItemService itemService;
  ...
}
```

##### Should services return business objects only?

Layered architecture states that specific type of objects should be transferred between layers. We have entity objects in persistence layer, business objects in service layer and DTO object in presentation layer. The question is, if we make a call from one service to another should only business objects be transferred or entities are allowable too? On the one hand, business object should contain all necessary information, on the other, having an entity we can use JPA features like lazy loading and retrieve data optimally. Moreover, using entity, it is easier to update entities relations.

For me it is a classical conflict between clean architecture and the pursuit of optimization. Services should always return and receive business objects as arguments. It will hide the magic that JPA proxies do and prevents a typical type of errors, when you try to get some lazy field from an entity when session was closed.

In a couple of projects we had a situation when entities were used through all the layers. And as an intermediate step of the refactoring we used both entities and business objects in services contracts. However, outer services interfaces that used in controllers contained only business objects and entities usage were only inside inner interfaces used by other services.


That's it for now. Keep your code clean!
