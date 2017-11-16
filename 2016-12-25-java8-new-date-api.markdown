---
layout: post
title: "Java 8 新特性之日期、时间 API"
date: 2016-12-25 13:51:09 +0800
comments: true
categories: [java] 
---
早在Java 8 之前，JDK 为我们提供了 Date 和 Calendar 这两个类来操作日期和时间，从使用角度来看，这两个类的 API 设计简直是反人类，所以在我们开始讲解新API之前，我们先看看已有的 API 存在什么问题，请看下面的代码：
```java
Date date = new Date(12, 12, 12);
System.out.println(date);
```

你能说出上面的代码会打印出什么吗？很多程序员会说 “0012-12-12”，但实际上会打印出“Sun Jan 12 00:00:00 IST 1913”，是不是与期望的相差很多，这说明这个 API 从设计上来说对程序员很不友好，使用起来容易产生很多不必要的麻烦，下面我们来分析下上面的代码都有哪些问题：

* 代码中每个 12 都是什么意思？是年月日、日月年还是其他的组合？
* 月份是从 0 开始的，如果想要设置为 12 月那么参数应该写成 11，当写成 12 时又会从 1 月开始，所以打印的结果是 1 月
* 现有的 API 年份是从 1900 年开始计算的，而月份写成 12 后会进位，所以打印的结果年份是 1900 + 12 + 1 = 1913
* 我们在 new Date() 的过程中只传入了年月日，而打印出来的还有时分秒以及时区，这些信息对我们来说都是多余的

此外，用于解析、格式化日期或时间的 DateFormat 类只能处理 Date 类型，无法处理时间类型且还是非线程安全的，上面只是列举了几个反例而已，实际使用起来还有不少坑要踩，总之用两个字来总结，那就是——难用！

由于存在这些难用的日期 API，大家伙儿都转到了第三方库上，比如 Joda-Time。这让 Oracle 颜面有些挂不住了，于是奋发图强，找来了一批牛人，在吸取了 Joda-Time 的不少设计上的优点以及分析了以前 API 中存在的问题后重新设计了一套原生日期时间 API，这些 API 都位于 java.time 包下。设计者们在设计这些 API 时应用了 领域驱动(domain-driven design)和不可变(Immutability)的设计原则使得 API 简洁容易理解。注意，由于所有位于 java.time 包下的核心类都是不可变的，也就是说对这些类的对象进行的所有操作都会创建一个新的对象，并不会修改原有对象的值，这就避免了线程安全的问题。下面我们将通过代码快速讲解如何去使用它们。
<!--more-->

### LocalDate, LocalTime, LocalDateTime, Instant, Duration, Period

上面这 6 个类是日期时间 API 的核心类，掌握了它们相当于学会了这套 API 的一半。

**LocalDate** 对象只能用于表示日期，即年、月、日，时分秒都表示不了，并且也不保存任何时区相关的信息。创建 LocalDate 的实例可以通过调用 LocalDate 的 4 个静态工厂方法：
```java
LocalDate.of(int year,int month,int day);
LocalDate.of(int year,Month month,int day);
LocalDate.ofYearDay(int year,int day);
LocalDate.ofEpochDay(long day);
LocalDate.now();
```

前 2 个方法是传入的是年月日，只不过第 2 个方法使用了 Month 枚举，让代码更易读，前两个方法的效果都是一样的。第 3 个方法用于获取某年的第几天的日期，比如 2016 年第 59 天是 2016-02-28；第 4 个方法是计算从 Unix 元年( 1970-01-01)开始后第几天的日期，如 Unix 元年后的第 1000 天的日期是 1972-09-27；第 5 个方法是通过系统时钟来获取当期的日期，很便捷。

```java
System.out.println(LocalDate.of(2016, 12, 6));             //2016-12-06
System.out.println(LocalDate.of(2016, Month.DECEMBER,6));  //2016-12-06
System.out.println(LocalDate.ofYearDay(2016,59));          //2016-02-28
System.out.println(LocalDate.ofEpochDay(1000));
```

