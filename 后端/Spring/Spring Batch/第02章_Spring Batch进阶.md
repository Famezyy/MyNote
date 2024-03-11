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

接下来模拟一个手动结束的场景：设计两个步骤，step1 用于叠加 readCount 模拟从数据库中读取资源，step2 用于执行逻辑：当 totalCount = 100 时正常结束，不等式不执行 step2，直接手动结束。

有两种实现方式：

#### 1.Step监听器方式

**监听器**

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
        if (readCount != 100) return ExitStatus.STOPPED;
        return stepExecution.getExitStatus();
    }
}
```

**任务**

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public Tasklet tasklet1() {
        return (contribution, chunkContext) -> {
            System.out.println("---------- tasklet1 ----------");
            final ExecutionContext executionContext = chunkContext.getStepContext().getStepExecution().getJobExecution().getExecutionContext();
            executionContext.put("readCount", 50);
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
            // 此时 step1 的 status 是 COMPLETED，因此需要开启 allowStartIfComplete
            // 否则再次执行时会无限循环
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

这种方式第一次 STOP 后的 `EXIST_CODE` 也是 `STOPPED`。

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public Tasklet tasklet1() {
        return (contribution, chunkContext) -> {
            System.out.println("---------- tasklet1 ----------");
            int readCount = 50;
            if (readCount != 100) {
                // 设置停止标记
                // // 此时 step1 的 status 是 STOPPED，因此不需要开启 allowStartIfComplete
                chunkContext.getStepContext().getStepExecution().setTerminateOnly();
            }
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
        return new JobBuilder("manual-stopped-job-2", jobRepository)
			// 正常设置步骤即可
            .start(step1)
            .next(step2)
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

### 2.3 作业步骤重启

表示允许作业步骤重新执行，默认情况下，只允许异常或终止状态的步骤重启，但有时存在特殊场景，要求需要其他状态步骤重启。Spring Batch 提供 3 种操作来控制重启。

#### 1.禁止作业重启

执行失败后不允许重复执行。

在 `Job` 中使用 `preventRestart()` 禁止重启。

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public Tasklet tasklet1() {
        return (contribution, chunkContext) -> {
            System.out.println("---------- tasklet1 ----------");
            // 设置停止标记
            chunkContext.getStepContext().getStepExecution().setTerminateOnly();
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
        return new JobBuilder("restart-job-1", jobRepository)
            // 默认情况下执行失败后可以再从 step1 开始执行，开启 preventRestart 后再次执行会报错
            .preventRestart()
            .start(step1)
            .next(step2)
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

#### 2.限制步骤执行次数

在 `Step` 中使用 `startLimit(limit)` 设置执行次数。

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public Tasklet tasklet1() {
        return (contribution, chunkContext) -> {
            System.out.println("---------- tasklet1 ----------");
            // 设置停止标记
            chunkContext.getStepContext().getStepExecution().setTerminateOnly();
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
                // 设置只能执行 2 次，重起执行 1 次
                .startLimit(2)
                .tasklet(tasklet1(), transactionManager)
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
        return new JobBuilder("restart-limit-job-1", jobRepository)
                .start(step1)
                .next(step2)
                .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

#### 3.无限重启步骤

Spring Batch 限制同 Job 名和标识参数作业只能成功执行一次，但是对于步骤可以通过 `allowStartIfComplete(true)` 开启对 `COMPLETED` 状态步骤的无限执行。

默认情况下下面的 `Job` 只能成功执行一次，再次执行时会显示 `Step already complete or not restartable, so no action to execute` 而不执行步骤。

开启 `allowStartIfComplete(true)` 后发现再次执行时 tasklet1 成功执行了。

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public Tasklet tasklet1() {
        return (contribution, chunkContext) -> {
            System.out.println("---------- tasklet1 ----------");
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
			// 开启重复执行
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
        return new JobBuilder("step-unlimit-job", jobRepository)
            .start(step1)
            .next(step2)
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

## 3.ItemReader

`ItemRaeder` 是 Spring Batch 提供的输入组件，规范接口是 `ItemReader<T>`，里面有个 `read()` 方法，用来定义输入逻辑。

```java
@FunctionalInterface
public interface ItemReader<T> {
    @Nullable
    T read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException;
}
```

Spring Batch 根据常用的输入类型，提供了许多默认的实现，包括：平面文件、数据库、JMS 资源和其他输入源等。

### 3.1 读取平面文件

平面文件一般指简单行/多行结构的纯文本文件，比如记事本记录文件。与 XML 这种区别在于没有结构，没有标签的限制。Spring Batch 默认使用 `FlatFileItemReader` 实现平面文件的输入。

#### 1.delimited

字符串截取。

例如对于下面的文件：

```bash
1#zhang3#15
2#li4#13
3#wang5#18
```

创建一个实体类：

```java
@Data
public class User {
    private Long id;
    private String name;
    private int age;
}
```

创建任务：

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public ItemWriter<User> itemWriter() {
        // 打印各个对象
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    @Bean
    public FlatFileItemReader<User> flatFileItemReader() {
        return new FlatFileItemReaderBuilder<User>()
            .name("userItemReader")
            // 指定来源
            .resource(new ClassPathResource("user.txt"))
            // 指定解析器，默认使用 , 分割，这里使用 # 分割，并对各个项目进行命名
            .delimited().delimiter("#").names("id", "name", "age")
            // 将读取的数据封装为 User 类型
            .targetType(User.class)
            .build();
    }

    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
            // 使用 chunk，指定加载 1 次
            .<User, User>chunk(1, manager)
            .reader(flatFileItemReader())
            .writer(itemWriter())
            .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("read-source-job", jobRepository)
            .start(step)
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

#### 2.FieldSetMapper

如果数据源和目标对象的字段不是一一对应的，则可以使用 `FieldSetMapper` 进行字段映射处理。

```java
public interface FieldSetMapper<T> {
    T mapFieldSet(FieldSet fieldSet) throws BindException;
}
```

例如对于下面的文件：

```bash
1#zhang3#15#广东#广州#天河区
2#li4#13#四川#成都#武侯区
3#wang5#18#广西#贵林#雁山区
```

用户对象：

```java
@Data
public class User {
    private Long id;
    private String name;
    private int age;
    private String address;
}
```

创建映射器：

```java
public class UserFieldMapper implements FieldSetMapper<User> {
    @Override
    public User mapFieldSet(FieldSet fieldSet) throws BindException {
        User user = new User();
        user.setId(fieldSet.readLong("id"));
        user.setName(fieldSet.readString("name"));
        user.setAge(fieldSet.readInt("age"));
        user.setAddress(fieldSet.readString("province") + fieldSet.readString("city") + fieldSet.readString("area"));
        return user;
    }
}
```

创建任务：

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public ItemWriter<User> itemWriter() {
        // 打印各个对象
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    @Bean
    public FlatFileItemReader<User> flatFileItemReader() {
        return new FlatFileItemReaderBuilder<User>()
                .name("userItemReader")
                // 指定来源
                .resource(new ClassPathResource("user.txt"))
                // 指定解析器，默认使用 , 分割，这里使用 # 分割，并对各个项目进行命名
                .delimited().delimiter("#").names("id", "name", "age", "province", "city", "area")
                // 使用自定义的 FieldSetMapper 封装对象
                .fieldSetMapper(new UserFieldMapper())
                .build();
    }

    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
                .<User, User>chunk(1, manager)
                .reader(flatFileItemReader())
                .writer(itemWriter())
                .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("read-source-job", jobRepository)
                .start(step)
                .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

### 3.2 读取JSON文件

Spring Batch 提供了专门操作 JSON 文档的 API：`JsonItemReader`。

对于下面的 JSON 文件：

```json
[
  {"id": 1, "name": "zhang3", "age":  15},
  {"id": 1, "name": "li4", "age":  5},
  {"id": 1, "name": "wang5", "age":  11}
]
```

创建任务：

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public ItemWriter<User> itemWriter() {
        // 打印各个对象
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    @Bean
    public JsonItemReader<User> jsonItemReader() {
        // 创建 jsonObjectReader
        JacksonJsonObjectReader<User> jsonObjectReader = new JacksonJsonObjectReader<>(User.class);
        // 设置 Mapper
        jsonObjectReader.setMapper(new ObjectMapper());
        return new JsonItemReaderBuilder<User>()
                .name("userJsonItemReader")
                .resource(new ClassPathResource("user.json"))
                .jsonObjectReader(jsonObjectReader)
                .build();
    }

    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
                .<User, User>chunk(1, manager)
                .reader(jsonItemReader())
                .writer(itemWriter())
                .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("json-source-job", jobRepository)
                .start(step)
                .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

### 3.3 读取数据库

 对于下面的数据库结构：

```sql
CREATE TABLE `user` (
	`id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键',
	`name` VARCHAR(255) DEFAULT NULL COMMENT '用户名',
	`age` INT DEFAULT NULL COMMENT '年龄',
	PRIMARY KEY (`id`)
) ENGINE=INNODB AUTO_INCREMENT=6 DEFAULT CHARSET=utf8mb3;
```

```sql
INSERT INTO `user` VALUES (1, 'zhang3', 11);
INSERT INTO `user` VALUES (2, 'li4', 12);
INSERT INTO `user` VALUES (3, 'wang5', 13);
```

Spring Batch 提供 2 种读取方式。

#### 1.游标方式

游标遍历时，获取数据表中某一条数据，如果使用 JDBC 操作，游标指向的那条数据会被封装到 `ResultSet` 中，之后使用 `RowMapper` 实现表数据与实体对象的映射。

实体类

```java
@Data
public class User {
    private Long id;
    private String name;
    private int age;
}
```

`RowMapper`

```java
public class UserRowMapper implements RowMapper<User> {
    @Override
    public User mapRow(ResultSet rs, int rowNum) throws SQLException {
        User user = new User();
        user.setId(rs.getLong("id"));
        user.setName(rs.getString("name"));
        user.setAge(rs.getInt("age"));
        return user;
    }
}
```

任务

```java
@SpringBootApplication
public class HelloJob {


    @Bean
    public ItemWriter<User> itemWriter() {
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    @Bean
    public JdbcCursorItemReader<User> itemReader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder()
                .name("jdbcCursorItemReader")
                .dataSource(dataSource)
                // 执行 SQL，逐行读取返回的数据
                .sql("select * from user")
                // 映射数据
                .rowMapper(new UserRowMapper())
                .build();
    }


    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
                .<User, User>chunk(1, manager)
                .reader(itemReader(null))
                .writer(itemWriter())
                .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("cursor-db-job", jobRepository)
                .start(step)
                .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

如果希望动态拼接参数，则需要使用 `preparedStatementSetter()`：

```java
@Bean
public JdbcCursorItemReader<User> itemReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<User>()
        .name("jdbcCursorItemReader")
        .dataSource(dataSource)
        // 执行 SQL，逐行读取返回的数据
        .sql("select * from user where age = ?")
        // 映射数据
        .rowMapper(new UserRowMapper())
        // 拼接参数
        .preparedStatementSetter(new ArgumentPreparedStatementSetter(new Object[]{11}))
        .build();
}
```

#### 2.分页方式

游标的方式是查询出所有满足条件的数据，然后一条一条读取，分页则是按照指定的 pageSize 数，一次性读取。

```java
@SpringBootApplication
public class HelloJob {


