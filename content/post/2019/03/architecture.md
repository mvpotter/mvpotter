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

##### How to create optimal subset of services that implement required business logic?



##### Can service read entities of another one using its repository or entity manager?
