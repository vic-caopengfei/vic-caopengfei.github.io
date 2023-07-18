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

## 4. 事务的分类及其实现

### 扁平事务

在扁平事务中操作要么全部回滚，要么全部成功，事务要保持原子性，**现实项目中用的最多的事务类型**，对应到Spring传播机制中就是REQUIRED

#### mysql 实现

```SQL
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

#### Spring 实现 

 ```Java
 //Spring 默认的事务传播机制，propagation=TransactionDefinition.PROPAGATION_REQUIRED 可以省略
 @Transactional(propagation=TransactionDefinition.PROPAGATION_REQUIRED)
 public void tranfrom(){
   Account account1 = queryAccount(1);
   Account account2 = queryAccount(2);
   update(account1)
   update(account2)
 }
 ```

然而现实项目有的项目的



### 带有保存点的扁平事务

在 MySQL 中，你可以使用保存点（Savepoint）来实现扁平事务（Flat Transaction），这样可以在事务内部设置多个保存点，并在需要的情况下回滚到指定的保存点，而不必回滚整个事务，下面是一个带有保存点的扁平事务的例子。**发生异常时撤销其中的一部分操作**



#### Mysql 实现

```SQL
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

#### Spring 实现

可以使用着两句来实现事务的保存点，手动控制事务的流程

```Java
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

### 链式事务

- 链事务可视为保存点模式的一种变种，带有保存点的扁平事务，当发生系统崩溃时，所有的保存点都将消失，因为其保存点是易失的（volatile），而非持久的( persistent)。这意味着当进行恢复时，事务需要从开始处重新执行，而不能从最近的一个保存点继续执行
  链事务的思想是：在提交一个事务时，释放不需要的数据对象，将必要的处理上下文隐式地传给下一个要开始的事务

注意，提交事务操作和开始下一个事务操作将合并为一个原子操作。这意味着下一个事务将看到上一个事务的结果，就好像在一个事务中进行的一样。下图显示了链事务的工作方式：

#### mysql 实现（不支持）

mysql不直接支持链式事务

#### Spring实现

```java
@Transactional(propagation=TransactionDefinition.PROPAGATION_REQUIRES_NEW)
```

### 嵌套事务

嵌套事务是指在一个事务内部可以再开启一个新的子事务，形成了多层嵌套的事务结构。在嵌套事务中，子事务可以独立于父事务进行提交或回滚，而不会影响父事务的提交或回滚。嵌套事务允许在一个事务内部有更细粒度的事务控制，对于复杂的业务场景和数据操作，可以提供更灵活和精确的事务管理。

嵌套事务的特点包括：

1. 嵌套事务形成多层结构：一个事务内部可以开启另一个事务，形成多个层次的事务结构。
2. 子事务独立于父事务：子事务可以独立于父事务进行提交或回滚，子事务的提交或回滚不会影响父事务的状态。
3. 事务传播行为：在嵌套事务中，需要定义事务传播行为，指定子事务如何与父事务进行交互。

需要注意的是，嵌套事务并不是所有数据库都原生支持的特性，具体的支持程度取决于数据库的类型和版本。在某些数据库中，嵌套事务被转换成普通的非嵌套事务，即嵌套事务的提交或回滚会被转化为整个事务的提交或回滚。

#### mysql 实现（不支持）

mysql不直接支持链式事务

#### Spring实现

[PROPAGATION_NESTED](### PROPAGATION_NESTED)

```Java
@Transactional(propagation=TransactionDefinition.PROPAGATION_NESTED)
```

### 分布式事务



