# 第03章_Spring Batch高级

## 1.STEP操作

### 3.1 多线程

默认情况下 `Step` 是在单线程中执行，可以更改为多线程，但是在多线程环境下 `Step` 要**设置为不可重启**。

Spring Batch 的多线程步骤是通过 Spring 的 `TaskExecutor` 实现的，约定每一个块开启一个线程独立执行，等所有线程的 `Reader` 操作完成后，然后将所有的数据封装成一个参数传给 `Writer`。

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public ItemWriter<User> itemWriter() {
        return chunk -> chunk.getItems().forEach(System.out::println);
    }

    @Bean
    public FlatFileItemReader<User> flatFileItemReader() {
        return new FlatFileItemReaderBuilder<User>()
                .name("userItemReader")
                // 不保存状态，防止状态被覆盖
                .saveState(false)
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
                .writer(itemWriter())
                // 设置一个taskExecutor
                .taskExecutor(new SimpleAsyncTaskExecutor())
                .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step step) {
        return new JobBuilder("thread-job", jobRepository)
                .start(step)
                .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

> **提示：`saveState(false)`**
>
> Spring Batch 提供大部分的 `itemReader` 是有状态的，作业重启基本通过状态来确定作业停止位置，而在多线程环境下，如果对象维护状态被多个线程访问，可能存在线程间状态相互覆盖问题。所以设置 false 表示关闭，这也意味着作业不能重启了。

### 3.2 并行

当读取 2 个或者多个互不关联的文件时，可以多个文件同时读取。

```java
@SpringBootApplication
public class HelloJob {

    @Bean
    public ItemWriter<User> itemWriter() {
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

    @Bean
    public JsonItemReader<User> jsonItemReader() {
        return new JsonItemReaderBuilder<User>()
            .name("jsonItemReader")
            .resource(new ClassPathResource("user.json"))
            .jsonObjectReader(new JacksonJsonObjectReader<>(User.class))
            .build();
    }

    @Bean
    public Step flatStep(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
            .<User, User>chunk(2, manager)
            .reader(flatFileItemReader())
            .writer(itemWriter())
            .build();
    }

    @Bean
    public Step jsonStep(JobRepository jobRepository, PlatformTransactionManager manager) {
        return new StepBuilder("step", jobRepository)
            // 每次执行两次 reader
            .<User, User>chunk(2, manager)
            .reader(jsonItemReader())
            .writer(itemWriter())
            .build();
    }

    @Bean
    public Job job(JobRepository jobRepository, Step flatStep, Step jsonStep) {
        // 线程 1 读取 flat 文件
        Flow flow1 = new FlowBuilder<Flow>("flat flow")
            .start(flatStep)
            .build();
        // 线程 2 读取 json 文件
        Flow flow2 = new FlowBuilder<Flow>("json flow")
            .start(jsonStep)
            // 开启线程执行步骤
            .split(new SimpleAsyncTaskExecutor())
            .add(flow1)
            .build();
        return new JobBuilder("thread-job", jobRepository)
            .start(flow2)
            .end()
            .build();
    }

    public static void main(String[] args) {
        SpringApplication.run(HelloJob.class, args);
    }
}
```

### 3.3 分区

Spring Batch 中分布步骤指的是给执行步骤区分上下级：

- 上级：Master Step
- 下级：Work Step

Work Step 不管多小也是一个完整的 Spring Batch 步骤，负责各自的读取、处理、写入等。

分布步骤一般用于海量数据的处理上，采用的是分治思想。主步骤将大的数据划分为多个小的数据集，然后开启多个从步骤，要求每个从步骤负责一个数据集。当所有从步骤处理结束，整作业流程才算结束。

**分区处理器**

Master Step 核心组件，统一管理 worker 步骤，并给 worker 步骤指派任务，管理规则由 `PartitionHandler` 接口定义，默认的实现类是 `TaskExecutorPartitionHandler`。

**分区器**

Master Step 核心组件，负责数据分区，将完整的数据拆解成多个数据集，然后指派给 Work Step。拆分规则由 `Partitioner` 分区器接口指定，默认的实现类为 `MultiResourcePartitioner`。

```java
@FunctionalInterface
public interface Partitioner {
	Map<String, ExecutionContext> partition(int gridSize);
}
```

`Partitioner` 接口只有唯一的方法 `partition()`，参数 `gridSize` 表示要分区的大小，可以理解为要开启多个 worker 步骤，返回值是一个 Map，key 为 worker 步骤名称，value 为 worker 步骤启动需要的参数值，一般包含分区元数据，例如起始位置，数据量等。

**案例**

现有 5 个文件：`user-1.txt`、`user-2.txt`、`user-3.txt`、`user-4.txt`、`user-5.txt`，要求读取并写入。

**（1）创建分区器**

配置从步骤分区名称 + 配置从步骤上下文环境

```java
public class UserPartitioner implements Partitioner {
    @SneakyThrows
    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Map<String, ExecutionContext> map = new HashMap<>();
        // 为每个分区创建一个执行上下文，将分区的信息放入上下文中
        for (int i = 0; i < gridSize; i++) {
            // 创建执行上下文
            ExecutionContext executionContext = new ExecutionContext();
            // 创建文件资源对象
            String fileName = "user-" + (i + 1) + ".txt";
            Resource resource = new ClassPathResource(fileName);
            // 将资源文件转换为字符串形式放入执行上下文
            executionContext.putString("file", resource.getURL().toExternalForm());
            // 设置从步骤名称作为 key，执行上下文作为 value
            map.put("user_partition_" + i, executionContext);
        }
        return map;
    }
}
```

**（2）创建从步骤**

```java
@Bean
// 开启懒加载
@StepScope
// 多个从步骤共用同一个 Reader，每次从 stepExecutionContext 中获取分区信息（file 资源），该资源在主步骤的分区器中设置
public FlatFileItemReader<User> flatFileItemReader(@Value("#{stepExecutionContext['file']}") Resource resource) {
    return new FlatFileItemReaderBuilder<User>()
        .name("flatFileItemReader")
        .resource(resource)
        .delimited()
        .delimiter("#")
        .names("id", "name", "age")
        .targetType(User.class)
        .build();
}

@Bean
public ItemWriter<User> itemWriter() {
    return chunk -> chunk.getItems().forEach(System.out::println);
}

// 创建从步骤
@Bean
public Step workStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("workStep", jobRepository)
        .<User, User>chunk(9, transactionManager)
        .reader(flatFileItemReader(null))
        .writer(itemWriter())
        .build();
}
```

**（3）创建分区处理器**

```java
@Bean
public PartitionHandler partitionHandler() {
    TaskExecutorPartitionHandler partitionHandler = new TaskExecutorPartitionHandler();
    // 设置分区个数，即从步骤个数
    partitionHandler.setGridSize(5);
    // 设置线程，每个分区由一个线程执行
    partitionHandler.setTaskExecutor(new SimpleAsyncTaskExecutor());
    // 关联从步骤
    partitionHandler.setStep(workStep(null, null));
    return partitionHandler;
}
```

**（4）创建主步骤及任务**

```java
@Bean
public Step masterStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
    return new StepBuilder("masterStep", jobRepository)
        // 设置分区器
        .partitioner(workStep(null, null).getName(), new UserPartitioner())
        // 设置分区处理器
        .partitionHandler(partitionHandler())
        .build();
}

@Bean
public Job job(JobRepository jobRepository) {
    return new JobBuilder("partition-job", jobRepository)
        .start(masterStep(null, null))
        .build();
}
```