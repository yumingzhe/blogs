---
layout: post
title: "JPA 处理 N+1 问题的办法"
date: 2017-06-20 17:43:59 +0800
comments: true
categories: [jpa, java, hibernate] 
---
### N+1 问题

在使用 JPA 开发过程中，项目中会存在大量的实体类，而实体间的关系通常有一对多、多对多的情况，我们通常将多的一方的加载方式设置成”延迟加载“，即当我们查询某个实体时只是查询出了实体的基本属性，与之相关联的记录并没有加载，这就会导致一个问题，当我们需要关联对象的某些属性时，就会触发 ORM 发出 SQL 语句与将关联的记录查询出来，这就是臭名昭著的 N+1 问题。举个例子，请看下面的代码：

```java
// 教师类
@Entity
@Table(name = "teacher")
public class Teacher {
   @Id
   @GeneratedValue(strategy = GenerationType.IDENTITY)
   private Integer id;
   @Column(name = "name_")
   private String name;
   public Teacher(String name) {
      this.name = name;
   }
   @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
   @JoinColumn(name = "teacher_id_")
   private List<Student> students;
}
// 学生类
@Entity
@Table(name = "student")
public class Student {
   @Id
   @GeneratedValue(strategy = GenerationType.IDENTITY)
   private Integer id;
   @Column(name = "name_")
   private String name;
}
```

上面的代码中包含了两个非常简单的实体：老师和学生，老师和学生是单向一对多关系的。

当我们查询出某个老师的记录后，再访问其“学生”属性，就会产生一条 SQL 语句去查询，代码如下：

```java
// 创建EM
EntityManagerFactory factory = Persistence.createEntityManagerFactory("hibernate");
entityManager = factory.createEntityManager();
entityManager.getTransaction().begin();
// n+1查询
Teacher teacher = entityManager.find(Teacher.class, 1);
List<Student> students = teacher.getStudents();
for (Student student : students) {
    System.out.println(student);
}
// 关闭资源
entityManager.getTransaction().commit();
entityManager.close();
factory.close();
```

上面的代码会产生下面的 SQL 日志：
```text
10135 Query select teacher0_.id as id1_1_0_, teacher0_.name_ as name_2_1_0_ from teacher teacher0_ where teacher0_.id=1
10135 Query select students0_.teacher_id_ as teacher_3_0_0_, students0_.id as id1_0_0_, students0_.id as id1_0_1_, students0_.name_ as name_2_0_1_ from student students0_ where students0_.teacher_id_=1
```

从日志中可以看出产生了 2 条SQL，设想一下，假如我们先通过一条语句查询出 100 个老师的记录，然后分别访问每个老师所关联的学生，那么将会再产生 100 条查询学生记录的 SQL 语句，一共 1+100 条语句，这就会给应用的性能带来极大的损耗，光是 100 条语句在应用和数据库间的通讯就会消耗大量的时间，更别说再进行复杂的业务数据处理所花费的时间了。如果没有 ORM 框架，我们可以用原生SQL通过一条 join 语句就可以将 100 个老师及其关联的学生记录查询出来，那么使用 ORM 框架将如何避免这类问题呢，下面我们就是用 JPA 2.1 提供的特性来处理 N+1 问题，测试环境为 JPA 2.1(Hibernate 5.2.10.Final) + MySQL 5.6，代码依赖如下：

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.2.10.Final</version>
</dependency>
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-entitymanager</artifactId>
    <version>5.2.10.Final</version>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.43</version>
