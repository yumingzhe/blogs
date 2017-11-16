---
layout: post
title: "为什么 Spring 里 DAO 是单例的"
date: 2016-11-26 10:43:33 +0800
comments: true
categories: [java, spring] 
---
当我们使用 Spring 框架进行企业级软件开发时，系统通常采用分层架构的方案，其中一层就是数据访问层即 DAO层，所以我们经常会写出下面的代码：
```java
// Dao 类
@Repository
public class StudentDao{
    @PersistenceContext
    private EntityManager em;

    public void save(Student student){
        em.persist(student);
    }
}

// Service 层
@Service
@Transactional
public class StudentServiceImpl impl StudentService{
    @Autowired
    private StudentDao dao;

    public void saveStudent(Student student){
        dao.save(student);
    }
}
```

上面的代码使用了 Spring + JPA 来表示，众所周知 Service 和 Dao 类都是单例的，当多个请求到达控制器时就会创建多个线程并发执行 service 对象，service 对象内部有一个字段 dao(这个字段属于一种状态变量)，而 dao 对象内部又注入了一个实体管理器对象 em，我们都知道 EntityManager 是线程不安全的，所以当多个线程执行 service 对象，就相当于多个线程在并发调用 dao 对象，进而相当于多个线程在并发调用 em 对象，那么这个 em 对象中管理的持久化数据就会不断地变化，甚至相互干扰冲突，进而可能造成意料不到的结果。那么既然存在线程安全问题，那么为什么我们按照这种模式编写代码却不会出错呢？

原因在于**当使用 Spring 时，在 Dao 类中注入的 EntityManager 并不是真正的实体管理器，而是一个代理，也就是说每次我们调用 em 的方法时，都被代理对象给拦截并做了一些预处理后才把方法调用委托给真正的实体管理器去处理**。下面我们通过解析源码的方式来回答这个问题。

<!--more-->
为了便于理解，我们先复习下如何使用 Java 来创建代理。Java 提供了 Proxy 这个类用于创建任何类的代理，假设我们想要创建 Foo 类的一个代理对象，可以使用下面的方法：

```java
Foo f = (Foo) Proxy.newProxyInstance(Foo.class.getClassLoader(), new Class[] { Foo.class }, handler);
```

上面的代码中 f 对象就是一个代理对象，最后一个参数 handler 是一个实现了InvocationHandler 接口的对象，当调用 f 对象上的任何方法时都会先触发 handler 对象中的 invoke(Object proxy, Method method, Object[] args)  方法的执行，所以在执行真正的方法前后我们都可以通过在 invoke 方法中添加某些业务逻辑代码来实现对原有功能的增强，与 AOP 很类似。

由于注入到 Dao 中的 em 是代理，那么在 em 上调用任何方法都会被 handler 方法拦截，那么找到真正的 EntityManager 肯定就在 handler 方法内部。所以，经过一番查找，在 org.springframework.org.jpa.SharedEntityManagerCreator 这个类里终于发现了蛛丝马迹，这个类实现了 InvocationHandler 接口并且在 invoke() 方法内部有这么一句代码：

```java
EntityManager target = EntityManagerFactoryUtils.doGetTransactionalEntityManager(...);
```

看来这才是真正的 em，但是这个 em 到底是如何创建的呢？我们再进入到 doGetTransactionalEntityManager() 这个方法里，会发现下面的代码：

```java
EntityManagerHolder emHolder = (EntityManagerHolder) TransactionSynchronizationManager.getResource(emf);
```

再进入到 getResource() 方法内部，我们可以发现 resource 其实是一个 ThreadLocal 类型的 map 对象，所以一切都能说通了，原来 Spring 首先会通过 EntityManagerFactory 获取 EntityManager 对象，然后将它存入到当前线程中的一个变量中即 resource 里，然后当多线程并发调用 em 代理时，代理从当前的线程中取得真正的 em 再将操作委托给它执行，所以也就不存在 em 上下文线程安全的问题了。

现在我们终于弄清了为啥 dao 可以是单例的，因为 Spring 内部使用了 代理和 ThreadLocal 机制将线程不安全的 EntityManager 绑定到了每个线程上，所以当多个线程并发访问 dao 对象时，其实都是分别使用线程自己的 em 对象，这样就扫除了线程安全的问题。
