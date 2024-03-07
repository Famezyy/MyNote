# 第02章_Spring Batch进阶

## 1.批处理数据表

如果选择数据库方式存储批处理数据，Spring Batch 在启动时会创建 9 张表，分别存储：`JobExecution`、`JobContext`、`JobParameters`、`JobInstance`、`JobExecution id 序列`、`Job id 序列`、`StepExecution`、`StepContext/ChunkContext`、`StepExecution id 序列` 等对象。Spring Batch 提供 `JobRepository` 组件来实现这些表的 CRUD 操作，并且这些操作基本上封装在步骤中。

### 1.1 batch_job_instance

当作业第一次执行时，会根据作业名、标识参数生成一个唯一 `JobInstance` 对象，`batch_job_instance` 表会记录一条信息代表这个作业实例。

```bash
+-----------------+---------+-------------------+----------------------------------+
| JOB_INSTANCE_ID | VERSION | JOB_NAME          | JOB_KEY                          |
+-----------------+---------+-------------------+----------------------------------+
|               1 |       0 | hello-job         | d41d8cd98f00b204e9800998ecf8427e |
+-----------------+---------+-------------------+----------------------------------+
```

| 字段            | 描述                                                |
| --------------- | --------------------------------------------------- |
| JOB_INSTANCE_ID | 作业实例主键                                        |
| VERSION         | 乐观锁控制的版本号                                  |
| JOB_NAME        | 作业名称                                            |
| JOB_KEY         | 作业名与标识性参数的哈希值，能唯一标识一个 job 实例 |

### 1.2 batch_job_execution

每次启动作业时，都会创建一个 `JobExecution` 对象，代表一次作业执行。

```bash
JOB_EXECUTION_ID: 1
         VERSION: 2
 JOB_INSTANCE_ID: 1
     CREATE_TIME: 2024-03-07 14:37:35.869895
      START_TIME: 2024-03-07 14:37:36.041894
        END_TIME: 2024-03-07 14:37:36.620659
          STATUS: COMPLETED
       EXIT_CODE: COMPLETED
    EXIT_MESSAGE:
    LAST_UPDATED: 2024-03-07 14:37:36.621659
```

| 字段             | 描述                   |
| ---------------- | ---------------------- |
| JOB_EXECUTION_ID | Job 执行对象主键       |
| VERSION          | 乐观锁控制的版本号     |
| JOB_INSTANCE_ID  | `JobInstantce` 的 ID   |
| CREATE_TIME      | 创建的时间             |
| START_TIME       | 执行开始时间           |
| END_TIME         | 执行结束时间           |
| STATUS           | 执行的批处理状态       |
| EXIT_CODE        | 执行的退出码           |
| EXIT_MESSAGE     | 执行的退出信息         |
| LAST_UPDATED     | 最后一次更新记录的时间 |

### 1.3 batch_job_execution_context

用于保存 `JobContext` 对应的 `ExecutionContext` 对象数据。

```bash
  JOB_EXECUTION_ID: 1
     SHORT_CONTEXT: rO0ABXNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAAAAx3CAAAABAAAAABdAANYmF0Y2gudmVyc2lvbnQABTUuMS4xeA==
SERIALIZED_CONTEXT: NULL
```

| 字段               | 描述                                  |
| ------------------ | ------------------------------------- |
| JOB_EXECUTION_ID   | Job 执行对象主键                      |
| SHORT_CONTEXT      | ExecutionContext 序列化后字符串缩减版 |
| SERIALIZED_CONTEXT | ExecutionContext 序列化后字符串       |

### 1.4 batch_job_execution_params

用于保存标识性参数。

```bash
JOB_EXECUTION_ID: 4
  PARAMETER_NAME: param
  PARAMETER_TYPE: java.lang.String
 PARAMETER_VALUE: 1
     IDENTIFYING: Y
```

| 字段             | 描述                       |
| ---------------- | -------------------------- |
| JOB_EXECUTION_ID | Job 执行对象主键           |
| PARAMETER_NAME   | 标记参数名称               |
| PARAMETER_TYPE   | 标记参数类型               |
| PARAMETER_VALUE  | 标记参数值                 |
| IDENTIFYING      | 标记该参数是否为标识性参数 |

### 1.5 batch_step_execution

