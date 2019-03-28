+++
title = "Thoughts on enterprise application architecture"
date = "2019-03-27T20:31:11+07:00"
tags = []
categories = []
description = ""
menu = ""
banner = ""
images = []
draft = true
+++

I have taken part in development of tens enterprise project during my career. Usually, the main approach of building them is quite similar: layered architecture with Spring framework and JPA. However, we had many discussions concerning specific issues that I want to specify further. I am not saying that approaches that I consider correct are suitable for every project. However, I think that they may reduce cognitive complexity during development.

##### How to create an optimal subset of services that implement required business logic?

One of the popular issues that I found in projects is big service classes. For example, we are developing an online store and need to create an order. For an order we need to fill in a customer, delivery information, a set of items to buy, and payment information. Simple tutorials usually explain how to deal with simple entities, e.g. we create a service for each entity: ```OrderService```, ```ItemService```, ```CustomerService```. They can make CRUD operations on and allows to search information about specified entities. However, when we need to set order's customer or add a new item, one of the most obvious ways is to extend ```OrderService``` with required methods. That approach will lead to the situation when ```OrderService``` becomes a God service that makes almost everything.

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

To make service responsibility more separated we can move necessary operation to special services that should manage the relation. To manage items in an order we can create ```OrderItemService```, to manage order customer ```OrderCustomerService``` and so on.


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

##### Can service read entities of another one using its repository or entity manager?

I took part in a number of holy wars concerning database entities access from different kinds of services. Some people said that if you have JPA repository, you can inject any repository in any service and get required entities. Because, anyway we have the same database and it is easy to get what you need without writing additional code. Some developers even said that repository is redundant abstraction and we can inject entity manager right into the service and do everything we want. Having entity manager injected in ```OrderItemService``` we can extract  required order ```em.find(Order.class, orderId)``` and item ```em.find(Item.class, itemId)```. I understand the ideas that lead people to such conclusions, however, these approaches raise quite complex issues in a long run. In my opinion the best approach for a service to manage only its entities, interaction with others should be done using appropriate services. Because, only service knows how to create, find and update its entities appropriately. Sometimes, you have to not only get an entity from a database, but also fill in some properties by invoking external service. Or you can have a need to perform some specific modifications before saving an entity. If you do not incapsulate the way entity is managed, you can mistakenly miss some critical data when trying to update it using repository or entity manager instead of a service.

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
