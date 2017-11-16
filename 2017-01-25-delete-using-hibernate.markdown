---
layout: post
title: "使用 Hibernate 实现软删除"
date: 2017-01-25 11:03:14 +0800
comments: true
categories: [hibernate, soft delete] 
---
当我们做项目的时候，我们经常会遇到这样的需求：用户希望当删除数据时并不真正的从物理文件中删除记录而是将记录打上标记，下次再查询的时候就只查那些没有标记为删除的记录，也就是我们常说的“假删除”。之所以这样做是因为用户想要尽可能地保留数据的历史记录，供将来追溯，或现有的记录与其他表的数据存在关联关系，如果删除的话会导致数据完整性问题。一般来说，实现软删除的思路有 2 种，第一种是将要删除的记录写入到审计日志或转移到其他的表中，这样做有一定的局限性，以后查看不方便，甚至还要与其他表重新建立关联关系，比较麻烦；另一种方法就是为表增加一个额外的字段，比如是布尔、枚举来表示数据的状态。下面要讲的就是使用 Hibernate 实现第二种方式的软删除。

要实现软删除，我们需要克服 2 个困难：

* 当遇到删除操作时，我们需要让 Hibernate 不生成 DELETE 语句，而是生成 UPDATE 语句并更新相应的状态字段
* 当执行任何查询时，我们要让 Hibernate 自动根据标志字段过滤出数据，因为我们的系统通常会有大量的查询方法，如果没有一种自动机制，那么我们需要为所有查询方法都添加过滤条件，显然很麻烦也很容易出错，万一漏写了就会暴露了数据。

下面就通过代码来演示下如何实现软删除，演示环境是：JPA/Hibernate 5.2.6.Final，MySQL 5.6.26，项目具体配置如下:

```xml
<dependencies>
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-core</artifactId>
        <version>5.2.6.Final</version>
    </dependency>
    <dependency>
       <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.40</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.22</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.1.8</version>
    </dependency>
</dependencies>
```

首先我们先定义一个简单的实体 Student 类：
```java
@Entity
@Table(name = "student")
public class Student {
    /**
     * 主键
     */
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    /**
     * 姓名
     */
    @Column(name = "name_", length = 20)
    private String name;
    /**
     * 是否删除(标记位)
     */
    @Column(name = "is_deleted_")
    private Boolean isDeleted;
}
```

上面代码中，出于方便的目的我们让主键自增，该实体还有另外 2 个简单的字段，分别是姓名和删除标记位，其中 isDeleted 字段默认是 false 的，现在我们想要实现的目标是当我们使用 JPA 删除一个学生的记录时，生成的 SQL 语句不是 DELETE 而是更新 is_deleted_ 字段并将其更新为 true。
想要实现上面的目标，我们就需要重新实现 Hibernate 的删除操作，而 Hibernate 为我们提供了一个注解 @SQLDelete，我们只需通过注解指定进行删除操作时要执行的原生 SQL，这个 SQL 就会覆盖 Hibernate 原来生成的 DELETE 语句从而实现重写删除操作的目的，具体实现方式请看下面的代码：

```java
@Entity
@Table(name = "student")
@SQLDelete(sql = "update student set is_deleted_ = true where id=?", check = ResultCheckStyle.COUNT)
public class Student {...}
```

上面的代码中我们在 Student 实体上使用了 @SQLDelete 注解并设置了一个原生 SQL 用于更新状态位，下面我们就试验下是否能实现软删除：

```java
EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory("pu");
EntityManager entityManager = entityManagerFactory.createEntityManager();
entityManager.getTransaction().begin();
Student student = entityManager.find(Student.class, 3);
entityManager.remove(student);
entityManager.getTransaction().commit();
logger.info("{}", student);
entityManager.close();
entityManagerFactory.close();
```

上面的代码中我们先查询出 ID 为 3 的记录，然后删除并提交事务，最后再打印出刚才查询出来的记录，下面是 Hibernate 生成的日志：
```text
16:47:28.717 [main] DEBUG org.hibernate.SQL - select student0_.id as id1_0_0_, student0_.is_deleted_ as is_delet2_0_0_, student0_.name_ as name_3_0_0_ from student student0_ where student0_.id=?
16:47:28.735 [main] TRACE org.hibernate.type.descriptor.sql.BasicBinder - binding parameter [1] as [INTEGER] - [3]
16:47:28.748 [main] TRACE org.hibernate.type.descriptor.sql.BasicExtractor - extracted value ([is_delet2_0_0_] : [BOOLEAN]) - [false]
16:47:28.748 [main] TRACE org.hibernate.type.descriptor.sql.BasicExtractor - extracted value ([name_3_0_0_] : [VARCHAR]) - [abc]
16:47:28.773 [main] DEBUG org.hibernate.SQL - update student set is_deleted_ = true where id=?
16:47:28.774 [main] TRACE org.hibernate.type.descriptor.sql.BasicBinder - binding parameter [1] as [INTEGER] - [3]
16:47:28.780 [main] INFO  org.mingzhe.test.SoftDeleteTest - Student{id=3, name='abc', isDeleted=false}
```

