# 线程池相关

## 背景
之前的一些定时任务是使用@schedule注解定义，虽然@Scheduled注解非常便捷，但是它也存在一些多线程的问题，主要体现在以下两个方面：

定时任务未执行完毕时，后续任务可能会受到影响

在使用@Scheduled注解时，我们很容易忽略一个问题：如果定时任务在执行时，下一个周期的任务已经到了，那么后续任务可能会受到影响。例如，我们定义了一个间隔时间为5秒的定时任务A，在第1秒时开始执行，需要执行10秒钟。在第6秒时，定时任务A还没有结束，此时下一个周期的任务B已经开始等待执行。如果此时线程池中没有足够的空闲线程，那么定时任务B就会被阻塞，无法执行。

## 解决方案
使用线程池配置来执行定时任务

```java
@Configuration
@EnableScheduling
public class TaskExecutorConfig {
    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(1000);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("task-executor-");
        return executor;
    }
}
```
通过配置ThreadPoolTaskExecutor来创建一个线程池，并使用@EnableScheduling注解将定时任务开启。其中setCorePoolSize、setMaxPoolSize、setQueueCapacity、setKeepAliveSeconds等方法可以用于配置线程池的大小和任务队列等参数。

## 线程池配置策略
一般说来，大家认为线程池的大小经验值应该这样设置：（其中N为CPU processors的个数）
（1）如果是CPU密集型应用，则线程池大小设置为N+1(或者是N)，线程的应用场景：主要是复杂算法
（2）如果是IO密集型应用，则线程池大小设置为2N+1(或者是2N)，线程的应用场景：主要是：数据库数据的交互，文件上传下载，网络数据传输等等
+1的原因是：即使当计算密集型的线程偶尔由于缺失故障或者其他原因而暂停时，这个额外的线程也能确保CPU的时钟周期不会被浪费。
