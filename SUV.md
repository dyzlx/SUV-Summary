SUV 项目总结
===========

## 一、项目简述

​		SUV全称为STP Utilization Visualization（STP利用率可视化），STP是指爱立信硬件测试环境中的一套标准的测试组件。该项目的目标是将测试环境中的全部STP的利用率借助第三方的UI做可视化，这里STP的利用率主要包括：单位时间段内TGF STP的使用率、单位时间段内ERIS STP的使用率、单位时间段内STP的活跃时间、单位时间段内STP上执行的测试用例的通过率。用户（Tester）通过第三方UI将了解到某个STP在某个时间段下使用情况。

​		STP的使用信息和配置变更信息会被封装进Eiffel Message发送到消息队列中，SUV将负责从消息队列中接收特定的消息并使用其中的信息做计算，最后将数据持久化到数据库中，第三方UI将查询这些数据并展示。

## 二、技术栈

​		使用Spring Boot & Cloud实现的微服务架构，Maven做依赖管理，Consul做服务注册和发现，Mysql做数据存储，Redis做缓存实现，RabbitMQ作为系统的输入通道，使用Docker做服务的部署。

## 三、模块简述

​		SUV主要分为以下四个功能模块，每个模块对应一个微服务：

* 原始数据收集模块：data-collector service，负责从RabbitMQ中收集job和testcase相关的Eiffel Message，处理成原始数据并入库。
* STP信息管理模块：stp-manage，负责从外部接口（TGF）查询STP的详细信息，保存入库并缓存，以及从RabbitMQ中收集stpconfig相关的Eiffel Message用来更新数据库和缓存。
* 数据计算模块：data-processor service，从数据库中读取原始数据并计算，并将计算后的数据保存入库。
* 消息恢复模块：data-recovery service，查找由于异常丢失的Eiffel Message并发送给数据收集模块进行恢复。

## 四、系统结构图

![alt data-collector-db](./image/suv-system.png)

## 五、SUV中解决的主要问题

### 1. 数据量增大后，读数据缓慢，数据库索引的设置

### 2. 消息队列中消息处理时的一些问题

#### a) 预防消息丢失问题

​		首先，明确一点SUV系统整体上来说是作为一个消息队列的消费者，而TGF是作为消息的发送方，消息在正确进入我们设置的队列前丢失的问题，SUV是无法解决的，只能由TGF一方来保证消息被正确投递到rabbitmq server。

​		当消息已经被发送到rabbitmq server直到消息被SUV处理入库的过程中，可能存在以下两种消息丢失的场景：第一，Rabbitmq Server宕机，造成rabbitmq队列中的消息丢失；第二，SUV处理消息时出现异常，导致正在处理的消息没有处理完（写入数据库）而丢失。解决方案如下：

​		针对第一种情况，data-collector和stp-manage会将创建的队列设置持久化（消息本身在TGF发送端发送时已经设置为持久化，消费端只需要将队列设置为持久化），即使rabbitmq宕机，恢复后，队列和消息也不会丢失。

​		针对第二种情况，data-collector和stp-manage在消费消息时会使用手动确认模式，当消息被正确处理入库后，会发送ack到rabbitmq，如果消息处理过程中出现异常导致data-collector或者stp-manage崩溃，消息就不会被ack，没有ack的消息依旧会存在于rabbitmq的队列中处理unack状态，消费者下线后，该消息又会重新变为待消费状态，被发送给另一个服务实例或者留在队列中等待被消费；如果消息处理时发生其他的运行时异常，该消息也会被重新requeue，等待下次消费。

#### b) 消息重复处理问题

​		在SUV中导致消息重复消费的场景如下：第一种情况，TGF由于BUG发送了重复的消息，比如针对某个Job，发送了两次JobStartedMessage。第二种情况，使用消息手动确认机制后，虽然消息被正确消费，且发送了ack，但是由于网络原因，ack没有被rabbitmq接收，并且消费者宕机下线，这时，这个消息就会重新变成待消费状态。消费者恢复后，这条消息就会被重复处理。

​		SUV接收到的所有类型的消息，不会触发任何的事件或者任务，只是将消息中的信息更新到SUV的数据库，所以重复的消息不会造成业务上的问题，顶多只是将数据库中已有的数据再重复更新一遍。

### 3. 一次线程暴增问题的排查

