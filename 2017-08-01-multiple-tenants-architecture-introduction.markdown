---
layout: post
title: "多租户架构浅析"
date: 2017-08-01 17:32:37 +0800
comments: true
categories: [multi-tenant, saas] 
---
众所周知，云计算可以划分为以下几个层次的服务——IaaS、PaaS和SaaS，而今天我们今天讲的多租户架构就是一种常见的 SaaS 软件架构模式，或者说是商业模式。
通常，一个多租户软件指的是依托云计算的弹性环境，搭建并使用一个单一的应用程序实例来服务多个客户，每个客户称之为“租户”来共享同一个软件，很像现在很火的共享经济，客户们都只是来租用系统，按时收费，客户是不需要提供或关心软件的运行环境，只要开通账号即可使用，方便快捷。举个例子，假如我们推出了一款财务软件，可以为企业提供财务方面的管理功能，按照传统的部署方法，我们通常将程序、数据库等组件都部署在客户的服务器上，这需要企业有自己的机房和服务器并配备有相关专业知识的运维人员，但如果程序或其他组件出现 bug 等严重故障时，我们就需要派遣工程师到客户现场进行救援恢复工作，耗时耗力，而有了以云计算为依托的多租户架构软件，那么我们就可以将这套财务软件部署在自己的机房内，只要让“租户”注册账号，通过互联网即可访问系统，同时“租户”们的数据相互隔离，只能看到自己的数据，软件的安全性也得到了保障。最重要的是，当系统出现故障时，工程师可以快速定位问题，并将最新的补丁应用于系统上，将“租户”服务中断时间压缩到最小，而不用花费大量的时间到现场进行排错，同时也不必派遣工程师出差，为企业运维降低了成本。
由此可见，多租户架构的软件是云计算时代软件研发的一个重要方向，不仅对客户提供了便捷，也节省了企业的成本，是一个双赢的方案。

上面有句话可能不太严谨，通常只有小型的多租户程序才只部署一个单一的运行实例，这种情况只适用于租户较少、服务器压力负载低的用例，此时租户的费用支出较为低廉，适合预算有限的客户。但这种部署方案存在缺点就是由于多个租户公用一套程序，一旦程序有  bug，那么所有的租户都会受到影响，且没法为单独租户提供定制，灵活性低，不过一分价钱一分货。其部署架构如下图所示：

