---
layout: post
title: "开启Spring事务机制"
description: ""
tag: "transaction"
date: "2016-08-04 00:00:00 +0800"
categories: "SpringBoot"
---

#### 什么是事务？

在企业级开发过程中，对于与无人员来说一个实际的对数据库的操作可能是多步结合进行的。
<!--more--> 
由于对数据库的操作在任何一个步骤都有可能会发生异常，异常会导致后续操作无法完成，可是此时业务逻辑没有正确完成，之前的操作数据也就不可靠，需要在这种情况下对之前的操作的数据进行回滚。  

回滚是为了保证用户每一步的操作都是可靠的，事务中的每一步操作都必须正确完成，只要有一个异常的发生都需要回退到开始的未进行的状态。  

事务管理也是Spring框架中最为常用的功能之一。


#### 快速入门

在Spring Boot中，当我们使用了`spring-boot-starter-jdbc`或`spring-boot-starter-data-jpa`依赖的时候，框架会自动默认分别注入  `DataSourceTransactionManager`或`JpaTransactionManager`。所以我们不需要任何额外配置就可以用`@Transactional`注解进行事务的使用。


通常在单元测试中，我们只是想要测试业务逻辑的正确与否，并不想保存测试数据。  
```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(Application.class)
public class ApplicationTests {

    @Autowired
    private UserRepository userRepository;

    @Test
    public void test() throws Exception {

        // 创建10条记录
        userRepository.save(new User("AAA", 10));
        userRepository.save(new User("BBB", 20));
        userRepository.save(new User("CCC", 30));
        userRepository.save(new User("DDD", 40));
        userRepository.save(new User("EEE", 50));
        userRepository.save(new User("FFF", 60));
        userRepository.save(new User("GGG", 70));
        userRepository.save(new User("HHH", 80));
        userRepository.save(new User("III", 90));
        userRepository.save(new User("JJJ", 100));

        // 省略后续的一些验证操作
    }


}
```

可以看到，在这个单元测试用例中，使用UserRepository对象连续创建了10个User实体到数据库中，下面我们人为的来制造一些异常，看看会发生什么情况。

通过定义User的name属性长度为5，这样通过创建时User实体的name属性超长就可以触发异常产生。

```
@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false, length = 5)
    private String name;

    @Column(nullable = false)
    private Integer age;

    // 省略构造函数、getter和setter

}
```

修改测试用例中创建记录的语句，将一条记录的name长度超过5，如下：name为HHHHHHHHH的User对象将会抛出异常。  

```
// 创建10条记录
userRepository.save(new User("AAA", 10));  
userRepository.save(new User("BBB", 20));  
userRepository.save(new User("CCC", 30));  
userRepository.save(new User("DDD", 40));  
userRepository.save(new User("EEE", 50));  
userRepository.save(new User("FFF", 60));  
userRepository.save(new User("GGG", 70));  
userRepository.save(new User("HHHHHHHHHH", 80));  
userRepository.save(new User("III", 90));  
userRepository.save(new User("JJJ", 100));
```

执行测试用例，可以看到控制台中抛出了如下异常，name字段超长：  

```
2016-05-27 10:30:35.948  WARN 2660 --- [           main] o.h.engine.jdbc.spi.SqlExceptionHelper   : SQL Error: 1406, SQLState: 22001  
2016-05-27 10:30:35.948 ERROR 2660 --- [           main] o.h.engine.jdbc.spi.SqlExceptionHelper   : Data truncation: Data too long for column 'name' at row 1  
2016-05-27 10:30:35.951  WARN 2660 --- [           main] o.h.engine.jdbc.spi.SqlExceptionHelper   : SQL Warning Code: 1406, SQLState: HY000  
2016-05-27 10:30:35.951  WARN 2660 --- [           main] o.h.engine.jdbc.spi.SqlExceptionHelper   : Data too long for column 'name' at row 1

org.springframework.dao.DataIntegrityViolationException: could not execute statement; SQL [n/a]; nested exception is org.hibernate.exception.DataException: could not execute statement  
```

此时查数据库中，创建了name从AAA到GGG的记录，没有HHHHHHHHHH、III、JJJ的记录。而若这是一个希望保证完整性操作的情况下，AAA到GGG的记录希望能在发生异常的时候被回退，这时候就可以使用事务让它实现回退，做法非常简单，我们只需要在test函数上添加`@Transactional`注解即可。

```
@Test
@Transactional
public void test() throws Exception {

    // 省略测试内容

}
```

#### 事务详解  
上面的例子中我们使用了默认的事务配置，可以满足一些基本的事务需求，但是当我们项目较大较复杂时（比如，有多个数据源等），这时候需要在声明事务时，指定不同的事务管理器。对于不同数据源的事务管理配置可以见《Spring Boot多数据源配置与使用》中的设置。在声明事务时，只需要通过value属性指定配置的事务管理器名即可，例如：@Transactional(value="transactionManagerPrimary")。

除了指定不同的事务管理器之后，还能对事务进行隔离级别和传播行为的控制，下面分别详细解释：  

##### 隔离级别  

隔离级别是指若干个并发的事务之间的隔离程度，与我们开发时候主要相关的场景包括：脏读取、重复读、幻读。

我们可以看`org.springframework.transaction.annotation.Isolation`枚举类中定义了五个表示隔离级别的值：  
```
public enum Isolation {  
    DEFAULT(-1),
    READ_UNCOMMITTED(1),
    READ_COMMITTED(2),
    REPEATABLE_READ(4),
    SERIALIZABLE(8);
}
```

* `DEFAULT`：这是默认值，表示使用底层数据库的默认隔离级别。对大部分数据库而言，通常这值就是：READ_COMMITTED。
* `READ_UNCOMMITTED`：该隔离级别表示一个事务可以读取另一个事务修改但还没有提交的数据。该级别不能防止脏读和不可重复读，因此很少使用该隔离级别。  
* `READ_COMMITTED`：该隔离级别表示一个事务只能读取另一个事务已经提交的数据。该级别可以防止脏读，这也是大多数情况下的推荐值。   
* `REPEATABLE_READ`：该隔离级别表示一个事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。即使在多次查询之间有新增的数据满足该查询，这些新增的记录也会被忽略。该级别可以防止脏读和不可重复读。  
* `SERIALIZABLE`：所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。  

指定方法：通过使用`isolation`属性设置，例如：  

```
@Transactional(propagation = Propagation.REQUIRED)
```

#### 实现回滚的三种方式 

##### 1.直接抛出异常

发生异常时在catch中直接通过throw抛出异常，因为Spring事务默认接收到抛出异常时回滚，从`@Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)`中的配置也可以看出。

##### 2.抛出RunTimeException

或者在catch中通过自己手动`throw new RunTimeException();`抛出异常。

##### 3.TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();

通过这种放发可以不需要抛出异常直接回滚。在前台需要返回异常信息时用这种方法就可以起到很好的作用。