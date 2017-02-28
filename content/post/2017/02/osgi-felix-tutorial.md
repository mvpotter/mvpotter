+++
categories = [
]
title = "Pluggable architecture with Apache Felix"
draft = true
date = "2017-02-24T22:31:11+07:00"
tags = [
]
description = ""
menu = ""
banner = ""
images = [
]

+++

JVM does not support dynamic component model. To solve this issue [OSGi (Open Services Gateway initiative)](https://www.osgi.org/) was found. OSGi is a framework that allows to deploy modules and libraries on the fly. For using the framework you need to install and start OSGi container (e.g. [Apache Felix](http://felix.apache.org/), [Eclipse Equinox](http://www.eclipse.org/equinox/), [Knophlerfish](http://www.knopflerfish.org/), [ProSyst](http://www.prosyst.com/) etc.) and then you can add and remove bundles (libraries and modules in OSGi terminology) to it dynamically. In the post you will find how to create basic application with pluggable architecture using [Apache Felix](http://felix.apache.org/). The source of the whole application can be found on [GitHub](https://github.com/mvpotter/osgi-felix-tutorial).
<!--more-->

1. Simple app worth thousand docs
2. [maven-bundle-plugin](http://felix.apache.org/documentation/subprojects/apache-felix-maven-bundle-plugin-bnd.html)

[documentation](http://felix.apache.org/documentation.html)
[OSGi bundles tutorial](http://felix.apache.org/documentation/tutorials-examples-and-presentations/apache-felix-osgi-tutorial.html)
[embedded mode](http://felix.apache.org/documentation/subprojects/apache-felix-framework/apache-felix-framework-launching-and-embedding.html)