    @Bean
    public ItemWriter<User> itemWriter() {
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    // 创建 pagingQueryProvider
    @Bean
    @SneakyThrows
    public PagingQueryProvider pagingQueryProvider(DataSource dataSource) {
        final SqlPagingQueryProviderFactoryBean factoryBean = new SqlPagingQueryProviderFactoryBean();
        factoryBean.setDataSource(dataSource);
        factoryBean.setSelectClause("select *");
        factoryBean.setFromClause("from user");
        // :age 表示占位符
        factoryBean.setWhereClause("where age >= :age");
        factoryBean.setSortKey("id");
        return factoryBean.getObject();
    }

    @Bean
    public JdbcPagingItemReader<User> itemReader(DataSource dataSource) {
        return new JdbcPagingItemReaderBuilder<User>()
            .name("jdbcPagingItemReader")
            .dataSource(dataSource)
            .pageSize(10)
            // 使用 pagingQueryProvider
            .queryProvider(pagingQueryProvider(null))
            // SQL 的参数
            .parameterValues(Map.of("age", 12))
            .rowMapper(new UserRowMapper())
            .build();
    }

    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
            .<User, User>chunk(1, manager)
            .reader(itemReader(null))
            .writer(itemWriter())
            .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("page-db-job", jobRepository)
            .start(step)
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

#### 3.扩展：Mybatis-plus

Mybatis-plus 也提供了上面介绍的两种方式，这里演示 `Paging` 方式。

**（1）引入依赖**

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
```

**（2）`Mapper.xml`**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.hello.UserMapper">
    <select id="select" resultType="com.example.hello.User" >
        <!-- 必须要指定 limit，否则可能会出现无限循环 -->
        select * from user where id between #{from} and #{to} limit #{_skiprows}, #{_pagesize}
    </select>
</mapper>
```

**（3）配置文件**

声明使用 batch 处理。

```yaml
mybatis-plus:
  configuration:
    default-executor-type: batch
```

**（4）`ItemReader`**

```java
@Bean
public MyBatisPagingItemReader<User> myBatisPagingItemReader(SqlSessionFactory sqlSessionFactory) {
    return new MyBatisPagingItemReaderBuilder<User>()
        .sqlSessionFactory(sqlSessionFactory)
        .queryId("com.example.hello.UserMapper.select")
        // 指定 pageSize，会自动注入到 SQL 的 _pageSize 中，并计算 _skiprows
        // 可以大于 chunk，因为是从数据库，chunk 控制的是每次传给 writer 的数量
        .pageSize(4)
        .parameterValues(Map.of("from", "10", "to", "100"))
        .build();
}
```

### 3.4 读取异常

#### 1.跳过异常

`ItemReader` 可以按照约定跳过指定的异常，也可以限制跳过次数。

```java
@Bean
public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
    return new StepBuilder("step", jobRepository)
        .<User, User>chunk(1, manager)
        .reader(itemReader(null))
        .writer(itemWriter())
        // 容错
        .faultTolerant()
        // 跳过的异常
        .skip(Exception.class)
        // 不应该跳过的异常
        .noSkip(RuntimeException.class)
        // 跳过异常的上限
        .skipLimit(3)
        .skipPolicy(new SkipPolicy() {
            @Override
            public boolean shouldSkip(Throwable t, long skipCount) throws SkipLimitExceededException {
                // 定制跳过异常的逻辑
                return false;
            }
        })
        .build();
}
```

#### 2.记录异常日志

使用 `ItemReadListener` 可以在 `itemReader` 读取数据抛出异常时将数据信息记录下来。

```java
public interface ItemReadListener<T> extends StepListener {

	default void beforeRead() {}

	default void afterRead(T item) {}

	default void onReadError(Exception ex) {}

}
```

## 4.ItemProcessor

使用 `ItemReader` 读取数据后，既可以直接通过 `ItemWriter` 输出数据，也可以通过 `ItemProcessor` 加工数据后再输出。

### 4.1 默认实现

Spring Batch 提供了 4 种实现。

#### 1.ValidatingItemProcessor：检验处理器

例如对于下面的数据：

```bash
1#li4#18
2##19
3#zhang3#24
```

第 2 个数据的 name 缺失，默认情况下依然会创建相应的对象，如果不想创建，则可以通过 `ValidatingItemProcessor` 来对数据进行校验。

**（1）导入依赖**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

**（2）定义实体类**

在相应字段添加校验注解：

```java
@Data
public class User {
    private Long id;
    @NotBlank(message = "用户名不能为空")
    private String name;
    private int age;
}
```

**（3）创建 Job**

Spring Bath 提供了 `BeanValidatingItemProcessor` 作为默认的实现类，此处创建 `BeanValidatingItemProcessor` 后在 step 中添加处理器即可。

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public ItemWriter<User> itemWriter() {
        // 打印各个对象
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    @Bean
    public FlatFileItemReader<User> flatFileItemReader() {
        return new FlatFileItemReaderBuilder<User>()
                .name("userItemReader")
                .resource(new ClassPathResource("user.txt"))
                .delimited().delimiter("#").names("id", "name", "age")
                .targetType(User.class)
                .build();
    }

    // 创建 BeanValidatingItemProcessor
    @Bean
    public BeanValidatingItemProcessor<User> validatingItemProcessor() {
        BeanValidatingItemProcessor<User> validatingItemProcessor = new BeanValidatingItemProcessor<>();
        // 不满足条件则丢弃数据
        validatingItemProcessor.setFilter(true);
        return validatingItemProcessor;
    }

    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
                // 使用 chunk，指定加载 1 次
                .<User, User>chunk(1, manager)
                .reader(flatFileItemReader())
                .processor(validatingItemProcessor())
                .writer(itemWriter())
                .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("validate-source-job", jobRepository)
                .start(step)
                .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

#### 2.ItemProcessorAdapter：适配器处理器

可以通过 `ItemProcessorAdapter` 对已有的校验方法进行适配。

例如我们已经定义了一个转换 name 大小写的 util：

```java
public class ValidationUtils {
    // 一定要有返回值
    public User toUppercase(User user) {
        user.setName(user.getName().toUpperCase());
        return user;
    }
}
```

对这个方法进行适配：

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public ItemWriter<User> itemWriter() {
        // 打印各个对象
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    @Bean
    public FlatFileItemReader<User> flatFileItemReader() {
        return new FlatFileItemReaderBuilder<User>()
            .name("userItemReader")
            .resource(new ClassPathResource("user.txt"))
            .delimited().delimiter("#").names("id", "name", "age")
            .targetType(User.class)
            .build();
    }

    // 创建适配器
    @Bean
    public ItemProcessorAdapter<User, User> itemProcessorAdapter() {
        ItemProcessorAdapter<User, User> adapter = new ItemProcessorAdapter<>();
        // 设置适配对象
        adapter.setTargetObject(new ValidationUtils());
        // 设置适配方法
        adapter.setTargetMethod("toUppercase");
        return adapter;
    }

    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
            // 使用 chunk，指定加载 1 次
            .<User, User>chunk(1, manager)
            .reader(flatFileItemReader())
			// 添加适配器
            .processor(itemProcessorAdapter())
            .writer(itemWriter())
            .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("validate-source-job-3", jobRepository)
            .start(step)
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

#### 3.ScriptItemProcessor：脚本处理器

Spring Batch 提供 JS 脚本的形式，将一些操作逻辑定义在 JS 文件中进行加载。

**（1）引入 JS 脚本引擎**

```xml
<dependency>
   <groupId>org.openjdk.nashorn</groupId>
   <artifactId>nashorn-core</artifactId>
   <version>15.3</version> <!-- latest -->
</dependency>
```

**（2）定义 userScript.js**

```javascript
item.setName(item.getName().toUpperCase());
item;
```

> **注意**
>
> `item` 是约定的单词，表示 `ItemReader` 读取的每个条目

**（3）使用 JS 脚本校验**

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public ItemWriter<User> itemWriter() {
        // 打印各个对象
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    @Bean
    public FlatFileItemReader<User> flatFileItemReader() {
        return new FlatFileItemReaderBuilder<User>()
            .name("userItemReader")
            .resource(new ClassPathResource("user.txt"))
            .delimited().delimiter("#").names("id", "name", "age")
            .targetType(User.class)
            .build();
    }

    // 创建脚本处理器
    @Bean
    public ScriptItemProcessor<User, User> itemProcessor() {
        return new ScriptItemProcessorBuilder<User, User>()
            // 指定文件源
            .scriptResource(new ClassPathResource("userScript.js"))
            .build();
    }

    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
            .<User, User>chunk(1, manager)
            .reader(flatFileItemReader())
            .processor(itemProcessor())
            .writer(itemWriter())
            .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("script-validation-job", jobRepository)
            .start(step)
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

#### 4.CompositeItemProcessor：组合处理器

 组合处理器类似于过滤器链，数据先经过第一个处理器，然后再经过第二个处理器，直到最后。

需求：先对 name 进行非空判断，再将 name 转换为大写。

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public ItemWriter<User> itemWriter() {
        // 打印各个对象
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    @Bean
    public FlatFileItemReader<User> flatFileItemReader() {
        return new FlatFileItemReaderBuilder<User>()
                .name("userItemReader")
                .resource(new ClassPathResource("user.txt"))
                .delimited().delimiter("#").names("id", "name", "age")
                .targetType(User.class)
                .build();
    }

    // 参数校验
    @Bean
    public BeanValidatingItemProcessor<User> validatingItemProcessor() {
        BeanValidatingItemProcessor<User> processor = new BeanValidatingItemProcessor<>();
        processor.setFilter(true);
        return processor;
    }

    // 转换大写
    @Bean
    public ScriptItemProcessor<User, User> itemProcessor() {
        return new ScriptItemProcessorBuilder<User, User>()
                .scriptResource(new ClassPathResource("userScript.js"))
                .build();
    }

    // 组装
    @Bean
    public CompositeItemProcessor<User, User> compositeItemProcessor() {
        final CompositeItemProcessor<User, User> processor = new CompositeItemProcessor<>();
        // 组合多个处理器
        processor.setDelegates(List.of(validatingItemProcessor(), itemProcessor()));
        return processor;
    }

    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
                // 使用 chunk，指定加载 1 次
                .<User, User>chunk(1, manager)
                .reader(flatFileItemReader())
                .processor(compositeItemProcessor())
                .writer(itemWriter())
                .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("composite-validation-job", jobRepository)
                .start(step)
                .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

### 4.2 自定义实现

 只需要实现 `ItemProcessor` 接口即可。

```java
public class MyItemProcessor implements ItemProcessor<User, User> {
    @Override
    public User process(User item) throws Exception {
        // 筛选出年龄为基数的 user，放弃则返回 null
        return item.getAge() % 2 != 0 ? item : null;
    }
}
```

## 5.ItemWriter

### 5.1 输出平面文件

可以通过 `FlatFileItemWriter` 输出器实现。

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public FlatFileItemWriter<User> flatFileItemWriter() {
        return new FlatFileItemWriterBuilder<User>()
            .name("userItemWriter")
            // 设置输出流
            .resource(new PathResource("H:\\project\\spring-batch-demo\\src\\main\\resources\\user-new.txt"))
            // 设置输出格式
            .formatted()
            .format("id: %s, 姓名: %s, 年龄: %s")
            .names("id", "name", "age")
            // 如果输入数据为空，输出时不创建文件
            // 默认为 false，即输入数据为空时也会创建一个空的输出文件
			.shouldDeleteIfEmpty(true)
            // 如果输出文件已经存在则直接删除
            .shouldDeleteIfExists(true)
            // 如果输出文件已经存在则追加写入
            .append(true)
            .build();
    }

    @Bean
    public FlatFileItemReader<User> flatFileItemReader() {
        return new FlatFileItemReaderBuilder<User>()
            .name("userItemReader")
            .resource(new ClassPathResource("user.txt"))
            .delimited().delimiter("#").names("id", "name", "age")
            .targetType(User.class)
            .build();
    }

    @Bean
    public Step step(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
            .<User, User>chunk(1, manager)
            .reader(flatFileItemReader())
            .writer(flatFileItemWriter())
            .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("writer-job", jobRepository)
            .start(step)
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

### 5.2 输出JSON

可以通过 `JsonFileItemWriter` 输出器实现。

```java
@Bean
public JsonFileItemWriter<User> jsonFileItemWriter() {
    // 创建 JSON 对象调度器
    final JacksonJsonObjectMarshaller<User> objectMarshaller = new JacksonJsonObjectMarshaller<>();
    objectMarshaller.setObjectMapper(new ObjectMapper());
    return new JsonFileItemWriterBuilder<User>()
        .name("userItemWriter")
        // 设置输出流
        .resource(new PathResource("H:\\project\\spring-batch-demo\\src\\main\\resources\\user-new.json"))
        // 设置 JSON 对象调度器
        .jsonObjectMarshaller(objectMarshaller)
        .build();
}
```

### 5.3 输出数据库

#### 1.JdbcBatchitemWriter

先定义一个 SQL 占位符参数设置器：

```java
public class MySetter implements ItemPreparedStatementSetter<User> {
    @Override
    public void setValues(User item, PreparedStatement ps) throws SQLException {
        ps.setLong(1, item.getId());
        ps.setString(2, item.getName());
        ps.setInt(3, item.getAge());
    }
}
```

`JdbcBatchItemWriter`

```java
@Bean
public JdbcBatchItemWriter<User> jdbcBatchItemWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<User>()
        .dataSource(dataSource)
        .sql("insert into user(id,name,age) values(?,?,?)")
        // 设置 SQL 中的占位符参数
        .itemPreparedStatementSetter(new MySetter())
        .build();
}
```

#### 2.Mybatis-plus

**（1）引入依赖**

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
```

**（2）`Mapper.xml`**

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.hello.UserMapper">
    <insert id="save" keyColumn="id" useGeneratedKeys="true" keyProperty="id">
        insert into user(id, name, age) values(#{id}, #{name}, #{age})
    </insert>
</mapper>
```

**（3）配置文件**

声明使用 batch 处理。

```yaml
mybatis:
	configuration:
		default-executor-type: batch
```

**（4）`ItemWriter`**

```java
@Bean
public MyBatisBatchItemWriter<User> batisBatchItemWriter(SqlSessionFactory sqlSessionFactory) {
    return new MyBatisBatchItemWriterBuilder<User>()
        .sqlSessionFactory(sqlSessionFactory)
        // 设置 mapper 文件中需要执行的语句的 namespace + id
        .statementId("com.example.hello.UserMapper.save")
        .build();
}
```

> **注意**
>
> 采用这种方式时不需要对应的接口。

### 5.4 输出多终端

可以通过 `CompositeItemWriter` 输出器实现。

```java
@Bean
public CompositeItemWriter<User> compositeItemWriter() {
    return new CompositeItemWriterBuilder<User>()
        .delegates(List.of(jsonFileItemWriter(), jdbcBatchItemWriter(null)))
        .build();
}
```

