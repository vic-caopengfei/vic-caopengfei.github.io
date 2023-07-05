---
title: Springboot自带定时器使用和原理解析
author: vic
date: 2023-07-05 00:34:00 +0800
categories: [Blogging, Tutorial]
tags: [favicon]

---


# Springboot定时任务使用和原理解析
- 大型项目中一般会用到quartz进行定时任务的管理，但是一些小型目不需要进行分布式部署或者简单的任务则可以使用Springboot自带的@Scheduled来进行实现。我们一块来看下是如何使用以及实现的原理

#### 一、Springboot定时任务使用
#### 1. 简单配置定时任务
1).  在Springboot启动类添加注释@EnableScheduling 例如:
```
@EnableScheduling
public class Application{

}
```
2). 在执行的方式上使用@Scheduled注解 例如：
```
@Scheduled(cron = "0 0/1 * * * ?")
    public void cronJob() {
}
```
这样就搞定了
#### 2. 通过@Configuration配置
第一步  将启动类的@EnableScheduling去掉

第二步  添加配置类SchedulerConfig
```
@Configuration
@ConditionalOnProperty("scheduling.enabled")
public class SchedulerConfig {

    @Bean(name = TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)
    public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
        return new ScheduledAnnotationBeanPostProcessor();
    }
}
```
3. 在application.yml 添加配置
```
scheduling:
  enabled: true #默认不启动定时器true 启动false 不启动
```
这种方式可以通过配置文件来控制定时任务是否启用
>1. @ConditionalOnProperty用法: 配合@Configuration进行使用，当ConditionalOnProperty的为true时当前配置类生效

------------
#### 二、定时器原理解析
先说原理：
1.  Springboot定时任务是通过jdk java.util.concurrent包的`ScheduledThreadPoolExecutor`实现，
2.  本次任务开始执行时，会确定下次任务的执行时间，将任务提交到ScheduledThreadPoolExecutor中，以达到循环执行的效果

>所以想要了解原理必须先了解ScheduledThreadPoolExecutor。总结一句话就是 ：ScheduledThreadPoolExecutor继承自ThreadPoolExecutor。它主要用来在给定的延迟之后执行任务，或者定期执行任务。

我们就以ScheduledAnnotationBeanPostProcessor 这个类为起点看下源码是如何实现的
- 第一步 加载@Scheduled注解信息 
ScheduledAnnotationBeanPostProcessor初始化完成调用postProcessAfterInitialization，扫描带有注解 @Scheduled的，并把缓存起来（根据bean的声明周期postProcessAfterInitialization会在bean初始化完成后执行，）。
不了解Spring bean的初始化过程的可以看下图

![](https://vic-caopengfei.github.io/assets/img/post_image/up-5da34323581e78362b857c1c30d30f96ba5.webp)

源码如下：

![](https://vic-caopengfei.github.io/assets/img/post_image/up-65ad558251b67e5c188cdd1518a57254691.webp)
- 第二步:实例化完成后进行服务注册
>ScheduledAnnotationBeanPostProcessor加载并实例化后触发onApplicationEvent finishRegistration() -> afterPropertiesSet 进行服务注册,任务最终由ReschedulingRunnable的schedule添加到了线程池中的

1. 调用onApplicationEvent
![](https://vic-caopengfei.github.io/assets/img/post_image/up-6944a09b2f43906f16bbeab68afcc27275c.webp)
2. ScheduledTaskRegistrar的scheduledCornTask进行的服务注册
![](https://vic-caopengfei.github.io/assets/img/post_image/up-4defa1f27ace71d30aebfbcf4e17d13dd5d.webp)
3. ScheduledTaskRegistrar中进行任务注册
![](https://vic-caopengfei.github.io/assets/img/post_image/up-cff7e14dc224bf8eb86139f49fe23909b39.webp)
4.  ConcurrentTaskScheduler 的schedule方法
![](https://vic-caopengfei.github.io/assets/img/post_image/up-77bcdd4e641ebf59dfe1d7725a4b5d8b422.webp)
5.  最终在ReschedulingRunnable的schedule()方法中提交了ScheduledThreadPoolExecutor进行执行
![](https://vic-caopengfei.github.io/assets/img/post_image/up-8659ca75b5143c3ab4a6da0f70afffea48a.webp)
到此任务注册完成，指定时间会有线程池进行执行。
- 第三步： 循环执行逻辑
>本次任务开始执行时，会确定下次任务的执行时间，并将任务提交到ScheduledThreadPoolExecutor中，以达到循环执行的效果

![](https://vic-caopengfei.github.io/assets/img/post_image/up-62d9780d40ef468830a51aa219b083bab6e.webp)
> 以上这就是Springboot定时器实现的全部逻辑。经过分析要注意一下两个问题
1. ScheduledThreadPoolExecutor 是使用`Executors.newSingleThreadScheduledExecutor();`创建，所有定时任务会共用这一个线程池，并且线程池的核心线程数是1，任务执行不及时，会造成其他任务的延时。
2. 如果系统是分布式，那定时任务不能同时开启。不然造成任务的重复执行