保存每个步骤执行信息。

```bash
 STEP_EXECUTION_ID: 1                                   # 步骤执行对象 ID
           VERSION: 3 									# 乐观锁控制版本号
         STEP_NAME: step1								# 步骤名称
  JOB_EXECUTION_ID: 1									# 作业执行对象 ID
       CREATE_TIME: 2024-03-07 14:37:36.140893			# 步骤的创建时间
        START_TIME: 2024-03-07 14:37:36.287899			# 步骤执行的开始时间
          END_TIME: 2024-03-07 14:37:36.507660			# 步骤执行的结束时间
            STATUS: COMPLETED							# 步骤批处理状态
      COMMIT_COUNT: 1									# 步骤执行中提交的事务次数
        READ_COUNT: 0									# 读入的条目数量
      FILTER_COUNT: 0									# 由于 ItemProcessor 返回 null 而过滤掉的条目数
       WRITE_COUNT: 0									# 写入的条目数量
   READ_SKIP_COUNT: 0									# 由于 ItemReader 中抛出异常而跳过的条目数量
  WRITE_SKIP_COUNT: 0									# 由于 ItemWriter 中抛出异常而跳过的条目数量
PROCESS_SKIP_COUNT: 0									# 由于 ItemProcessor 中抛出异常而跳过的条目数量
    ROLLBACK_COUNT: 0									# 在步骤执行中被回滚的事务数量
         EXIT_CODE: COMPLETED							# 步骤的退出码
      EXIT_MESSAGE:										# 步骤执行的返回信息
      LAST_UPDATED: 2024-03-07 14:37:36.508658			# 最后一次更新记录时间
```

### 1.6 batch_step_execution_context

用于保存 `StepContext` 的 `ExecutionContext`。

```bash
 STEP_EXECUTION_ID: 1
     SHORT_CONTEXT: rO0ABXNyABFqYXZhLnV0aWwuSGFzaE1hcAUH2sHDFmDRAwACRgAKbG9hZEZhY3RvckkACXRocmVzaG9sZHhwP0AAAAAAAAx3CAAAABAAAAADdAARYmF0Y2gudGFza2xldFR5cGV0ABxjb20uZXhhbXBsZS5oZWxsby5IZWxsb0pvYiQxdAANYmF0Y2gudmVyc2lvbnQABTUuMS4xdAAOYmF0Y2guc3RlcFR5cGV0ADdvcmcuc3ByaW5nZnJhbWV3b3JrLmJhdGNoLmNvcmUuc3RlcC50YXNrbGV0LlRhc2tsZXRTdGVweA==
SERIALIZED_CONTEXT: NULL
```

| 字段               | 描述                                  |
| ------------------ | ------------------------------------- |
| STEP_EXECUTION_ID  | Step 执行对象主键                     |
| SHORT_CONTEXT      | ExecutionContext 序列化后字符串缩减版 |
| SERIALIZED_CONTEXT | ExecutionContext 序列化后字符串       |

### 1.7 H2内存型数据库

Spring Batch 也支持内存型数据库例如 H2，当批处理结束后数据会被清除。

## 2.作业运行

作业的运行是指对作业的控制，包括作业启动、作业停止、作业异常处理、作业重启处理等。

### 2.1 作业启动

#### 1.SpringBoot启动

核心类为：`JobLauncherApplicationRunner`，在 SpringBoot 启动后会调用该类的 `run()` 方法，然后委托给 `SimpleJobLauncher` 的 `run()` 方法。

#### 2.单元测试启动

在测试类中需要注入 `JobLauncher` 后，在测试方法中手动执行 `run()` 方法传入 `Job` 和 `JobParameters` 对象。

#### 3.RESTful API启动

**（1）禁止启动执行**

默认情况下 SpringBoot 已启动马上执行作业，如果想禁止，则使用如下配置：

```yaml
spring:
	batch:
		job:
			enabled: false
```

**（2）引入 web 依赖**

**（3）创建配置类注入 Tasklet、Step 和 Job**

**（4）在 `Controller` 中注入 `JobLauncher` 调用方法**

