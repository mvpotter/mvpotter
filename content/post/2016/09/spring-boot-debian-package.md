+++
banner = "/post/2016/09/spring-boot-debian-package.png"
date = "2016-09-03T10:42:43+07:00"
tags = ["spring-boot", "maven", "debian"]
title = "Spring boot debian package"
draft = true
+++

Recently got a task to create a debian package with Spring boot application to facilitate deployment process. The target OS is Ubuntu 14.04 and the application build system is Maven. Surfing the web, I found only one relative [article](https://www.ccampo.me/java/spring/linux/2016/02/15/boot-service-package.html) described how to achieve the goal, however it uses [Gradle ospackage plugin](https://github.com/nebula-plugins/gradle-ospackage-plugin), which is not compatible with Maven builds. Thus, I started to find out the best way to create debian packages using Maven and how to apply the knowledge to make a package with Spring boot application.
<!--more-->
[jdeb](https://github.com/tcurdt/jdeb)
[Installing Spring Boot applications](http://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html)


```
mvn clean install
```