从第 5 行日志中我们可以看出删除操作变成了我们指定的更新语句，说明我们的目的实现了，但是又出现了新问题，大家可以看到最后一行日志显示我们实体管理器中 student 的 isDeleted 字段仍然为 fasle。这是为什么呢？这是因为我们执行了原生 SQL，而Hibernate 是无法探测到原生语句到底对哪些记录进行了更改，所以也就无法对其上下文中管理的实体状态进行更新，所以我们还需改进一下，让受管的实体状态得到更新。针对这个问题，比较优雅的方式就是使用 JPA 的生命周期方法，我们在 Student 类中定义一个方法并用 @PostRemove 注解，告诉 Hibernate 当执行完删除操作后更新实体的状态，代码如下：

```java
@Entity
@Table(name = "student")
@SQLDelete(sql = "update student set is_deleted_ = true where id=?", check = ResultCheckStyle.COUNT)
public class Student {
    @PostRemove
    public void delete() {
        this.isDeleted = true;
    }
}
```

从上面的代码中我们可以看出，当执行完删除操作有 JPA 会自动调用 delete() 方法将删除的实体的 isDeleted 字段设置为 true

下面我们要解决的就是如何在执行查询时自动排除那些已经标记为删除的记录。这次我们还是使用一个 Hibernate 私有的注解——@Where，这个注解用于在每次生成的 WHERE 语句后面都自动加上我们指定的过滤条件，下面代码演示了通过这个注解根据状态字段来自动过滤数据：

```java
@Entity
@Table(name = "student")
@SQLDelete(sql = "update student set is_deleted_ = true where id=?", check = ResultCheckStyle.COUNT)
@Where(clause = “is_deleted_  != true”)
public class Student {...}
```

上面的代码中我们在实体类上使用了 @Where 注解，并指定了过滤条件为 is_deleted_ 不等于 true，每次查询时这个条件都会自动加到 where 语句后面，下面我们进行一次查询，看看是否能得到正确的结果：

```java
EntityManagerFactory entityManagerFactory = Persistence.createEntityManagerFactory("pu");
EntityManager entityManager = entityManagerFactory.createEntityManager();
entityManager.getTransaction().begin();
Student student1 = entityManager.find(Student.class, 3);
logger.info("{}", student1);
Student student2 = entityManager.find(Student.class, 4);
logger.info("{}", student2);
entityManager.getTransaction().commit();
entityManager.close();
entityManagerFactory.close();
```

上面的代码中，我们分别查询了 id 为 3,4 的记录，并打印查询结果，其中 id 为 3 的记录删除标记位已设置为 true，所以打印出 student1 为 null，而 student2 则正常查询出来，说明我们的设置起作用了，下面是生成的日志：

```text
08:25:50.864 [main] DEBUG org.hibernate.SQL - select student0_.id as id1_0_0_, student0_.is_deleted_ as is_delet2_0_0_, student0_.name_ as name_3_0_0_ from student student0_ where student0_.id=? and ( student0_.is_deleted_ != 1)
08:25:50.885 [main] TRACE org.hibernate.type.descriptor.sql.BasicBinder - binding parameter [1] as [INTEGER] - [3]
08:25:50.896 [main] INFO  org.mingzhe.test.SoftDeleteTest - null
08:25:50.897 [main] DEBUG org.hibernate.SQL - select student0_.id as id1_0_0_, student0_.is_deleted_ as is_delet2_0_0_, student0_.name_ as name_3_0_0_ from student student0_ where student0_.id=? and ( student0_.is_deleted_ != 1)
08:25:50.897 [main] TRACE org.hibernate.type.descriptor.sql.BasicBinder - binding parameter [1] as [INTEGER] - [4]
08:25:50.918 [main] TRACE org.hibernate.type.descriptor.sql.BasicExtractor - extracted value ([is_delet2_0_0_] : [BOOLEAN]) - [false]
08:25:50.920 [main] TRACE org.hibernate.type.descriptor.sql.BasicExtractor - extracted value ([name_3_0_0_] : [VARCHAR]) - [张三]
08:25:50.925 [main] INFO  org.mingzhe.test.SoftDeleteTest - Student{id=4, name='张三', isDeleted=false}
```

从日志中可以印证在执行查询过程中 Hibernate 会自动为生成的语句添加上我们指定的过滤条件。

有同学可能会问如果我想要查询所有记录该如何呢？这种情况下我们可以定义一个专门的方法，用原生 SQL 来查询出来所有的记录。本文讲解的使用 Hibernate 来实现软删除其实就是使用了 Hibernate 私有的注解，虽然可以很优雅的完成目标，但也存在一些弊端，如不可移植，如果大家有更好的方法，欢迎指教。

