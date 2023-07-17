---

title: Spring 事务详解
author: vic
date: 2022-02-21 00:34:00 +0800
categories: [Blogging, Java]
tags: [favicon]
typora-root-url: ../../
---



## 1. 什么是事务

### 1.1 Introduction

数据库事务处理的一系列操作，事务管理是面向 RDBMS的企业应用中保证数据完整性和一致性的重要组成部分

### 1.2 Transaction的特性(ACID)

- 原子性（Atomicity） 原子性是指事务是一个不可分割的工作单位，事务中的操作要么全部成功，要么全部失败。比如在同一个事务中的 SQL 语句，**要么全部执行成功，要么全部执行失败**
- 一致性（Consistency） 官网上事务一致性的概念是：事务必须使数据库从一个一致性状态变换到另外一个一致性状态。换一种方式理解就是：事务按照预期生效，**数据的状态是预期的状态**。
- 隔离性（Isolation） 事务的隔离性是多个用户并发访问数据库时，数据库为每一个用户开启的事务，不能被其他事务的操作数据所干扰，多个并发事务之间要相互**隔离**
- 持久性（Durability） 持久性是指一个事务一旦被提交，它对数据库中数据的改变就是永久性的，接下来即使数据库发生故障也不应该对其有任何影响

针对这四个特性，RDBMS  都会支持，一致性和持久性我们在这里不用讨论，这个是DBMS的基础。

### 1.3 数据库事务隔离界别(Isolation)

数据库的隔离级别分为四种`read uncommitted`  `read committed`  `repeatable read` `serializable`

依次 隔离级别越来越高，对数据库性能影响越来越大。

#### **Read-uncommitted** 

顾名思义 读未提交，在一个数据库session中，可以读到另一个线程session操作了数据库但是没有commit 结果，这种就会造成***脏读(“Dirty” read)***

这是`脏读(“Dirty” read)`场景



![Non-repeatable read](/assets/img/post_image/tveqyjcfpkib1kudw0uiunuyrm8.png)

#### **Read-committed**

读已提交，在一个数据库session中，可以读到另一个线程已经提交的数据，这样虽然避免了脏读，但是还是不够安全，如果在一个线程中 重复查询同一条数据多次，在这期间发生了commit 就会导致 多次查询的结果不一致，这在正常业务中也是有问题 这个我们称之为**Non-repeatable read不可重复读**

这是`Non-repeatable read不可重复读`场景

![Non-repeatable read](/assets/img/post_image/vzqcdiwkywlpv3z-csty11mm4jo.png)



#### **Repeatable-read**

可重复读，可以解决read committed 不可重复读的问题，**这也是目前大部分DBMS默认的事务隔离界别** 能保证在一个线程中多次读取的一致性；但在一些严格的场景中 还是会存在一些问题  比如此级别的事务正在执行时，另一个事务成功插入了某条数据，但因为它每次查询的结果都是一样的，所以会导致查询不到这条数据，自己重复插入时又失败（因为唯一约束的原因）。**明明在事务中查询不到这条信息，但自己就是插入不进去，这就叫幻读 （Phantom Read）。** 

这是`幻读 （Phantom Read）`场景

![Phantom Read](/assets/img/post_image/j9g_byffovppjeqxaogzvunufei.png)

#### **Serializable**

串行读写, 所有的操作都是排成一个队列依次执行,一致性不会有任何问题，但是会带来严重的性能问题，所以不推荐这么做

总结这四种级别的针对脏读，不可重复读，幻读

| Isolation level  | Phantom read | Non-repeatable read | “Dirty” read |
| ---------------- | ------------ | ------------------- | ------------ |
| READ_UNCOMMITTED | possible     | possible            | possible     |
| READ_COMMITTED   | possible     | possible            | not possible |
| REPEATABLE_READ  | possible     | not possible        | not possible |
| SERIALIZABLE     | not possible | not possible        | not possible |

在Spring中也提供了的这四种隔离级别的配置

##  2. Spring 中事务传播机制

