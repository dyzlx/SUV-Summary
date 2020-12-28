SUV 项目总结
===========

## 一、项目简述

​		SUV全称为STP Utilization Visualization（STP利用率可视化），STP是指爱立信硬件测试环境中的一套标准的测试组件。该项目的目标是将测试环境中的全部STP的利用率借助第三方的UI做可视化，这里STP的利用率主要包括：单位时间段内TGF STP的使用率、单位时间段内ERIS STP的使用率、单位时间段内STP的活跃时间、单位时间段内STP上执行的测试用例的通过率。用户（Tester）通过第三方UI将了解到某个STP在某个时间段下使用情况。

​		STP的使用信息和配置变更信息会被封装进Eiffel Message发送到消息队列中，SUV将负责从消息队列中接收特定的消息并使用其中的信息做计算，最后将数据持久化到数据库中，第三方UI将查询这些数据并展示。

## 二、技术栈

​		使用Spring Boot & Cloud实现的微服务架构，Maven做依赖管理，Consul做服务注册和发现，Mysql做数据存储，Redis做缓存实现，RabbitMQ作为系统的输入通道，使用Docker做服务的部署。

## 三、模块

​		SUV主要分为以下四个功能模块，每个模块对应一个微服务：

* 原始数据收集模块：data-collector service，负责从RabbitMQ中收集STP使用状态和测试用例执行状态的Eiffel Message，并将原始数据保存入库。
* 数据计算模块：data-processor service，从数据库中读取原始数据并计算，并将计算后的数据保存入库。
* STP信息管理模块：stp-manager service，负责保存和管理STP的基本信息，并负责从RabbitMQ中收集STP配置变更的Eiffel Message。
* 消息恢复模块：data-recovery service，查找由于异常丢失的Eiffel Message并发送给数据收集模块进行恢复。



## X、问题

1）明确一下每个计算项目都是怎么计算的。

2）



