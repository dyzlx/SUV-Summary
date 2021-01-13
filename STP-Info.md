# STP-Info模块总结

## 一、数据库表结构

![alt data-collector-db](./image/stp-info-db.png)

## 二、该模块中相关的Eiffel Message

​		Stp-Info模块将从Message Bus中收集2种Stp配置变更相关的Eiffel Message，分别为StpCapabilityAddedMessage和StpCapabilityDeletedMessage。Stp-Info service收到这些Message后，将及时的更新缓存和数据库中对应的值，确保data-processed服务计算的正确性。

### 1. StpCapabilityAddedMessage

​		该类型的Message表示某个Stp增加了一个capability，其中的关键字段包括StpName、CapabilityKey以及CapabilityValue。

### 2. StpCapabilityDeletedMessage

​		该类型的Message表示某个Stp删除了一个capability，其中的关键字段包括StpName和CapabilityKey。

## 三、缓存的实现

## 四、数据处理流程

![alt data-collector-db](./image/stp-info-flow.png)

## 五、该模块中解决的主要问题



