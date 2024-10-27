---
title: spring-boot:run 运行的小小问题
date: 2024-10-23
tags:
  - java
---

已经多年不写 `java` 应用了， 但最近维护着一个旧项目，基于 `jdk 11`（现在已经是 `jdk 24` 了），使用 `spring` 全家桶，`maven` 构建工具。10 几个微服务，用户量却没多少，维护很麻烦。实在看不下去，试着改造成单体，减少维护成本。

其实，真的，现在的服务不急于都是直接上微服务，现在 `单体 > 微服务` 收益，以前单体弊端很大，主要是因为一个发动就得重启服务，或者一个小 `bug` 就导致程序崩溃，但现在得益于像 `k8s` 这种部署架构，可以一定程序上弥补，又可以平滑地更新服务。

更关键的是，多少数据量你真正知道吗？ 也不多扯了，回归正题。

对于已经不习惯在 `IDE` 里面直接 `run` 或者 `debug run` 的方式，是使用命令行来启动。基于肌肉记忆，修改了子模块 `logback-spring.xml` 的日志配置，然后 `mvn clean compile && mvn -pl xxx spring-boot:run` 运行，但是没效果。 又试了 `mvn clean package && java -jar xxx.jar` 能正常。

那么 `mvn -pl xxx spring-boot:run` 和 `java -jar xxx.jar` 运行方式有什么不同吗？

`jar` 包方式的运行，是把所有的资源都打进去了， 所以是脱离 `maven`，而 `mvn` 命令行这种是依赖 `spring boot` 的插件 `spring-boot-maven-plugin` 来构建。 既然有构建了，但为什么启动后对修改不生效？ 这是因为 `mvn spring-boot:run` 启动的时候，依赖包没有重新 `install` 导致，如上我们只是 `compile`。所以需要改成 `mv install && mvn -pl xxx spring-boot:run`。

为什么使用 `mvn` 来启动？ 这是因为可以快速指定一个模块启动，构建更快，但如果修改了子模块相关的，需要重新 `install` 下。