LocalDate 自身也提供了一些很便利的方法供我们使用：
```java
LocalDate date = LocalDate.of(2016, 12, 6);
System.out.println(date.getYear());       //2016
System.out.println(date.getMonth());      //DECEMBER
System.out.println(date.getDayOfMonth()); //6
System.out.println(date.getDayOfWeek());  //TUESDAY
System.out.println(date.lengthOfMonth()); //31
System.out.println(date.lengthOfYear());  //366
System.out.println(date.isLeapYear());    //true
```

除了使用 LocalDate 提供的便捷的方法用于获取日期中各个字段的值，其实我们还可以通过更通用的 get() 方法来获取年月日这些字段：
```java
LocalDate date = LocalDate.of(2016, 12, 6);
System.out.println(date.get(ChronoField.YEAR));          //2016
System.out.println(date.get(ChronoField.MONTH_OF_YEAR)); //12
System.out.println(date.get(ChronoField.DAY_OF_MONTH));  //6
```

get() 方法接收 TemporalField 类型的参数，TemporalField 是一个接口，定义了如何访问某个时间对象中的字段。ChronoField 枚举类实现了此接口，所以我们只需要通过传入相应的枚举就能清晰的表达出取值的意图。

注意，由于 LocalDate 只包含年月日字段，如果我们传入的枚举值是时分秒的话就会抛出异常：
```java
System.out.println(date.get(ChronoField.HOUR_OF_DAY)); //UnsupportedTemporalTypeException: Unsupported field: HourOfDay
```

** LocalTime** 对象只能用于表示时间，即时、分、秒，无法表示年、月、日，并且也不保存任何时区相关的信息。创建 LocalTime 的实例和 LocalDate 类似，也是通过调用静态工厂方法：
```java
System.out.println(LocalTime.of(13,56));                //13:56
System.out.println(LocalTime.of(13,56,26));             //13:56:26
System.out.println(LocalTime.of(13,56,26,1000));        //13:56:26.000001
System.out.println(LocalTime.ofSecondOfDay(36000));     //10:00
System.out.println(LocalTime.ofNanoOfDay(1000000000));  //00:00:01
System.out.println(LocalTime.now()); 
```

上面代码中前 3 个方法传入的值依次为时、分、秒、纳秒；第 4 个方法用于计算一天中过去多少秒后的时间，如新的一天在过去 36000 秒后的时间是 10:00；第 5 个方法是用于计算一天中过去多少纳秒后的时间，如新的一天在过去 1000000000 秒后的时间是 00:00:01；第 6 个方法通过系统时钟来获取当前的时间，使用起来很便捷。

LocalTime 也提供了用于获取时间对象中各个字段的方法，示例如下：
```java
LocalTime localTime = LocalTime.now();
System.out.println(localTime.getHour());
System.out.println(localTime.getMinute());
System.out.println(localTime.getSecond());
System.out.println(localTime.getNano());
```

LocalTime 也可以通过 get() 方法来获取各个字段，但注意的是传入的 ChronoField 枚举值不是时、分、秒、纳秒的话也会抛出异常。

LocalDate 和 LocalTime 都支持通过字符串解析来创建相应的对象：
```java
System.out.println(LocalDate.parse("2016-12-06")); //2016-12-06
System.out.println(LocalTime.parse("14:22:26"));   //14:22:26
```

其中 parse() 方法还支持传入 DateTimeFormatter 对象，用于指定解析日期的格式，JDK 默认为我们提供了很过的格式，具体用法查询下文档即可，此处不在赘述。

LocalDateTime 是将 LocalDate 和 LocalTime 组合起来，能表示年月日时分秒纳秒，但仍然不能保存时区信息。所有以 Local 开头的时间类都无法保存时区信息，否则 Local 不是会显的很多余吗？LocalDateTime 的性质和上面 2 个类一样都是不可变的，任何操作都会创建新的对象

创建 LocalDateTime 的方法也有很多，很灵活：
```java
// 方法1
System.out.println(LocalDateTime.now()); //2016-12-06T15:13:30.516
// 方法2
System.out.println(LocalDateTime.of(2016, Month.DECEMBER, 6, 15, 6, 56, 1000)); //2016-12-06T15:06:56.000001
// 方法3
LocalDate date = LocalDate.now();
LocalTime time = LocalTime.now();
System.out.println(LocalDateTime.of(date, time));//2016-12-06T15:13:30.518
// 方法4
System.out.println(date.atTime(15, 12, 36));//2016-12-06T15:12:36
// 方法5
System.out.println(time.atDate(LocalDate.of(2016, 12, 6)));//2016-12-06T15:13:30.518
```

