---
layout: post
title: "JUnit5 Console Launcher"
subtitle: "JUnit5 Console Launcher run with dependency jar"
author: "Archon"
header-img: "img/header.jpg"
catalog: true
tags:
  - java
  - junit
  - unit test
---

# 说明与依赖
说明：https://junit.org/junit5/docs/current/user-guide/#running-tests-console-launcher

依赖：https://repo1.maven.org/maven2/org/junit/platform/junit-platform-console-standalone/

# 使用

## 编译

```sh
javac -cp junit-platform-console-standalone-1.4.2.jar \
-d out \
MyTest.java
```

## 运行

加入JDBC依赖

```sh
java -cp out:junit-platform-console-standalone-1.4.2.jar:mysql-connector-java-5.1.46-bin.jar \
org.junit.platform.console.ConsoleLauncher \
-m MyTest#test1 \
-m MyTest#test2
```
