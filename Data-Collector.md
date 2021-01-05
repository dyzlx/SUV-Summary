# Data-Collector模块总结

## 一、数据库表结构

![alt data-collector-db](./image/data-collector-db.png)

## 二、该模块相关的Eiffel Message

​		data-collector服务将从RabbitMQ中收集5种Eiffel Message，分别为Job相关的三种：JobQueueEvent、JobStartedEvent和JobFinishedEvent，以及Testcase相关的两种：TestcaseStartedEvent、TestcaseFinishedEvent。

### 1、JobQueueEvent

​		表示一个Job已经加入了某个STP pool的等待队列中，等待空闲的STP被分配出来用以执行该任务。该message中的关键字段有：domainId表示该message的发送方、eventType表示event类型、eventTime表示该event的发生时间、jobExecutionId表示该job的唯一标识、jobId也表示该job的唯一标识。值得注意的是，该类message中没有STP的名字，因为等待阶段该job的STP还没有分配。

### 2、JobStartedEvent

​		表示一个Job已经被分配了STP并开始执行，该message中的关键字段有：domainId、eventType、eventTime、jobExecutionId、jobId、stp表示执行该job的STP的名字。Job执行过程中，除了最开始的install UP环节外，大部分时间都是在串行执行多个Testcase。

### 3、TestcaseStartedEvent

​		表示一个testcase开始执行，该message中的关键字段除了domainId、eventType、eventTime、jobExecutionId、jobId、stp等字段外，还有testCaseExecutionId是该testcase的唯一标识。

### 4、TestcaseFinishedEvent

​		表示一个testcase执行结束，该message中的关键字段为domainId、eventType、eventTime、jobExecutionId、jobId、stp、testCaseExecutionId，除了这些还有resultCode表示该testcase的执行结果是成功还是失败。

### 5、JobFinishedEvent

​		表示一个Job已经执行完毕，该message中的关键字段包括domainId、eventType、eventTime、jobExecutionId、jobId、stp等。

### 6、Message Flow

​		如图所示是一个完整的Job执行流程，该Job执行期间运行了三个testcase。图上涉及的9个event中的domainId、jobExecutionId、jobId、stp应该都是相同的。每一对testcase的event中testCaseExecutionId应该是相同的。每一个event中还有一个inputEvent字段，其值为该message上游message的eventId的值，该字段的作用是：当我们拿到某个message，便可以通过inputEvent字段找到该message所处job工作流的上游全部message。

![alt eiffel-flow](./image/eiffel-flow.png)

## 三、 数据处理流程

### 1、当接收到JobQueueEvent时

- Job表插入一条新的记录，写入job_id，job_execution_id，并将该Event中的eventTime做为job_start_time。或者根据jobId/jobExecutionId查询已经存在的纪录，并更新以上字段。（有可能是data-collector先收到了该Job其他的event，所以jobId/jobExecutionId对应的纪录已经存在）

