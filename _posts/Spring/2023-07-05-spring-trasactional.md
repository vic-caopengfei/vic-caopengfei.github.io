---

title: Spring 事务详解
author: vic
date: 2022-02-21 00:34:00 +0800
categories: [Blogging, Java]
tags: [favicon]
typora-root-url: ../../
---

# 1. 什么是事务

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

# 2. Spring 中事务传播机制

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

# 3. Spring 中控制事务的方式

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

# 4. 事务的分类及其实现

## 扁平事务

在扁平事务中操作要么全部回滚，要么全部成功，事务要保持原子性，**现实项目中用的最多的事务类型**，对应到Spring传播机制中就是REQUIRED

### mysql 实现

```sql
START TRANSACTION;

-- 查询账户余额
SELECT balance FROM accounts WHERE account_id = 1;

-- 查询账户余额
SELECT balance FROM accounts WHERE account_id = 2;

-- 转账操作
UPDATE accounts SET balance = balance - 200 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 200 WHERE account_id = 2;

-- 提交整个事务
COMMIT;

```

### Spring 实现 

 ```java
 //Spring 默认的事务传播机制，propagation=TransactionDefinition.PROPAGATION_REQUIRED 可以省略
 @Transactional(propagation=TransactionDefinition.PROPAGATION_REQUIRED)
 public void tranfrom(){
   Account account1 = queryAccount(1);
   Account account2 = queryAccount(2);
   update(account1)
   update(account2)
 }
 ```

然而现实项目中当进行一些列复杂的操作后，事务发生异常但不需要全部回滚，而是想回滚其中的一部分操作，这样就出现**带有保存点的事务**，这样做的好处就是减少了数据库访问，优化程序的性能

## 带有保存点的扁平事务


在 MySQL 中，你可以使用保存点（Save point）来实现扁平事务（Flat Transaction），这样可以在事务内部设置多个保存点，并在需要的情况下回滚到指定的保存点，而不必回滚整个事务，下面是一个带有保存点的扁平事务的例子。**发生异常时撤销其中的一部分操作**

### Mysql 实现

```sql
START TRANSACTION;

-- 保存点1
SAVEPOINT sp1;

UPDATE employees SET salary = salary + 10000 WHERE id = 1;
UPDATE employees SET salary = salary + 8000 WHERE id = 2;

-- 保存点2
SAVEPOINT sp2;

UPDATE employees SET salary = salary + 12000 WHERE id = 3;

-- 执行一些其他操作...

-- 回滚到保存点2，只撤销了保存点2后的修改，不影响保存点1的修改
ROLLBACK TO SAVEPOINT sp2;

-- 继续执行其他操作...

-- 提交整个事务，包括保存点1的修改
COMMIT;
```



在上面的例子中，我们创建了一个带有保存点的扁平事务。首先，我们启动了事务（`START TRANSACTION`），然后在两个 `UPDATE` 语句之间设置了保存点1（`SAVEPOINT sp1`）。接着进行了一系列操作，然后又设置了保存点2（`SAVEPOINT sp2`）。

在事务执行的过程中，我们可以随时回滚到指定的保存点，这里我们选择回滚到保存点2（`ROLLBACK TO SAVEPOINT sp2`），这样只会撤销保存点2后的修改，保存点1的修改仍然保留。最后，如果一切正常，我们提交整个事务（`COMMIT`），使所有的修改生效。

### Spring 实现

可以使用着两句来实现事务的保存点，手动控制事务的流程

```java
//方式1 TransactionAspectSupport
Object savePoint = TransactionAspectSupport.currentTransactionStatus().createSavepoint();		 
...		
TransactionAspectSupport.currentTransactionStatus().rollbackToSavepoint(savePoint);

//方式2 SQL
// 设置保存点1
jdbcTemplate.execute("SAVEPOINT sp1");

// 第三个更新操作
jdbcTemplate.update("UPDATE employees SET salary = salary + 12000 WHERE id = 3");

// 执行一些其他操作...

// 回滚到保存点1，只撤销了保存点1后的修改，不影响前面的修改
jdbcTemplate.execute("ROLLBACK TO SAVEPOINT sp1");

```

## 链式事务

- 链事务可视为保存点模式的一种变种，带有保存点的扁平事务，当发生系统崩溃时，所有的保存点都将消失，因为其保存点是易失的（volatile），而非持久的( persistent)。这意味着当进行恢复时，事务需要从开始处重新执行，而不能从最近的一个保存点继续执行
  链事务的思想是：在提交一个事务时，释放不需要的数据对象，将必要的处理上下文隐式地传给下一个要开始的事务