{% img http://image-1255444646.cossh.myqcloud.com/multi-tenant/multi-tenant-2.png %}

而对于中大型规模的应用来说，只部署一个实例就略显单薄，并且也无法承载大量的用户，这时就需要部署多个实例，并在多实例的基础上进行负载均衡以保证性能。最重要的是，多实例的部署架构允许为每个租户定制代码，提供特色服务，其部署架构如下图所示：


{% img http://image-1255444646.cossh.myqcloud.com/multi-tenant/multi-tenant-1.png %}

由于大型的多实例方案中需要加入负载均衡以及受商业、业务相关的因素的影响，会导致在多租户架构之外包裹这更复杂的业务设计，因此本文就只讨论单实例方案下的多租户架构。从上面的介绍可知，在传统的部署方案中，客户数据始终是保留在自己的手中，而在云计算时代，多租户系统为用户提供了一个集中式的数据存储方案，这需要说服用户将他们的数据保留在云端，因此多租户系统最为关键也是客户最关心的便是数据的安全性，因此本文对多租户系统的架构分析也是从数据的安全性方面作为切入点。由于用户数据是集中存储的，数据的安全性就是能否实现对数据的隔离，防止数据不经意或被他人恶意地获取，多租户系统架构主要有3种方案实现数据的隔离，即为每个租户提供独立的数据库、独立的表空间或按字段区分租户，下面我们依次讲解这3种方案。

<!--more-->

### 一、每个租户提供独立的数据库

这种方案就是所有租户共享同一个应用，但应用后端会连接多个数据库，每个租户一个数据库，这样租户间的数据就实现了物理隔离，其部署架构图如下所示：


{% img http://image-1255444646.cossh.myqcloud.com/multi-tenant/multi-tenant-5.png %}

从上图可以看出，每个租户都有自己独立的数据库，因此对每个租户的数据进行备份和还原是很容易的，但是此种方案存在一个弊端就是租户的开销比较大，相比于另外2种方案，开销主要耗费在硬件方面，为了支持多数据库实例及保持数据库较高的性能，我们可能需要将数据库部署到单独的服务器上，随着租户数量不断的递增，那么开销也会成倍的增长，导致运营成本的增加，但是这种方案却是3种方案中数据隔离性最高的一种，虽然这种隔离仍然属于逻辑上的隔离，通常使用这种方案的租户对安全的要求非常高，比如像金融、医疗客户，他们要求数据要有很强的隔离并且对支出也能负担的起。需要注意的是，在3种方案种，多租户应用负责将用户请求路由到不同的数据源上，路由的依据可以在用户请求头上做某些识别，如发送的请求头可以构造成下面的格式：

```text
> GET /index HTTP/1.1
> Host: localhost:8080
> User-Agent: Mozilla/5.0
> Accept: */*
> X-TENANT-ID: tenant_1
```

后台接收到请求后根据 X-TENANT-ID 字段来识别用户，也可以通过url模式来识别租户，如为每个租户都设置一个独立的URL模式，如 tenant1 用户的 url 为 tenant1.app.com，或者最直接的方式就是在用户登录后将其用户信息保存在 session 中，每次请求从 session 中获取用户 ID。

### 二、每个租户提供独立的表空间

本方案就是所有租户共享同一个应用，应用后端只连接一个数据库，每个租户在数据库实例中拥有一个独立的表空间，其部署架构图如下所示： 

{% img http://image-1255444646.cossh.myqcloud.com/multi-tenant/multi-tenant-4.png %}

从上图可以看出，这种设计应用后端只连接了一个数据库实例，数据库实例中存在多个表空间，每个表空间属于不同的租户，但每个表空间内都存在相同的表，一般适用于表空间小于 100 张表的系统，这种方案无论是对于用户还是对运维来说都是一个比较好的这种方案，因为从硬件方面来说一般只需要一台性能足够强劲的服务器就可以支撑上百用户，适合预算有限但又对数据隔离有较大需求的客户，从维护方面看，对每个租户的数据库进行备份和恢复也非常简单，缺点可能就是备份出来的文件数量比较多，需要额外的进行管理。

### 三、按字段区分租户

本方案是多租户方案中最简单的设计模式了，几乎所有的信息系统或多或少都用过这种设计，即在每张表中都添加一个字段（如租户id或租户代码）来标识每条数据属于哪个租户，很像外键的作用，当进行查询的时候每条语句都要添加该字段作为过滤条件，其特点是所有租户的数据全都存放在同一个表中，数据的隔离性是最低的，完全是通过字段来区分的，适用于预算及其有限的、对数据隔离要求不高及数据量不大的低端用户。这种设计方式相较于前两种在成本上是最廉价的，但维护起来很麻烦，比如要对用户进行数据备份，就会把所有租户的数据都备份，想要单独备份某个租户的数据需要特殊处理，而要恢复数据也要所有租户的数据一起恢复，想要恢复某个用户的数据需要将用户数据单独提取出来再恢复，操作起来很麻烦，不具有实际的可操作性，其设计架构如下所示：

{% img http://image-1255444646.cossh.myqcloud.com/multi-tenant/multi-tenant-3.png %}

从上面的图中可以看出所有租户的数据都位于同一个表空间中，表空间中存放着租户共享表，每个表中通过“tenant_id”字段来区分不同租户的数据，这种模式虽然是最简单但却丧失了为租户可定制的可能性，如果要对表进行扩展那么所有租户都要受到影响。

四、总结

以上就是多租户架构模式常见的三种方案，它们各有优缺点，而在实际中到底采用哪种设计方案，要与企业所面向的用户群体、业务量等方面相结合才能决定采用哪种方案是最合理也是最经济的，下面就从不同的方面对三种方案进行归纳：
数据隔离性 	可定制性 	可承载租户数 	租户开销 	可维护性(备份恢复) 	运营成本
独立的数据库 	高 	高 	少 	高 	优 	高
独立的表空间 	中 	中 	多 	中 	优 	中
按字段区分租户 	低 	低 	较多 	低 	中 	低

以上基本讲解了多租户的架构模式，下一讲我将用代码的方式实现一个“独立的表空间”的多租户系统，因为这种方案在保证数据隔离的情况下，对企业来说也是最经济的，大部分都采用这种方案，敬请关注。

参考文档：https://docs.microsoft.com/en-us/azure/sql-database/sql-database-design-patterns-multi-tenancy-saas-applications
