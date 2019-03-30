+++
banner = ""
date = "2016-09-03T10:42:43+07:00"
tags = ["spring-boot", "maven", "debian"]
title = "Spring boot debian package"
+++

Recently got a task to create a debian package with Spring boot application to facilitate deployment process. The target OS is Ubuntu 14.04 and the application build system is Maven. Surfing the web, I found only one relative [article](https://www.ccampo.me/java/spring/linux/2016/02/15/boot-service-package.html) described how to achieve the goal, however it uses [Gradle ospackage plugin](https://github.com/nebula-plugins/gradle-ospackage-plugin), which is not compatible with Maven builds. Thus, I started to find out the best way to create debian packages using Maven and how to apply the knowledge to make a package with Spring boot application. Complete source of the project could be found on [GiHub](https://github.com/mvpotter/spring-boot-debian-package).
<!--more-->

Among various options I have chosen [jdeb](https://github.com/tcurdt/jdeb) Maven plugin. It allows to create debian packages on any platform, pretty mature and well documented.

[Spring boot introductory tutorial](https://spring.io/guides/gs/spring-boot/) was taken as a source project. The only thing that modified is spring boot maven plugin configuration. It is necessary to make jar [fully executable](ttp://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html) to be able to launch it as a service on Unix systems.

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <executable>true</executable>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Separate module is responsible for creating debian package with executable jar. Firstly, the main artifact is copied using maven dependency plugin

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>2.10</version>
    <executions>
        <execution>
            <id>copy</id>
            <phase>package</phase>
            <goals>
                <goal>copy</goal>
            </goals>
            <configuration>
                <artifactItems>
                    <artifactItem>
                        <groupId>com.springframework</groupId>
                        <artifactId>gs-spring-boot</artifactId>
                        <version>1.0.0</version>
                        <overWrite>true</overWrite>
                        <outputDirectory>${project.build.directory}</outputDirectory>
                        <destFileName>${build.finalName}.jar</destFileName>
                    </artifactItem>
                </artifactItems>
            </configuration>
        </execution>
    </executions>
</plugin>
```

And then debian package is created

```xml
<plugin>
    <artifactId>jdeb</artifactId>
    <groupId>org.vafer</groupId>
    <version>1.5</version>
    <configuration>
        <controlDir>${project.build.directory}/classes/control</controlDir>
        <deb>${project.build.directory}/${build.finalName}.deb</deb>
    </configuration>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>jdeb</goal>
            </goals>
            <configuration>
                <dataSet>
                    <data>
                        <src>${project.build.directory}/${build.finalName}.jar</src>
                        <type>file</type>
                        <mapper>
                            <type>perm</type>
                            <prefix>/var/${build.finalName}</prefix>
                            <filemode>755</filemode>
                        </mapper>
                    </data>
                    <data>
                        <type>link</type>
                        <symlink>true</symlink>
                        <linkName>/etc/init.d/${build.finalName}</linkName>
                        <linkTarget>/var/${build.finalName}/${build.finalName}.jar</linkTarget>
                    </data>
                </dataSet>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Data tags specify two entities for packaging:

- executable jar that will be copied to */var/${build.finalName}* with execute access rights
- symlink to executable jar that will be located at */etc/init.d/${build.finalName}* for installing init.d service

Another important thing is *control* directory, that is specified in plugin configuration:

```xml
<controlDir>${project.build.directory}/classes/control</controlDir>
```

Looking at the directory contents you can find **control** file. It is like a pom.xml for Maven, but for debian packages. It specifies package name, description, dependencies upon other packages, etc. Detailed fields description can be found at [debian documentation](https://www.debian.org/doc/debian-policy/ch-controlfields.html).

```
Package: ${build.finalName}
Version: ${project.version}
Section: misc
Priority: optional
Architecture: all
Depends: oracle-java8-installer | openjdk-8-jre
Maintainer: Michael Potter <supermegapotter@gmail.com>
Description: Spring boot Hello World application
Distribution: development
```

Other control files description could be found at [debian documentation](https://www.debian.org/doc/manuals/debian-faq/ch-pkg_basics.en.html#s-maintscripts).

The only thing that is left is create a package by executing ```mvn clean package```.
