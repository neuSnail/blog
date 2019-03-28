---
title: 通过maven profile配置运行环境以及@...@占位符使用
date: 2019-03-28 13:19:48
tags: 
- maven
- spring
---

在maven的pom文件中可以设置profile标签.在旧版的springMVC中可以通过这种方式结合filter和resource标签指定当前加载的配置文件.

在springboot中没必要这么麻烦 因为springboot会根据当前激活的profile自动加载application-{profile}.properties配置文件.

**注意这里maven中的profile和spring中的profile(spring.profiles.active)完全不是一个东西,maven的profile是只在编译(build)时生效的,而spring的profile是在运行时(runtime)使用的**

所以现在的目的就是通过maven的profile指定spring的profile.这一步可以通过占位符(delimiter)来实现. 如果你使用的是spring-boot-starter-parent可以直接按如下配置:

```xml
<profiles>
        <profile>
            <!-- 本地开发环境 -->
            <id>dev</id>
            <properties>
                <env>dev</env>
            </properties>
            <activation>
                <!-- 设置默认激活这个配置 -->
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
        <profile>
            <!-- 测试环境 -->
            <id>test</id>
            <properties>
                <env>test</env>
            </properties>
        </profile>
    </profiles>
```
之后在application.properties中加上一行
```properties
spring.profiles.active=@env@
```
在pom中我们为不同的profile配置了一个`env`属性,该属性可以在application.properties中通过占位符@...@来引用
在使用时我一直好奇这个@占位符是在哪定义的,后来在spring官方文档中终于查到是在spring-boot-starter-parent中定义的,这个文件在IDEA中可以直接点进去编辑,或者直接在下面的路径中找到打开编辑

.m2/repository/org/springframework/boot/spring-boot-starter-parent/2.1.0.RELEASE/spring-boot-starter-parent-2.1.0.RELEASE.pom

*(当然在实际使用时不应该直接编辑这个 parent而应该在自己的pom中重写属性)*

在properties标签中可以找到该分隔符的定义
``` xml
<resource.delimiter>@</resource.delimiter>
```
这里可以修改这个配置改成我们想用的分割符比如
``` xml
<resource.delimiter>^</resource.delimiter>
```
之后在配置文件中就可以这样使用了`spring.profiles.active=^env^`

如果想使用spring之前默认的占位符${…} 可以把这行注释掉,之后修改useDefaultDelimiters为true

```xml
<configuration>
    <delimiters>
        <delimiter>${resource.delimiter}</delimiter>
    </delimiters>
    <useDefaultDelimiters>true</useDefaultDelimiters>
</configuration>
```