```java
@RestController
public class HelloController {

    @Autowired
    JobLauncher jobLauncher;

    @Autowired
    Job job;

    @GetMapping("/hello")
    @SneakyThrows
    public ExitStatus hello(String name) {
        JobParameters parameters = new JobParametersBuilder()
            .addString("name", name)
            .toJobParameters();
        return jobLauncher.run(job, parameters).getExitStatus();
    }
}
```

但此时如果传入的参数值相同，则执行 `Job` 任务时会出错。需要注意的是，使用 `Incrementer` 时需要先获取之前的 `JobParameters` 进行增量操作，否则不会执行 `Job` 的 `incrementer()` 的逻辑。

```java
@RestController
public class HelloController {

    @Autowired
    JobLauncher jobLauncher;

    @Autowired
    Job job;

    // 注入 jobExplorer
    @Autowired
    JobExplorer jobExplorer;

    @GetMapping("/hello")
    @SneakyThrows
    public ExitStatus hello(String name) {
        JobParameters parameters = new JobParametersBuilder(jobExplorer)
            // 获取下一个 JobParameters
            .getNextJobParameters(job)
            .addString("name", name)
            .toJobParameters();
        return jobLauncher.run(job, parameters).getExitStatus();
    }
}
```

此时就可以传入同一个参数了。

### 2.1 作业停止

- 自然结束：正常执行完成后返回 `COMPLETED`
- 异常结束：发生任何异常后返回 `FAILED`
- 手动结束：手动结束后返回 `STOPPED`

接下来模拟一个手动结束的场景：设计两个步骤，step1 用于叠加 readCount 模拟从数据库中读取资源，step2 用于执行逻辑：当 totalCount = readCount 时正常结束，不等式不执行 step2，直接手动结束。

有两种实现方式：

#### 1.Step监听器方式

监听器

```java
@Component
public class StopStepListener implements StepExecutionListener {
    @Override
    public void beforeStep(StepExecution stepExecution) {
        StepExecutionListener.super.beforeStep(stepExecution);
    }

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        final ExecutionContext executionContext = stepExecution.getJobExecution().getExecutionContext();
        final int readCount = (int) executionContext.get("readCount");
        final int totalCount = (int) executionContext.get("totalCount");
        if (readCount != totalCount) return ExitStatus.STOPPED;
        return stepExecution.getExitStatus();
    }
}
```

任务

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public Tasklet tasklet1() {
        return (contribution, chunkContext) -> {
            System.out.println("---------- tasklet1 ----------");
            final ExecutionContext executionContext = chunkContext.getStepContext().getStepExecution().getJobExecution().getExecutionContext();
            executionContext.put("readCount", 50);
            executionContext.put("totalCount", 100);
            return RepeatStatus.FINISHED;
        };
    }

    @Bean
    public Tasklet tasklet2() {
        return (contribution, chunkContext) -> {
            System.out.println("---------- tasklet2 ----------");
            return RepeatStatus.FINISHED;
        };
    }

    @Bean
    public Step step1(StepExecutionListener stopStepListener, JobRepository jobRepository, PlatformTransactionManager transactionManager) {
        return new StepBuilder("step1", jobRepository)
            .tasklet(tasklet1(), transactionManager)
            .listener(stopStepListener)
            // 当 status 为 complete 时，即使同一 Job 也允许从 step1 重新执行
            .allowStartIfComplete(true)
            .build();
    }

    @Bean
    public Step step2(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
        return new StepBuilder("step2", jobRepository)
            .tasklet(tasklet2(), transactionManager)
            .build();
    }


    @Bean
    public Job job(JobRepository jobRepository, Step step1, Step step2) {
        return new JobBuilder("manual-stopped-job", jobRepository)
            .start(step1)
            // 当 step1 返回 STOPPED 则手动结束任务，设置状态为 STOPPED，并设置后续重启从 step1 开始
            .on("STOPPED").stopAndRestart(step1)
            .from(step1).on("*").to(step2)
            .end()
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

观察发现第一次运行时只执行了 tasklet1，并且表中插入了一条记录显示 `EXIST_CODE` 为 `STOPPED`。

修改 readCount 为 100 后再次执行发现成功执行了 tasklet1 和 tasklet2，此时插入了一条新的记录退出码为 `COMPLETED`。

#### 2.停止标记方式

### 2.3 作业重启

## 3.ItemReader

## 4.ItemProcessor

## 5.ItemWriter