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
* STP信息管理模块：stp-manager，负责从外部接口查询STP的详细信息，保存入库并缓存，以及从RabbitMQ中收集stpconfig相关的Eiffel Message用来更新数据库和缓存。
* 数据计算模块：data-processor service，从数据库中读取原始数据并计算，并将计算后的数据保存入库。
* 消息恢复模块：data-recovery service，查找由于异常丢失的Eiffel Message并发送给数据收集模块进行恢复。

## 四、原始数据收集模块（data-collector service）简述

### 1. 表结构

![alt data-collector-db](image/data-collector-db.png)

### 2. Job&Testcase相关的Eiffel message

​		data-collector服务将从RabbitMQ中收集5种Eiffel Message，分别为Job相关的三种：JobQueueEvent、JobStartedEvent和JobFinishedEvent，以及Testcase相关的两种：TestcaseStartedEvent、TestcaseFinishedEvent。

​		JobQueueEvent表示一个Job已经加入了某个STP pool的等待队列中，等待空闲的STP被分配出来用以执行该任务。该message中的关键字段有：domainId表示该message的发送方、eventType表示event类型、eventTime表示该event的发生时间、jobExecutionId表示该job的唯一标识、jobId也表示该job的唯一标识。值得注意的是，该类message中没有STP的名字，因为等待阶段该job的STP还没有分配。

​		JobStartedEvent表示一个Job已经被分配了STP并开始执行，该message中的关键字段有：domainId、eventType、eventTime、jobExecutionId、jobId、stp表示执行该job的STP的名字。Job执行过程中，除了最开始的install UP环节外，大部分时间都是在串行执行多个Testcase。

​		TestcaseStartedEvent表示一个testcase开始执行，该message中的关键字段除了domainId、eventType、eventTime、jobExecutionId、jobId、stp等字段外，还有testCaseExecutionId是该testcase的唯一标识。

​		TestcaseFinishedEvent表示一个testcase执行结束，该message中的关键字段为domainId、eventType、eventTime、jobExecutionId、jobId、stp、testCaseExecutionId，除了这些还有resultCode表示该testcase的执行结果是成功还是失败。

![alt eiffel-flow](image/eiffel-flow.png)

​		如上图所示是一个完整的Job执行流程，该Job执行期间运行了三个testcase。图上涉及的9个event中的domainId、jobExecutionId、jobId、stp应该都是相同的。每一对testcase的event中testCaseExecutionId应该是相同的。每一个event中还有一个inputEvent字段，其值为该message上游message的eventId的值，该字段的作用是：当我们拿到某个message，便可以通过inputEvent字段找到该message所处job工作流的上游全部message。

### 3. 数据处理流程





## X、问题