注意，提交事务操作和开始下一个事务操作将合并为一个原子操作。这意味着下一个事务将看到上一个事务的结果，就好像在一个事务中进行的一样。下图显示了链事务的工作方式：

### mysql 实现（不支持）

Mysql不直接支持链式事务,可以通过savepoint模拟

### Spring实现

```java
@Transactional(propagation=TransactionDefinition.PROPAGATION_REQUIRES_NEW)
```

## 嵌套事务

嵌套事务是指在一个事务内部可以再开启一个新的子事务，形成了多层嵌套的事务结构。在嵌套事务中，子事务可以独立于父事务进行提交或回滚，而不会影响父事务的提交或回滚。嵌套事务允许在一个事务内部有更细粒度的事务控制，对于复杂的业务场景和数据操作，可以提供更灵活和精确的事务管理。

嵌套事务的特点包括：

1. 嵌套事务形成多层结构：一个事务内部可以开启另一个事务，形成多个层次的事务结构。
2. 子事务独立于父事务：子事务可以独立于父事务进行提交或回滚，子事务的提交或回滚不会影响父事务的状态，主事务的回滚会导致子事务回滚
3. 事务传播行为：在嵌套事务中，需要定义事务传播行为，指定子事务如何与父事务进行交互。

需要注意的是，嵌套事务并不是所有数据库都原生支持的特性，具体的支持程度取决于数据库的类型和版本。在某些数据库中，嵌套事务被转换成普通的非嵌套事务，即嵌套事务的提交或回滚会被转化为整个事务的提交或回滚。

### mysql 实现（不支持）

mysql不直接支持链式事务

### Spring实现

Spring多种传播机制，可以自己组合达到自己的想要的事务结果。详情可以参考。

PROPAGATION_NESTED：子事务回滚，父事务不回滚，父事务回滚，子事务回滚 使用 

PROPAGATION_REQUIRES_NEW： 父子事务相互不影响，但是有有先后顺序，所有的子事务执行完成，父事务才会提交

## 分布式事务Distributed Transaction

随着系统的微服务化，出现了进程间的事务，这种操作无法通过的数据库的机制来保证事务的AICD。

### CAP理论

CAP 原则又称 CAP 定理，指的是在一个分布式系统中， Consistency（一致性）、 Availability（可用性）、Partition tolerance（分区容错性），三者不可得兼

- 一致性（C）

在分布式系统中的所有数据备份，在同一时刻是否同样的值。（等同于所有节点访问同一份最新的数据副本）

- 可用性（A）

在集群中一部分节点故障后，集群整体是否还能响应客户端的读写请求。（对数据更新具备高可用性）

- 分区容错性（P）

以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在 C 和 A 之间做出选择。

![](/assets/img/post_image/v2-2f26a48f5549c2bc4932fdf88ba4f72f_720w.jpg)

CAP 原则的精髓就是要么 AP，要么 CP，要么 AC，但是不存在 CAP。如果在某个分布式系统中数据无副本， 那么系统必然满足强一致性条件， 因为只有独一数据，不会出现数据不一致的情况，此时 C 和 P 两要素具备，但是如果系统发生了网络分区状况或者宕机，必然导致某些数据不可以访问，此时可用性条件就不能被满足，即在此情况下获得了 CP 系统，但是 CAP 不可同时满足。而分布式系统数据库也会存在多个的情况，没有办法做到强一致性

### Base 理论

BASE 理论指的是基本可用 Basically Available，软状态 Soft State，最终一致性 Eventual Consistency，核心思想是即便无法做到强一致性，但应该采用适合的方式保证**最终一致性**

- BASE，Basically Available Soft State Eventual Consistency 的简写：

- BA：Basically Available 基本可用，分布式系统在出现故障的时候，允许损失部分可用性，即保证核心可用

- S：Soft State 软状态，允许系统存在中间状态，而该中间状态不会影响系统整体可用性。

- E：Consistency 最终一致性，系统中的所有数据副本经过一定时间后，最终能够达到一致的状态。

  BASE 理论本质上是对 CAP 理论的延伸，是对 CAP 中 AP 方案的一个补充

### 柔性事务

不同于 ACID 的刚性事务，在分布式场景下基于 BASE 理论，就出现了柔性事务的概念。要想通过柔性事务来达到最终的一致性，就需要依赖于一些特性，这些特性在具体的方案中不一定都要满足，因为不同的方案要求不一样；但是都不满足的话，是不可能做柔性事务的

### 幂等操作

在编程中一个幂等操作的特点是其任意多次执行所产生的影响均与一次执行的影响相同。幂等函数，或幂等方法，是指可以使用相同参数重复执行，并能获得相同结果的函数。这些函数不会影响系统状态，也不用担心重复执行会对系统造成改变。例如，支付流程中第三方支付系统告知系统中某个订单支付成功，接收该支付回调接口在网络正常的情况下无论操作多少次都应该返回成功