</dependency>
```

JPA 提供了 2 种方式用于处理延迟加载问题，一个是“fetch join”，另一个是 EntityGraph，当然我们也可以调用原生 SQL 来实现，不过就丧失了可移植性，下面我们就分别讲解这两种技术：

<!--more-->

### Fetch Join

Fetch Join 是 JPA 提供的一种简便语法，用于立即抓取实体相关联的对象或集合属性，通常这些对象或集合都是延迟加载的，fetch join 可以是内连接也可以是外连接，语法如下：
```text
fetch_join ::= [ LEFT [ OUTER ] | INNER ] JOIN FETCH join_association_path_expression
```

下面我们用代码来演示下 fetch join，修复上面的 N+1 问题：
```java
Teacher teacher1 = entityManager.createQuery("select t from Teacher as t join fetch t.students where t.id = 1", Teacher.class).getSingleResult();
for (Student student : teacher1.getStudents()) {
   System.out.println(student);
}
```

产生是 SQL 日志如下：
```text
10152 Query select teacher0_.id as id1_1_0_, students1_.id as id1_0_1_, teacher0_.name_ as name_2_1_0_, students1_.name_ as name_2_0_1_, students1_.teacher_id_ as teacher_3_0_0__, students1_.id as id1_0_0__ from teacher teacher0_ inner join student students1_ on teacher0_.id=students1_.teacher_id_ where teacher0_.id=1
```

从日志可以看出这次查询就只产生了 1 条语句，并且是内连接。如果我们想用外连接，只要把 JPQL 改成下面语句即可：

```java
select t from Teacher as t left outer join fetch t.students where t.id = 1
```

### EntityGraph(实体图)

为了演示实体图的用法，我们再添加一个实体——University，实体间的关系如下图所示：

{% img http://image-1255444646.cossh.myqcloud.com/jpa_n_plus_one/diagram.png %}

从图中可以看出，一个大学可以有多个老师，每个老师又可以教授多个学生。现在我们想要查询某个大学中，所有老师及其教授学生的信息，按照传统的面向对象的思路一般都会写成下面的代码：

```java
University university = entityManager.find(University.class, 3);
for (Teacher teacher : university.getTeachers()) {
   teacher.getStudents().forEach(System.out::println);
}
```

上面的代码产生的 SQL 日志如下，很明显是 N+1 问题：
```text
10181 Query select university0_.id as id1_2_0_, university0_.address_ as address_2_2_0_, university0_.name_ as name_3_2_0_ from university university0_ where university0_.id=3
10181 Query select teachers0_.university_id_ as universi3_1_0_, teachers0_.id as id1_1_0_, teachers0_.id as id1_1_1_, teachers0_.name_ as name_2_1_1_ from teacher teachers0_ where teachers0_.university_id_=3
10181 Query select students0_.teacher_id_ as teacher_3_0_0_, students0_.id as id1_0_0_, students0_.id as id1_0_1_, students0_.name_ as name_2_0_1_ from student students0_ where students0_.teacher_id_=3
```

下面我们就用实体图的方式来解决多层延迟加载的问题。首先讲一下实体图的概念，实体图是 jpa 2.1 新增加的特性，**由于实体间的关系错综复杂，如果实体间存在关系那么就用线将实体进行连接，那么实体间将形成一个网状的图，而实体图技术就是对这个关系图的操作，它允许开发者指定某个实体及其发展出来的关系网中的某些节点，这些节点间形成一条路径，当调用查询方法时，查询方法会按照实体图中的路径将路径中的这些节点立即加载，而绕过延迟加载，从而更高效的实现数据的检索**。在大中型项目中，几乎所有的关联关系都被定义为延迟加载，而由于业务需求不同，造成了对实体的查询深度的要求，因此为了避免 N+1 问题，在数据访问层通常会写出很多的查询方法，如 findEntityWithX(), findEntityWithY(), findEntityWithXandY()等等。**有了实体图技术，我们就可以动态的构建查询计划，绕过延迟加载**。实体图有 2 种使用方式，一种是使用注解的方式进行定义，另一个是通过编程的方式动态构建查询计划，下面我们使用注解的方式来改写上面的例子：

```java
@Entity
@Table(name = "university")
@NamedEntityGraph(
      name = "universityWithTeacherAndStudent",
      attributeNodes = {@NamedAttributeNode(value = "teachers", subgraph = "teacherWithStudent")},
      subgraphs = {
            @NamedSubgraph(name = "teacherWithStudent", attributeNodes = {@NamedAttributeNode("students")})
      }
)
public class University {……}
```

通过注解的方式我们在实体上用 @NamedEntityGraph 来定义一个实体图，name属性指定一个实体图的名称，attributeNodes 属性指定要立即加载的节点，节点用 @NamedAttributeNode 定义，其中 values 表示节点的属性名称，在本例中是 teachers 这个属性，而如果还想加载teacher节点下的students属性的话，需要再为该节点定义一个子图，子图用 @NamedSubgraph 来定义并用subgraph来引用，方式与 @NamedEntityGraph 一样，这样我们就定义了一个以 university 为起点，到 teacher 再到student 的路径，下一步就是调用查询方法，并将实体图传给查询方法，让查询方法按照我们构造的路径进行数据的加载，代码如下：

```java
Map<String, Object> hints = new HashMap<String, Object>();
hints.put("javax.persistence.fetchgraph", entityManager.getEntityGraph("universityWithTeacherAndStudent"));
University university = entityManager.find(University.class, 3, hints);
for (Teacher teacher : university.getTeachers()) {
   teacher.getStudents().forEach(System.out::println);
}
```

上面的代码 1、2 行，我们首先构建了一个 JPA 的 hint，hint 用于告知底层的 JPA 实现类即 Hibernate 我们要使用某些特性了，请予以支持，如果不支持的话请忽略。hint的key我们指定为“javax.persistence.fetchgraph”，也可以指定为“javax.persistence.loadgraph”，前者作用是立即加载指定的节点，未指定的节点延迟加载；后者是立即加载指定的节点，未指定的节点按实体中定义的加载策略进行加载，hint 的 value 为实体图，我们通过 em.getEntityGraph() 方法来获取。第3行代码，我们调用查询时，将 hint 传给 find 方法，至此就完成了实体图的查询方式。

运行代码后，会产生如下日志：

```text
10185 Query select university0_.id as id1_2_0_, university0_.address_ as address_2_2_0_, university0_.name_ as name_3_2_0_, teachers1_.university_id_ as universi3_1_1_, teachers1_.id as id1_1_1_, teachers1_.id as id1_1_2_, teachers1_.name_ as name_2_1_2_, students2_.teacher_id_ as teacher_3_0_3_, students2_.id as id1_0_3_, students2_.id as id1_0_4_, students2_.name_ as name_2_0_4_ from university university0_ left outer join teacher teachers1_ on university0_.id=teachers1_.university_id_ left outer join student students2_ on teachers1_.id=students2_.teacher_id_ where university0_.id=3
```

可以看出只需要一条语句即可查询出 3 个关联表的数据，避免了 N+1 问题。

### 后记

N+1 问题在项目初期很难被发现，当项目上线后，随着数据量的暴增，性能问题才会暴露出来，所以我们在编码的过程中，脑中要有一定的意识，提前估算下将来的数据量，问问自己这样写将来会不会产生过多的 SQL，这样才不会为将来留下坑。本文只是简单的介绍了两种用于处理延迟加载问题的方法，对于这两种技术的运用还有很多实验需要进行，感兴趣的朋友可以自己尝试模拟一个复杂的场景来进行数据抓取。本文写作如有纰漏，欢迎指正。

