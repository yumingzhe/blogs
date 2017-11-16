---
layout: post
title: "Hiberntae 5 新特性之一次查询多个实体"
date: 2017-04-20 13:12:22 +0800
comments: true
categories: [jpa, hiberate, java]
---
在使用 JPA 的过程中，我们可能会经常遇到这样的场景：我们要通过多个主键查询实体，可是遗憾的是，JPA 并未定义这样的接口允许我们通过主键加载多实体。在 Hibernate 5 之前，我们可以通过下面 2 个方法进行曲线救国：

1.通过在一个循环里多次调用 EntityManager.find() 方法，一次查询一个实体，缺点是会生成较多的 SQL，查询的时间跟查询数量成正比，速度比较慢，时间全浪费在与数据库的通信上了

2.创建一个JPQL语句，将所有的主键放到 IN 语句中，如下面代码所示：

```java
List<Long> ids = Arrays.asList(new Long[]{1L, 2L, 3L});
List<PersonEntity> persons = em.createQuery("SELECT p FROM Person p WHERE p.id IN :ids").setParameter("ids", ids).getResultList();
```

此方法虽然在性能上没啥问题，但仍有以下缺点：

* 1.有些数据库如 Oracle 对 IN 子句的参数个数有限制，如果传入的主键数量太多，会产生 SQL 语法级别的错误
* 2.如果传入的主键数据量较大，那么一次抓取可能会产生巨大的流量，造成系统性能颠簸
* 3.在使用这种方式时，Hibernate 根据 JPQL 生成 SQL 并从数据库中加载所有实体时，并不会检查一级缓存(session)中是否已经缓存某些实体，重复加载对象

上面提到的这些问题我们都可以通过代码控制得以解决，但这些代码都是非业务逻辑的代码，将会分散在所有需要性能考虑的地方，增加了代码的复杂度、难以维护。因此，Hibernate 5.1 引入了新的 API 扩展了原有 Session 的功能，让我们可以通过简单的调用接口即可轻松的实现一次加载多个实体并且还解决了性能相关的问题。 下面我们将演示新的 API 用于一次加载多个实体，我演示的环境是：JPA 2.1 + Hibernate 5.2.2。

首先，我们假设有一个实体，叫做 Student，如果想要一次通过主键加载多个 Student 的记录的话，可以通过下面的代码实现：

```java
EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory("pu");
EntityManager entityManager = entityManagerFactory.createEntityManager();
// 将em解封为底层实现，即hibernate的session
Session session = entityManager.unwrap(Session.class);
session.getTransaction().begin();
// 新api，可以一次加载多个实体
List<Student> students = session.byMultipleIds(Student.class).multiLoad(2,3,4,5,6);
System.out.println(students.size()); // 输出5，说明新api确实能够实现一次多实体加载
session.getTransaction().commit();
entityManager.close();
entityManagerFactory.close();
```

从上面的代码可以看出，我们调用 session 的 byMultipleIds()方法，并提供实体的类作为参数，会返回一个类型，然后再调用 multiLoad() 方法，并传入主键即可实现一次加载多个实体的效果。此时，Hibernate 生成的 SQL 和上面方案 2 中 JPQL 生成的语句一样，主键都放在了 in 子句中，如下所示：

```text
14:32:57,602 DEBUG SQL:92 – select personenti0_.id as id1_0_0_, personenti0_.firstName as firstNam2_0_0_, personenti0_.lastName as lastName3_0_0_ from Person personenti0_ where personenti0_.id in (?,?,?)
```

从上面生成的结果可以看出，Hibernate 生成的 SQL 和我们自己手写 JPQL 生成的 SQL 并无不同，但我们只是提供了少量的 id，如果 id 数量变大，就需要考虑批处理了。

### 批量加载实体

批量加载有如下好处：

不是所有数据库都允许 IN 子句有无限个参数；为了将内存占用量控制在一定范围内，我们希望在加载下一批次的实体前要从1级缓存中移除上一批次的对象；我们想要在业务逻辑中检测是否加载的实体是我们需要的。

Hibernate 默认的批大小是与我们配置的数据库方言(dialect)相关联的，因此我们不必担心生成的sql语句会违反数据库的限制。但是 Hibernate 仍为我们提供了接口用于更改批大小，示例如下：

```java
List<PersonEntity> persons = session.byMultipleIds(PersonEntity.class).withBatchSize(2).multiLoad(1L, 2L, 3L);
```

从下面生成的日志可以看出，如果我们传入的 id 数量大于批大小，那么 Hibernate 会生成多个 select 语句：
```text
15:20:52,314 DEBUG SQL:92 – select personenti0_.id as id1_0_0_, personenti0_.firstName as firstNam2_0_0_, personenti0_.lastName as lastName3_0_0_ from Person personenti0_ where personenti0_.id in (?,?)
15:20:52,331 DEBUG SQL:92 – select personenti0_.id as id1_0_0_, personenti0_.firstName as firstNam2_0_0_, personenti0_.lastName as lastName3_0_0_ from Person personenti0_ where personenti0_.id in (?)
```

### 不加载已经缓存的实体

如果我们通过 JPQL 的方式批量加载实体，Hibernate 是无法检测实体是否已经存在于 session (一级缓存)中，因此可能会导致不必要的加载。由于 Hibernate 引入了新的 MultiIdentifierLoadAccess 接口，可以让 Hibernate 在加载前检测实体是否已经处于缓存中，但是这个特性默认是关闭的，我们要通过 enableSessionCheck(boolean enabled) 开启，示例代码如下：
```java
PersonEntity p = em.find(PersonEntity.class, 1L);
log.info("Fetched PersonEntity with id 1");
Session session = em.unwrap(Session.class);
List<PersonEntity> persons = session.byMultipleIds(PersonEntity.class).enableSessionCheck(true).multiLoad(1L, 2L, 3L);
```
日志如下：
```text
15:34:07,449 DEBUG SQL:92 – select personenti0_.id as id1_0_0_, personenti0_.firstName as firstNam2_0_0_, personenti0_.lastName as lastName3_0_0_ from Person personenti0_ where personenti0_.id=?
15:34:07,471 INFO TestMultiLoad:118 – Fetched PersonEntity with id 1
15:34:07,476 DEBUG SQL:92 – select personenti0_.id as id1_0_0_, personenti0_.firstName as firstNam2_0_0_, personenti0_.lastName as lastName3_0_0_ from Person personenti0_ where personenti0_.id in (?,?)
```

从代码中可以看出我们首先加载了 id 为 1 的实体，然后我们再批量加载 id 为 1,2,3 的实体，但是 Hibernate 检测到 id 为 1 的实体已经在缓存中了，生成的 SQL 则只加载剩余的两个实体。

### 总结
在开发中按主键批量加载实体情况还是比较常见的，我们可以简单的通过 JPQL 进行实现，但要考虑生成的 SQL 是否能符合数据库的限制，以及在性能上是否存在问题。Hibernate 引入的 MultiIdentifierLoadAccess 接口为开发人员提供了开箱即用的功能，让我们能通过简单的调用 API 即可实现性能优良的功能，把关注点更多的放在业务上而不是技术实现上。
