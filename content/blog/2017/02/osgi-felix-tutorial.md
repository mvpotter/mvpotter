+++
categories = [
]
title = "Pluggable architecture with Apache Felix"
date = "2017-02-24T22:31:11+07:00"
tags = ["osgi"]
description = ""
menu = ""
banner = ""
images = [
]

+++

JVM does not support dynamic component model. To solve this issue [OSGi (Open Services Gateway initiative)](https://www.osgi.org/) was found. OSGi is a framework that allows to deploy modules and libraries on the fly. To use the framework you need to install and start OSGi container (e.g. [Apache Felix](http://felix.apache.org/), [Eclipse Equinox](http://www.eclipse.org/equinox/), [Knophlerfish](http://www.knopflerfish.org/), [ProSyst](http://www.prosyst.com/) etc.) and then you can add and remove bundles (libraries and modules in OSGi terminology) to it dynamically. In the post you will find how to create basic application with pluggable architecture using [Apache Felix](http://felix.apache.org/). The source of the whole application can be found on [GitHub](https://github.com/mvpotter/osgi-felix-tutorial).
<!--more-->

The project could be divided into two parts: extensible host application (**osgi-host** module) and pluggable bundles (**hello-api**, **hello-en-bundle**, **hello-es-bundle**).

### Host application

As was mentioned host application should start OSGi container and provide API for loading / unloading bundles. To achieve this, it is required to add a dependency on Apache Felix Framework:

``` xml
<dependency>
    <groupId>org.apache.felix</groupId>
    <artifactId>org.apache.felix.framework</artifactId>
</dependency>
```

And write a couple lines of code to start OSGi container:

``` java
felix = Felix(configMap)
felix?.start()
```

As you can see, Felix constructor accepts configuration map as an argument. The map should define which packages and bundles should be lunched on startup. In our example configuration looks the following:

``` java
val configMap = StringMap()
configMap.put(Constants.FRAMEWORK_SYSTEMPACKAGES_EXTRA,
              "kotlin; version=1.0.6," +
              "kotlin.jvm.internal; version=1.0.6," +
              "com.mvpotter.osgi.hello; version=1.0.0")

val list = listOf(activator)
configMap.put(FelixConstants.SYSTEMBUNDLE_ACTIVATORS_PROP, list)
```

The first block specifies required packages, they are

- *kotlin* packages, as bundles are written in Kotlin they either should provide the dependency or host defines it itself and free bundles from this obligation.
- package that contains common API should be specified as OSGi classes are loaded with separate classloader and it is insufficiently for host to just have a dependency on API interfaces, they should be loaded by container appropriately for service instances to be casted without issues.

The last line defines bundle activators that will be launched on startup. It is necessary to launch **HostActivator** to have an access to **BundleContext**. It allows to manage bundles: install, start, stop, uninstall, list, find bundle services and so on.
Complete list of Apache Felix properties can be found in [documentation](http://felix.apache.org/documentation/subprojects/apache-felix-framework/apache-felix-framework-configuration-properties.html).

### Bundles

Bundles are built using Felix [maven-bundle-plugin](http://felix.apache.org/documentation/subprojects/apache-felix-maven-bundle-plugin-bnd.html). All you need to do is specify packaging

``` xml
<packaging>bundle</packaging>
```

and add required attributes to bundle manifest file

``` xml
<plugin>
    <groupId>org.apache.felix</groupId>
    <artifactId>maven-bundle-plugin</artifactId>
    <configuration>
        <instructions>
            <Export-Package>com.mvpotter.osgi.hello.impl</Export-Package>
            <Bundle-Activator>com.mvpotter.osgi.hello.impl.ProviderActivator</Bundle-Activator>
        </instructions>
    </configuration>
</plugin>
```

**hello-api** bundle defines interfaces that should be implemented by other bundles and further used by the host application. Other bundles contain class with API implementation and activator that registers its service in bundle context.

*main.kt* file shows how to register bundles from provided JAR files by host application and invoke their service methods.
Apache Felix documentation provides [useful tutorials](http://felix.apache.org/documentation/tutorials-examples-and-presentations/apache-felix-osgi-tutorial.html) on creating and consuming bundle services.
