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
- 从该Event中获取stp字段，使用该值访问stp-cache模块，获取stp的详细信息。
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

### 1、多条Message并发更新同一条记录

#### a) Job表记录的startTime和endTime同时被JobQueueEvent和JobFinishedEvent更新产生的主键冲突问题和字段丢失问题

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

#### b) JobActivity表中testcase_execution类型的activity记录的startTime和endTime并发被多个testcase message更新造成的startTime和endTime不正确问题

​		JobActivity表中activity_type为testcase_execution的记录会在Testcase表中对应多个testcase记录，其startTime和endTime分别对应这些Testcase中的最早的startTime和最晚的endTime。

​		但是同一个Job的多个TestcaseStartEvent和FinishedEvent不会按照其执行的顺序到达，而是乱序到达的，所以当收到一个TestcaseStartedEvent时，首先要根据JobActivity表中对应的testcase_execution的记录的startTime做对比，如果该event中的startTime比现在表中的startTime还要早，就更新这个字段，否则就不更新，收到TestcaseFinishedEvent时同理，要选择最晚的endTime来更新。

​		问题在于，这多个Testcase对应的Message可能同时到达，数据库中就会出现多线程并发更新同一个字段的问题，在data-collector多实例部署时更会如此。最终就可能导致，JobActivity中记录的startTime并非最早的执行的testcase的startTime，记录的endTime也并非最晚执行的testcase的endTime。

​		解决方法就是在select对应的JobActivity记录时加锁，使用select ... where id=? for update语句，该语句会在RR和RC隔离级别下为搜索到的行加上排他的行锁（如果不使用唯一索引搜索就会在RR隔离级别下加Next-Key Lock），事务结束后会释放锁（每一个Eiffel Message的处理流程上加事务）。这样就保证了每次只会有一个线程来更新这个记录的startTime或者endTime。

### 2. 死锁问题

​		代码中事务的隔离级别使用的是RR隔离级别（Repeatable Read 可重复读），该隔离级别下存在Gap锁，即InnoDB的行锁行为实际上是Next-Key Lock（行锁+Gap锁），上面的问题中我们为了解决多个TestcaseEvent并发修改同一记录的同一字段的问题，在select语句上加了锁（select ... for update），这个锁在RR隔离级别下是Next-Key Lock，即存在Gap锁。

​		问题在于，当同一个Job的两个TestcaseStartedEvent同时被处理，两个线程A和B几乎同时在JobActivity表上执行select ... where testcase_execution_id=? for update语句，因为暂时不存在对应的testcase_execution类型的纪录，所以两个事务在数据库的同一位置加上了Gap Lock，然后A和B同时创建一个新的对象，先后执行insert语句，A事务发现该位置存在B事务的Gap锁，所以向该位置加一个插入意向锁，并处于等待状态，等待B事务的Gap锁释放；同样的，B事务发现A事务的Gap锁，于是也向该位置加一个插入意向锁，并处于等待状态，等待A事务的Gap锁释放。而两个事务的Gap锁只有在事务结束时才能释放，死锁发生了（如下表梳理）。

|                            事务A                             |                            事务B                             |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| select * from table where testcase_execution_id=a for update;<br>【唯一索引列检索不存在的纪录，该记录位置加Gap锁A，事务结束释放】 |                                                              |
|                                                              | select * from table where testcase_execution_id=a for update;<br>【唯一索引列检索不存在的纪录，该记录位置加Gap锁B，事务结束释放，（Gap锁之间不会冲突）】 |
| insert into table(id, ....) values(a, ....);<br>【该事务发现事务B在这个位置有Gap锁B，于是向该位置添加Insert Intention Lock，并等待事务B释放Gap锁B】 |                                                              |
|                                                              | insert into table(id, ....) values(a, ....);<br>【该事务发现事务A在这个位置有Gap锁A，于是向该位置添加Insert Intention Lock，并等待事务A释放Gap锁A】 |
|                           DeadLock                           |                           DeadLock                           |

​		综上，死锁的原因是由于Gap锁导致的，问题的解决方法就是将事务的隔离级别调整为RC隔离级别（Read Commit 读提交）。该隔离级别下不存在Gap锁，select ... for update在纪录不存在的情况下没有加锁。但是在隔离级别下的幻读问题如何处理？