## 分布式事务场景

#### 转账/跨行转账

用户A要从银行A 转账给银行B的用户B，这时候就出现

A扣款成功，B增加余额失败

A扣款失败，B增加余额成功

这两种情况对应银行系统是不能接受的

#### 下单扣库存

在微服务系统下，下单系统和库存系统都会属于不通的微服务实现，用户下单就要扣掉库存，同样会出现下单成功，未扣库存；下单失败，扣库存成功的情况

#### 第三方支付

继续以电商系统为例，在微服务体系架构下，我们的支付与订单都是作为单独的系统存在。订单的支付状态依赖支付系统的通知，假设一个场景：我们的支付系统收到来自第三方支付的通知，告知某个订单支付成功，接收通知接口需要同步调用订单服务变更订单状态接口，更新订单状态为成功。流程图如下，从图中可以看出有两次调用，第三方支付调用支付服务，以及支付服务调用订单服务，这两步调用都可能出现调用超时的情况，此处如果没有分布式事务的保证，就会出现用户订单实际支付情况与最终用户看到的订单支付情况不一致的情况。

## 分布式事务解决方案

### 两阶段提交/XA

#### 基本概念

XA 协议是由 X/Open 公司于 1991 年发布的一套标准协议。XA 是 eXtended Architecture 的缩写，因此该协议旨在解决如何在异构系统中保证全局事务的原子性。

XA一共分为两阶段

第一阶段（prepare）：即所有的参与者RM准备执行事务并锁住需要的资源。参与者ready时，向TM报告已准备就绪。 第二阶段 (commit/rollback)：当事务管理者(TM)确认所有参与者(RM)都ready后，向所有参与者发送commit命令。

![](/assets/img/post_image/Snipaste_2023-07-19_14-33-53.jpg)

目前主流的数据库基本都支持XA事务，包括mysql、oracle、sqlserver、postgre 

#### 存在的问题

- 同步阻塞问题

  本地资源管理器会同时锁资源，造成服务阻塞

- 单点故障问题

  依赖于事务管理器，如果事务管理器挂掉，或者和资源管理器网络有问题，就会**造成数据不一致** 或者结果无法确定

#### 适用场景

追求强一致性和数据的时效性，并发量不高的系统

### 补偿事务TCC

#### 基本概念

关于 TCC（Try-Confirm-Cancel）的概念，最早是由 Pat Helland 于 2007 年发表的一篇名为《Life beyond Distributed Transactions:an Apostate’s Opinion》的论文提出。 TCC 事务机制相比于上面介绍的 XA，解决了其

TCC(Try Confirm Cancel)
Try 阶段：尝试执行，完成所有业务检查（一致性）, 预留必须业务资源（准隔离性）
Confirm 阶段：确认执行真正执行业务，不作任何业务检查，只使用 Try 阶段预留的业务资源，Confirm 操作满足幂等性。要求具备幂等设计，Confirm 失败后需要进行重试。
Cancel 阶段：取消执行，释放 Try 阶段预留的业务资源 Cancel 操作满足幂等性 Cancel 阶段的异常和 Confirm 阶段异常处理方案基本上一致。

在 Try 阶段，是对业务系统进行检查及资源预览，比如订单和存储操作，需要检查库存剩余数量是否够用，并进行预留，预留操作的话就是新建一个可用库存数量字段，Try 阶段操作是对这个可用库存数量进行操作。
基于 TCC 实现分布式事务，会将原来只需要一个接口就可以实现的逻辑拆分为 Try、Confirm、Cancel 三个接口，所以代码实现复杂度相对较高

![](/assets/img/post_image/s6k4jk82zh.png)

#### 存在问题

1. 解决了协调者单点，由主业务方发起并完成这个业务活动。业务活动管理器也变成多点，引入集群。
2. 同步阻塞：引入超时，超时后进行补偿，并且不会锁定整个资源，将资源转换为业务逻辑形式，粒度变小。
3. 数据一致性，有了补偿机制之后，由业务活动管理器控制一致性
4. 增加了编码的复杂度，不能很好的被复用

#### 适用场景

业务复杂度低，追求强一致性和数据的时效性的系统

### 基于消息队列的最终一致性



![](/assets/img/post_image/99999999999native-message.jpg)

#### 基本概念

MQ中间件担当了事务协调的角色，实现最终的一致性，所有的操作要实现幂等操作

#### 存在问题

数据更新不及时

#### 适用场景

**不要求强一致性，高并发场景**，目前实现就阿里的Rocket MQ 可以实现

### Sagas 事务模型

基本概念



存在问题

使用场景