比较有意思的是后面 2 个方法，LocalDate 设置 time 会返回 LocalDateTime，LocalTime 设置 date 也会返回 LocalDateTime。LocalDateTime 也提供了各种方法用于获取各个时间字段的值，同样也支持通用的 get() 方法，传入 ChronoField 枚举来访问，不再赘述。

Instant 类使用一个大整数用于表示自 Unix 元年(1970年1月1日)过去的秒数，不适合人类阅读，它存在的意义就是给机器使用。其精度能达到纳秒级别，通过静态方法 ofEpochSecond() 来创建实例对象，也可以通过静态方法 now() 来获取当前的时间戳：
```java
System.out.println(Instant.ofEpochSecond(99999999)); //1973-03-03T09:46:39Z
System.out.println(Instant.now());//2016-12-06T08:20:29.349Z
```

由于 Instant 对象中只存储了秒和纳秒信息，所以我们是无法获取其他时间字段的值，比如下面代码想要通过 Instant 对象获取小时数就会抛异常：
```java
System.out.println(Instant.now().get(ChronoField.HOUR_OF_DAY));
//UnsupportedTemporalTypeException: Unsupported field: HourOfDay
```

到目前为止，上面讲的所有的类都实现了 Temporal 接口，表示的是某个时间点，而 **Duration** 类用来表示一段时间，即两个时间点间的差。可以通过调用 Duration.between() 静态方法来计算两个时间点的差，但需要注意的是这个方法只能用来计算 LocalTime，LocalDateTime 和 Instant 间的差，**不能用与计算两个 LocalDate 的差**，因为 Duration 也是由秒和纳秒组成的，而 LocalDate 无法精确到秒所以不能参与计算。此外，由于 LocalDateTime 和 Instant 都能精确到纳秒，但它们的用途不同，前者用于方便人类阅读，而后者使用大整数便于机器计算，所以不能将二者混用，否则会抛异常。

```java
Duration d1 = Duration.between(time1, time2);
Duration d1 = Duration.between(dateTime1, dateTime2);
Duration d2 = Duration.between(instant1, instant2);
```

Duration 类只能用日、时、分、秒或纳秒来表示时间间隔，如果我们想要用年、月、日来表示时间间隔的话，那就需要 Period 类了。下面演示如何计算两个日期的差:
```java
Period period = Period.between(LocalDate.of(2016,8,19),LocalDate.of(2016,11,16));
System.out.println(period); //P2M28D，表示日期差为2个月28天
```

Duration 和 Period 类除了通过调用 between() 方法来创建实例外，它们也提供了 of() 静态工厂方法来直接创建实例而不用额外的去创建 2 个时间对象，如下所示：
```java
Duration threeMinutes = Duration.ofMinutes(3);
Duration.of(3, ChronoUnit.MINUTES);
Period tenDays = Period.ofDays(10);
Period threeWeeks = Period.ofWeeks(3);
Period twoYearsSixMonthsOneDay = Period.of(2, 6, 1);
```

### 计算、解析、格式化日期

上一节只是讲解了如何创建日期对象，但在开发过程中可没这么简单，我们更多的时候需要对它们进行某些计算，比如计算当前日期 2 天后是星期几等这类问题，所以下面我们将讲解如何根据日期对象来进行计算。

操纵一个日期对象最容易也是最直接的方法就是调用 withAttribute() 方法了，此方法会返回一个全新的对象，而不会在原来对象的基础上进行修改，请看下面的代码：
```java
LocalDate date1 = LocalDate.of(2015,12,25); // 2015-12-25
LocalDate date2 = date1.withYear(2016);  // 2016-12-25
LocalDate date3 = date2.withDayOfMonth(24); // 2016-12-24
LocalDate date4 = date3.with(ChronoField.MONTH_OF_YEAR,11); // 2016-11-24
```

