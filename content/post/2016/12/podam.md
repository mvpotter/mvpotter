+++
draft = true
tags = [
]
description = ""
menu = ""
banner = ""
title = "PODAM (POjo DAta Mocker)"
images = [
]
categories = [
]
date = "2016-12-21T22:39:40+07:00"
+++

Writing unit tests, especially for data converters, requres creating a lot of POJOs and populate them to see that fields' values are mapped correctly. In my experience, projects had simple POJOs and it was not a big deal to fill it with data manually, however, one of the last projects has complex model and creating test objects with all the relations become an issue. I started to finding out a solution that can create and populate objects for me and found a great tool called [PODAM](https://devopsfolks.github.io/podam/).

<!--more-->

PODAM is abbreviation for POjo DAta Mocker and its API for creating beans is pretty simple. All you need to do is create ```PodamFactory``` instance and use its ```manufacturePojo``` method:

```java
PodamFactory factory = new PodamFactoryImpl();
Pojo myPojo = factory.manufacturePojo(Pojo.class);
```



Another alternative that I found later is [random-beans](https://github.com/benas/random-beans/wiki) project. However, I did not have time to test it thoroughly. If you know other tools for generating and populating beans feel free to share your experience in comments.