试想这样一个场景  在一个进程嵌套调用多个service，而每个service 可能都通过全局 AOP 或者@Transactional 来定义事务。这是我们有可能期望   多个service在一个事务中或者每个service单独有自己的事务 这个过程是灵活。所以就需要我们在项目定义事务的传播机制。spring 对传播机制有四种定义

#### PROPAGATION_REQUIRED(默认)

- 支持当前事务，如果当前没有事务，则新建事务
- 如果当前存在事务，则加入当前事务，合并成一个事务

#### PROPAGATION_SUPPORTS

- 如果当前存在事务，则加入事务
- 如果当前不存在事务，则以非事务方式运行，这个和不写没区别

#### PROPAGATION_MANDATORY

当前存在事务，则加入当前事务，如果当前事务不存在，则抛出异常

####  PROPAGATION_REQUIRES_NEW

- 新建事务，如果当前存在事务，则把当前事务挂起
- 这个方法会独立提交事务，不受调用者的事务影响，父级异常，它也是正常提交

#### PROPAGATION_NOT_SUPPORTED

- 以非事务方式运行
- 如果当前存在事务，则把当前事务挂起

#### PROPAGATION_NEVER

- 如果当前存在事务，则运行在当前事务中
- 如果当前无事务，则抛出异常，也即父级方法必须有事务

#### PROPAGATION_NESTED

- 如果当前存在事务，它将会成为父级事务的一个子事务，方法结束后并没有提交，只有等父事务结束才提交
- 如果当前没有事务，则新建事务
- 如果它异常，父级可以捕获它的异常而不进行回滚，正常提交
- 但如果父级异常，它必然回滚，这就是和 `REQUIRES_NEW` 的区别

## 3. Spring 中控制事务的方式

Spring 通过TransactionManager实现的事务的控制，不过我们的项目一般不直接使用TransactionManager 而是通过一下两种方式控制

- @Transactional

  在Class或者mothod上注解的@Transactional 即可实现事务的控制

  @Transactional(propagation={配置传播机制},isolation={配置隔离级别})

- AOP + TransactionInterceptor

  ```java
  @Configuration
  @Aspect
  public class TransactionConfig {
    @Autowired
    private TransactionManager transactionManager;
  
    @Bean
    public TransactionInterceptor txAdvice() {
        // read only
        RuleBasedTransactionAttribute readOnly = new RuleBasedTransactionAttribute();
        readOnly.setReadOnly(true);
        //配置传播机制
        readOnly.setPropagationBehavior(TransactionDefinition.PROPAGATION_SUPPORTS);
        // required
        RuleBasedTransactionAttribute required = new RuleBasedTransactionAttribute();
        RollbackRuleAttribute rollbackOnException = new RollbackRuleAttribute(Exception.class);
        required.setRollbackRules(Collections.singletonList(rollbackOnException));
        required.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
         //配置隔离级别
        required.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();
        source.addTransactionalMethod("update*", required);
        source.addTransactionalMethod("view", readOnly);
        return new TransactionInterceptor(transactionManager, source);
    }
  
    @Bean("txAdviceAdvisor")
    public Advisor txAdviceAdvisor() {
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression("");
        return new DefaultPointcutAdvisor(pointcut, txAdvice());
    }
  }
  
  ```

## 4. 事务的分类

### 扁平事务

再扁平事务中要么全部回滚，要么全部成功，事务要保持原子性，这个也各个数据库和框架默认事务类型，对应到传播机制中就是REQUIRED，扁平事务就是限制不能回滚或者提交一部分。



### 带有保存点的扁平事务

而允许提交操作中的一部分操作，这种情况下，就需要将一系列操作分为几个不同的事务，通过代码来控制事务的提交流程

这就要熟练运用事务传播机制，来构建自己的事务

比如：

事务A开始

操作A

操作B

  事务B Begin

​     操作C

​    操作D

 事务B Commit

操作D

事务A Commit







### 链式事务

### 嵌套事务

### 分布式事务

