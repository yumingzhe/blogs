---
layout: post
title: "Maven创建source、javadoc构件"
date: 2012-06-24 09:32:15 +0800
comments: true
categories: [maven, java] 
---
当我们进行项目开发时，通常要使用许多的框架，以java为例，常用的框架有hibernate，spring。当我们将整个框架下载下来时，查看其中包含的jar包，会发现通常有以xx-source.jar或xx-javadoc.jar结尾的jar包，从名字就可以看出这些jar包分别打包了源代码和文档。这样做的好处，就是可以将这些源码或文档包加入到ide的路径中，当进行调试时我们便可以跳入到框架的源码中或快速地查看文档以解决问题， 同时还可以随时对源码进行剖析。并且，将这些源码包或文档包随项目一起发布，将会给用户带来极大的方便，好处就不再赘述，下面就让我们一起看一下如何创为自己的项目创建这些便捷的jar包吧。

注意：以下内容需要有一定的maven基础，如果您还没接触过maven请查阅其他资料入门后再来参考本文档。

### 创建source构件
源码构件是比较容易创建的，只需要安装好maven-source-plugin这个插件即可。当我们进行正常的maven构建时，这个插件会自动调用(可以配置在哪个阶段调用)，为我们将源码打包。我们用下面的代码演示如何使用这个插件：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
......
<build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>2.2.1</version>
                <executions>
                    <execution>
                        <id>create-source</id><!--指定一个名字-->
                        <phase>compile</phase><!--在编译阶段生成source包-->
                        <goals>
                            <goal>jar</goal><!--指定生成的文件为jar包-->
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```
从代码中可以看出，我们需要先指定maven-source-plugin的GAV坐标，然后定义一下phase和goal即可。

### 创建Javadoc构件
与创建源码包类似，创建doc也需要一个插件——maven-javadoc-plugins。这个插件会调用JDK的javadoc工具，根据源代码生成JavaDoc文档。该插件使用方法如下所示：
```xml
<reporting>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-javadoc-plugin</artifactId>
                <version>2.9.1</version>
            </plugin>
        </plugins>
    </reporting>
```
基于上面的简单配置，当我们使用mvn site命令生成站点时，就可以得到项目源代码和测试代码的JavaDoc文档了。