从上面的代码中可以看出，我们通过调用 withXXX 方法就可以单独设置日期对象中的某些字段了，而最后一个 with 方法属于一个更通用的方法，我们只需传入 ChronoField 枚举即可，但是如果操作的日期对象不包含该枚举字段的话将会抛出异常。

此外，我们还可以使用声明式的编程风格来操作日期，以 LocalDate 为例，请看下面的代码：
```java
LocalDate date1 = LocalDate.of(2016,12,24); // 2016-12-24
LocalDate date2 = date1.plusDays(1); // 2016-12-25
LocalDate date3 = date2.minusYears(1); // 2015-12-25
LocalDate date4 = date3.plus(12, ChronoUnit.MONTHS); // 2016-12-25
```

从上面的代码中可以看出，我们不再调用 with 方法了，而是使用更具有语义的方法就能修改对象的某个字段，最后的方法也是一个更通用的方法，传入想要修改的字段及数据即可。

下面出个考题考考大家，请运用刚才讲的知识来实现下面的需求：

假设小明的生日是 1990 年 1 月 7 日，那么他 27 岁的生日那天是星期几？

```java
LocalDate birthday = LocalDate.of(1990, 1, 7);
birthday.plusYears(27).getDayOfWeek(); // SATURDAY
```

是不是很简单。有时我们需要进行更高级的日期操作，比如将某个日期调整到某个月最后一天、或下一个工作日等，这就需要进行某些逻辑判断，而非简单的进行加减，所以新的日期 API 增加了 TemporalAdjuster 用于帮我们进行复制的日期操作，其中 JDK 内置了很多的 adjuster，如果没有符合我们需要的，我们还可以实现 TemporalAdjuster 接口以满足业务需要，这里不再详解了，大家可以参考网上的资料学习，都很简单。

解析和格式化日期是对日期操作的另一个重要组成部分，java.time.format 包下的类就用于此目的。其中最重要的类就是 DateTimeFormatter 了，可以通过此类的静态方法来创建对象。下面演示下如何格式化和解析日期对象：
```java
LocalDate now = LocalDate.now();
System.out.println(now.format(DateTimeFormatter.ISO_DATE));
System.out.println(LocalDate.parse("2016-12-25",DateTimeFormatter.ISO_DATE));
```

从上面的代码中可以看出，我们只要在需要进行格式化的日期对象上调用 format() 方法，传入要格式化的模式即可，而解析也很简单，调用类的静态方法 parse()，传入一个日期格式的字符串，以及解析模式就能还原出日期对象。其中 DateTimeFormatter 内置了很多种日期模式，假如没有我们需要的，我们可以自定义解析格式，请看下面代码：

```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy/MM/dd");
LocalDate date1 = LocalDate.of(2016, 12, 25);
String formattedDate = date1.format(formatter);
System.out.println(formattedDate); // 2016/12/25
LocalDate date2 = LocalDate.parse(formattedDate, formatter);
System.out.println(date2); // 2016-12-25
```

上面的代码中，我们定义了一个 DateTimeFormatter 对象，调用 ofPattern 方法为其指定了解析模式为“yyyy/MM/dd”，然后我们用此模式格式化和解析日期，最终都能正确的结果。其中 ofPattern 还有一个重载的方法用于支持本地化，如：
```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy/MMMM/dd", Locale.CHINA);
LocalDate date1 = LocalDate.of(2016, 12, 25);
System.out.println(date1.format(formatter)); // 2016/十二月/25
```

在 ofPattern 方法中，我们还传入了第二个参数 Locale.CHINA，当我们再格式化日期是，就能看到原来的月份变成了“十二月”。

至此，Java 8 中增加的对日期的新特性就基本讲完了，总之新的 api 给开发者着实带来了开发效率上的提高，从难以使用到现在的傻瓜化，Java 8 确实是 Java 发展中的一个重要里程碑，其影响力不亚于当年 Java 5 发布时泛型、注解给程序员带来的冲击，而下一讲中我将介绍 Java 8 最重要的特性—— Lambda 表达式，它的到来将 Java 领入了函数式编程的世界，开启了 Java 世界的新篇章。