- JobActivity表插入一条job_activity_type为job_queue的新纪录，写入job_id，job_execution_id，并将该Event中的eventTime字段的值做为job_activity_start_time。或者根据jobId/jobExecutionId查询job_activity_type为job_queue的纪录，并更新以上字段。(有可能是data-collector先收到了该Job的JobStartedEvent，所以该jobId或者jobExecutionId对应的job_activity_type为job_queue的纪录已经存在）
- JobActivityEvent表中插入一条新的记录，写入event_id和job_activity_id字段。

- Event表中插入一条新的纪录，写入相关字段。

### 2、当接收到JobStartedEvent时

- Job表插入一条新的纪录，写入job_id，job_execution_id。或者根据jobId/jobExecutionId查询已经存在的纪录，并更新以上字段。

- JobActivity分别表插入两条job_activity_type为job_queue和job_execution的新纪录。写入job_id，job_execution_id，并将该Event中的eventTime字段的值分别做为job_activity_type为job_queue纪录的job_activity_end_time以及job_activity_type为job_execution纪录的job_activity_start_time。或者根据jobId/jobExecutionId分别查询job_activity_type为job_queue和job_execution的纪录，并更新以上字段。
- JobActivityEvent表中插入一条新的记录，写入event_id和job_activity_id字段。
- Event表中插入一条新的纪录，写入相关字段。

### 3、当接收到TestcaseStartedEvent时

- Job表插入一条新的纪录，写入job_id，job_execution_id。或者根据jobId/jobExecutionId查询已经存在的纪录，并更新以上字段（该Job的JobQueueEvent或者JobStartedEvent已经收到，该job在Job表中的记录已经存在）。
- JobActivity表插入一条job_activity_type为testcase_execution的新纪录，写入job_id，job_execution_id，并将该Event中的eventTime字段的值做为job_activity_start_time。或者根据jobId/jobExecutionId查询job_activity_type为job_execution的纪录，并更新以上字段。(有可能是data-collector先收到了TestcaseFinishedEvent，所以该记录已经存在）注意，对于同一个testcase_execution activity可能存在多个test case，所以job_activity_start_time要是用多个TestcaseStartedEvent中最早的eventTime。
- TestCase表中插入一条新的纪录，写入testcase_execution_id、activity_id、job_execution_id、job_id、testcase_start_time、testcard_name字段。或者使用testcase_execution_id查询已经存在的纪录，更新testcase_start_time的值。
- TestcaseEvent中插入一条新的纪录，写入event_id和testcase_execution_id。

- Event表中插入一条新的纪录，写入相关字段。

### 4、当接收到TestcaseFinishdEvent时

- Job表插入一条新的纪录，写入job_id，job_execution_id。或者根据jobId/jobExecutionId查询已经存在的纪录，并更新以上字段。
- JobActivity表插入一条job_activity_type为testcase_execution的新纪录，写入job_id，job_execution_id，并将该Event中的eventTime字段的值做为job_activity_end_time。或者根据jobId/jobExecutionId查询job_activity_type为job_execution的纪录，并更新以上字段。(有可能是data-collector先收到了TestcaseStartedEvent，所以该记录已经存在）注意，对于同一个testcase_execution activity可能存在多个test case，所以job_activity_end_time要是用多个TestcaseStartedEvent中最晚的eventTime。
- TestCase表中插入一条新的纪录，写入testcase_execution_id、activity_id、job_execution_id、job_id、testcase_end_time、testcase_name以及testcase_result字段。或者使用testcase_execution_id查询已经存在的纪录，更新testcase_end_time的值。
- TestcaseEvent中插入一条新的纪录，写入event_id和testcase_execution_id。

- Event表中插入一条新的纪录，写入相关字段。

### 5、当接收到JobFinishdedEvent时

- Job表插入一条新的纪录，写入job_id，job_execution_id。或者根据jobId/jobExecutionId查询已经存在的纪录，并更新以上字段。

- JobActivity表插入一条job_activity_type为job_execution的新纪录，写入job_id，job_execution_id，并将该Event中的eventTime字段的值做为job_activity_end_time。或者根据jobId/jobExecutionId查询job_activity_type为job_execution的纪录，并更新以上字段。(有可能是data-collector先收到了该Job其他的event，所以jobId/jobExecutionId对应的纪录已经存在）
- JobActivityEvent表中插入一条新的记录，写入event_id和job_activity_id字段。
- Event表中插入一条新的纪录，写入相关字段。

## 四、该模块中解决的关键问题

#### 1、多条Message并发更新同一条记录

​		Job表和Testcase表的主键不是自增主键，Job表为JobId，Tsetcase表为TestcaseExecutionId，表中每一条记录的start_time和end_time两个字段分别是两条Eiffel Message负责更新的，例如对于一个Job表的记录，其start_time是由JobQueueEvent更新，end_time是由JobFinishedEvent更新。更新该条记录的伪代码如下：

```java
Job job = db.queryByJobId(jobId);
if (job == null) {
	job = new Job();
}
job.setJobId(event.getJobId());
if (event.type == "JobQueueEvent") {
	job.setStartTime(event.getTime());
}
if (event.type == "JobFinishedEvent") {
	job.setEndTime(event.getTime());
}
db.save(job);
```

可以看到如果Job表中已经存在对应JobId的记录，那么就获取该记录然后根据event的类型修改对应的startTime或者endTime。如果不存在该记录就创建一个新的Job对象然后填写属性。正常情况下，当JobQueueEvent和JobFinishedEvent依次被处理后，Job表中就会存在一条记录，其startTime和endTime分别为两个event中的eventTime字段的值。

​		当JobQueueEvent和JobFinishedEvent几乎同时接收到，也就是两个线程（A和B，A处理JobQueueEvent，B处理JobFinishedEvent）同时走到该代码处，两个线程同时查询DB不存在对应JobId的记录，就同时创建了两个新的对象（objectA和objectB，其中objectA的endTime为null，objectB的startTime为null）。然后几乎同时执行db.save()函数。

​		我们在代码中使用的Spring Data JPA，其save()函数源代码如下，persist()方法直接执行insert，merge()函数会先select，然后执行insert或者update。Job表中JobId为主键，objectA和objectB都会有jobId的值，所以isNew()方法会返回false，所以执行的是merge()方法。

```java
public <S extends T> S save(S entity) {
	if (entityInformation.isNew(entity)) {
		em.persist(entity);
		return entity;
	} else {
		return em.merge(entity);
	}
}
```

​		这里会产生两种情况的问题：

​		第一种情况，A线程执行save()函数，merge()方法通过select判断不存在该JobId的记录，执行insert语句，在A线程的记录还没有写入数据库之前，B线程也执行了save()函数，并判断不存在该JobId的记录，也执行insert语句。综上，A和B线程最终都执行了insert语句，后执行的insert会报主键冲突，执行失败。

​		第二种情况，A线程执行稍快，merge()方法通过select判断不存在该JobId的记录，执行insert语句，在A线程的记录成功写入数据库后，B线程才执行到merge()方法，通过select判断记录存在，于是执行了update语句。因为objectB中不携带startTime，所以这个update就将A线程写入的startTime更新为NULL。

​		综合两种情况的解决方案如下：

​		首先，让实体类Job实现JPA的Persistable接口，覆盖其中的isNew()方法，这样save()方法中的entityInfomation就不会使用默认的JpaMetamodelEntityInformation实现，而是使用JpaPersistableEntityInformation实现类（SimpleJpaRepository类的构造函数中会做上述的判断），JpaPersistableEntityInformation的isNew()方法就会调用Persistable实现类的isNew()方法，即我们覆写的实体类的isNew()方法。

```java
public class Job implements java.io.Serializable, Persistable<Integer> {
    @Transient //不需要序列化的字段
    @Setter
    private boolean newFlag;
    
    @Override
    public boolean isNew() {
        return newFlag;
    }
    ...
}
```

​		然后，在我们的代码中当查询不存在对应JobId的Job记录，创建新的Job对象后，将该对象的isNew的flag设置为true。最后，在整个service方法上加上@Retry注解，在抛出主键冲突异常后重试。

```java
@Retryable(include = DuplicateEntryExecption)
public void service() {
	Job job = db.queryByJobId(jobId);
	if (job == null) {
		job = new Job();
		job.setNewFlag(true);
	}
	...
	db.save(job);
}
```

​		如此，A和B线程同时查询记录不存在并创建新对象后都会执行insert方法，后者主键冲突后重试整个方法，重新查询Job对象存在，就会正常更新。

​		