# Data-processor模块总结

## 一、数据库表结构

## 二、模块简述

​		该模块的主要功能是通过访问Data-collector和Stp-manager的API获取Job&Testcase的执行情况，以及Stp的capability信息。使用这些信息来分别计算某个STP每30分钟的：TestCase通过率、活动时间（Activity Time，包括JobExecution活动时间、JobQueued活动时间、TestCaseExecution活动时间）、STP在这段时间的使用率以及这段时间中Job的平均等待数；

​		这些计算结果会写入数据库，Tableau作为SUV系统的数据展示界面，将使用这些数据来绘制图表。

## 三、数据处理流程

### 1. TestCase通过率

​		通过testcase表中的testcase_start_time和testcase_end_time字段联合查询表testcase、stp_job、stp，获得该时间段中的全部testcase的执行情况。然后根据stp分组，在每一组中根据testcase_result字段计算该时间段中该stp的testcase通过率，然后将结果写入testcase_pass_rate_result表中。

### 2. ActivityTime（活跃时间）

​		这里活跃时间分为三个部分：针对每一个STP，JobExecution阶段在给定时间段中的活跃时间、JobQueue阶段在给定时间段中的活跃时间、TestcaseExecution阶段在给定时间段中的活跃时间。

#### a) JobExecution 活跃时间

​		通过job_activity表的job_activity_start_time和job_activity_end_time字段以及activity_name="job_execution"字段联合查询表job_activity、stp_job、stp_capability、stp表。获取这些数据后根据stp分组，分别遍历每一组中的数据，计算每一项数据的job_activity_end_time减job_activity_start_time的值，并将这一组中的全部差值相加，就得到该stp该时间段中JobExecution的活跃时间。

#### b) JobQueue 活跃时间

​    	通过job_activity表的job_activity_start_time和job_activity_end_time字段以及activity_name="job_queue"字段联合查询表job_activity、stp_job、stp_capability、stp表。获取这些数据后根据stp分组，分别遍历每一组中的数据，计算每一项数据的job_activity_end_time减job_activity_start_time的值，并将这一组中的全部差值相加，就得到该stp该时间段中JobQueue的活跃时间。

#### c) TestcaseExecution 活跃时间

​		通过job_activity表的job_activity_start_time和job_activity_end_time字段以及activity_name="testcase_execution"字段联合查询表job_activity、stp_job、stp_capability、stp表。获取这些数据后根据stp分组，分别遍历每一组中的数据，计算每一项数据的job_activity_end_time减job_activity_start_time的值，并将这一组中的全部差值相加，就得到该stp该时间段中TestcaseExecution的活跃时间。	

### 3. Stp的使用率

​		STP的使用率就是JobExecution活跃时间占时间段的比例。	

## 四、定时任务的实现

### 1. Realtime定时任务

### 2. Recovery定时任务

### 3. Aggregator定时任务

### 4. 多实例部署下定时任务的调度

## 五、该模块中解决的主要问题

### 1. 活跃时间计算中缺少startTime或者endTime

​		在Realtime定时任务计算当前时间段的活跃时间时，可能存在某个Job的活跃时间缺少start_time或者缺少end_time，导致无法正确求得差值，其原因可能是对应的Message还没有收到或者丢失，其解决方法就是如下：

​		如果JobExecution活跃时间的计算中缺少end_time，即还没有收到JobFinishedMessage，则使用当前STP下一个Job的JobQueue Activity的start_time替代，如果该start_time已经超过了当前时间区间，则使用当前时间区间的端点值替代。并且将这个异常计入数据库表job_activity_no_data。

​		如果JobQueue活跃时间的计算中缺少end_time（相当于JobExecution缺少start_time），即还没有收到JobStartedMessage，则使用当前STP当前Job的JobExecution Activity的end_time替代，如果是JobExecution活跃时间的计算，就使用当前STP当前Job的JobQueue Activity的start_time替代。如果该替代值已经超过了当前时间区间，则使用当前时间区间的端点值替代。并且将这个异常计入数据库表job_activity_no_data。

​		如果JobQueue活跃时间的计算中缺少start_time，即还没有收到JobQueueMessage，

​		如果TestcaseExecution活跃时间计算中缺少start_time或者end_time，就用相邻testcase的end_time或者start_time替代，替代的时间超过时间区间的端点，就用端点的值替代。并将这个异常计入数据表job_activity_no_data。

​		

​		

### 2. 活跃时间计算中发生两个Job时间段重叠

### 3. Stp的capability缓存变化对计算结果的影响





