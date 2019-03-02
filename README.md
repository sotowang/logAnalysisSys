# 以spark为基础电商项目

## 用户访问session分析模块
* 在实际企业项目中的使用架构：

  1、J2EE的平台（美观的前端页面），通过这个J2EE平台可以让使用者，提交各种各样的分析任务，其中就包括一个模块，就是用户访问session分析模块；

  可以指定各种各样的筛选条件，比如年龄范围、职业、城市等等。

  2、J2EE平台接收到了执行统计分析任务的请求之后，会调用底层的封装了spark-submit的shell脚本（Runtime、Process），shell脚本进而提交我们编写的Spark作业。

  3、Spark作业获取使用者指定的筛选参数，然后运行复杂的作业逻辑，进行该模块的统计和分析。

  4、Spark作业统计和分析的结果，会写入MySQL中，指定的表

  5、最后，J2EE平台，使用者可以通过前端页面（美观），以表格、图表的形式展示和查看MySQL中存储的该统计分析任务的结果数据。

* 用户访问session介绍：

1. 用户在电商网站上，通常会有很多的点击行为，首页通常都是进入首页；然后可能点击首页上的一些商品；点击首页上的一些品类；也可能随时在搜索框里面搜索关键词；还可能将一些商品加入购物车；对购物车中的多个商品下订单；最后对订单中的多个商品进行支付。
2. 用户的每一次操作，其实可以理解为一个action，比如点击、搜索、下单、支付
3. 用户session，指的就是，从用户第一次进入首页，session就开始了。然后在一定时间范围内，直到最后操作完（可能做了几十次、甚至上百次操作）。离开网站，关闭浏览器，或者长时间没有做操作；那么session就结束了。以上用户在网站内的访问过程，就称之为一次session。简单理解，session就是某一天某一个时间段内，某个用户对网站从打开/进入，到做了大量操作，到最后关闭浏览器。的过程。就叫做session。
4. session实际上就是一个电商网站中最基本的数据和大数据。那么大数据，面向C端，也就是customer，消费者，用户端的，分析，基本是最基本的就是面向用户访问行为/用户访问session。

* 模块的目标：对用户访问session进行分析

  1、可以根据使用者指定的某些条件，筛选出指定的一些用户（有特定年龄、职业、城市）；

  2、对这些用户在指定日期范围内发起的session，进行聚合统计，比如，统计出访问时长在0-3s的session占总session数量的比例；

  3、按时间比例，比如一天有24个小时，其中12:00-13:00的session数量占当天总session数量的50%，当天总session数量是10000个，那么当天总共要抽取1000个session，ok，12:00-13:00的用户，就得抽取1000*50%=500。而且这500个需要随机抽取。

  4、获取点击量、下单量和支付量都排名10的商品种类

  5、获取top10的商品种类的点击数量排名前10的session

  6、开发完毕了以上功能之后，需要进行大量、复杂、高端、全套的性能调优

  7、十亿级数据量的troubleshooting（故障解决）的经验总结

  8、数据倾斜的完美解决方案

  9、使用mock（模拟）的数据，对模块进行调试、运行和演示效果

### 基础表结构 

* 表名：task（MySQL表）

```
task_id：表的主键
task_name：任务名称
create_time：创建时间
start_time：开始运行的时间
finish_time：结束运行的时间
task_type：任务类型，就是说，在一套大数据平台中，肯定会有各种不同类型的统计分析任务，比如说用户访问session分析任务，页面单跳转化率统计任务；所以这个字段就标识了每个任务的类型
task_status：任务状态，任务对应的就是一次Spark作业的运行，这里就标识了，Spark作业是新建，还没运行，还是正在运行，还是已经运行完毕
task_param：最最重要，用来使用JSON的格式，来封装用户提交的任务对应的特殊的筛选参数
```

task表，其实是用来保存平台的使用者，通过J2EE系统，提交的基于特定筛选参数的分析任务，的信息，就会通过J2EE系统保存到task表中来。之所以使用MySQL表，是因为J2EE系统是要实现快速的实时插入和查询的。

* 表名：user_visit_action（Hive表）

```
date：日期，代表这个用户点击行为是在哪一天发生的
user_id：代表这个点击行为是哪一个用户执行的
session_id ：唯一标识了某个用户的一个访问session
page_id ：点击了某些商品/品类，也可能是搜索了某个关键词，然后进入了某个页面，页面的id
action_time ：这个点击行为发生的时间点
search_keyword ：如果用户执行的是一个搜索行为，比如说在网站/app中，搜索了某个关键词，然后会跳转到商品列表页面；搜索的关键词
click_category_id ：可能是在网站首页，点击了某个品类（美食、电子设备、电脑）
click_product_id ：可能是在网站首页，或者是在商品列表页，点击了某个商品（比如呷哺呷哺火锅XX路店3人套餐、iphone 6s）
order_category_ids ：代表了可能将某些商品加入了购物车，然后一次性对购物车中的商品下了一个订单，这就代表了某次下单的行为中，有哪些
商品品类，可能有6个商品，但是就对应了2个品类，比如有3根火腿肠（食品品类），3个电池（日用品品类）
order_product_ids ：某次下单，具体对哪些商品下的订单
pay_category_ids ：代表的是，对某个订单，或者某几个订单，进行了一次支付的行为，对应了哪些品类
pay_product_ids：代表的，支付行为下，对应的哪些具体的商品
```

user_visit_action表，其实就是放，比如说网站，或者是app，每天的点击流的数据。可以理解为，用户对网站/app每点击一下，就会代表在这个表里面的一条数据。

* 表名：user_info（Hive表）

```
user_id：其实就是每一个用户的唯一标识，通常是自增长的Long类型，BigInt类型
username：是每个用户的登录名
name：每个用户自己的昵称、或者是真实姓名
age：用户的年龄
professional：用户的职业
city：用户所在的城市
```

user_info表，实际上，就是一张最普通的用户基础信息表；这张表里面，其实就是放置了网站/app所有的注册用户的信息。那么我们这里也是对用户信息表，进行了一定程度的简化。比如略去了手机号等这种数据。因为我们这个项目里不需要使用到某些数据。那么我们就保留一些最重要的数据，即可。

>  spark从MySQL表中读取任务参数，执行作业逻辑，持久化作业结果数据。



### 需求分析

* 1、按条件筛选session搜索过某些关键词的用户、访问时间在某个时间段内的用户、年龄在某个范围内的用户、职业在某个范围内的用户、所在某个城市的用户，发起的session。找到对应的这些用户的session，也就是我们所说的第一步，按条件筛选session。

  这个功能，就最大的作用就是灵活。也就是说，可以让使用者，对感兴趣的和关系的用户群体，进行后续各种复杂业务逻辑的统计和分析，那么拿到的结果数据，就是只是针对特殊用户群体的分析结果；而不是对所有用户进行分析的泛泛的分析结果。比如说，现在某个企业高层，就是想看到用户群体中，28-35岁的，老师职业的群体，对应的一些统计和分析的结果数据，从而辅助高管进行公司战略上的决策制定。

* 2、统计出符合条件的session中，访问时长在1s-3s、4s-6s、7s-9s、10s-30s、30s-60s、1m-3m、3m-10m、10m-30m、30m以上各个范围内的session占比；访问步长在1-3、4-6、7-9、10-30、30-60、60以上各个范围内的session占比

  session访问时长，也就是说一个session对应的开始的action，到结束的action，之间的时间范围；

  访问步长，指的是，一个session执行期间内，依次点击过多少个页面，比如说，一次session，维持了1分钟，那么访问时长就是1m，然后在这1分钟内，点击了10个页面，那么session的访问步长，就是10.

  比如说，符合第一步筛选出来的session的数量大概是有1000万个。那么里面，我们要计算出，访问时长在1s-3s内的session的数量，并除以符合条件的总session数量（比如1000万），比如是100万/1000万，那么1s-3s内的session占比就是10%。依次类推，这里说的统计，就是这个意思。

  **这个功能的作用，其实就是，可以让人从全局的角度看到，符合某些条件的用户群体，使用我们的产品的一些习惯。**比如大多数人，到底是会在产品中停留多长时间，大多数人，会在一次使用产品的过程中，访问多少个页面。那么对于使用者来说，有一个全局和清晰的认识。

* 3、在符合条件的session中，按照时间比例随机抽取1000个session

  这个按照时间比例是什么意思呢？随机抽取本身是很简单的，但是按照时间比例，就很复杂了。比如说，这一天总共有1000万的session。那么我现在总共要从这1000万session中，随机抽取出来1000个session。但是这个随机不是那么简单的。需要做到如下几点要求：首先，如果这一天的12:00-13:00的session数量是100万，那么这个小时的session占比就是1/10，那么这个小时中的100万的session，我们就要抽取1/10 * 1000 = 100个。然后再从这个小时的100万session中，随机抽取出100个session。以此类推，其他小时的抽取也是这样做。

  **这个功能的作用，是说，可以让使用者，能够对于符合条件的session，按照时间比例均匀的随机采样出1000个session，然后观察每个session具体的点击流/行为，比如先进入了首页、然后点击了食品品类、然后点击了雨润火腿肠商品、然后搜索了火腿肠罐头的关键词、接着对王中王火腿肠下了订单、最后对订单做了支付。**

  **之所以要做到按时间比例随机采用抽取，就是要做到，观察样本的公平性。**

* 4、在符合条件的session中，获取点击、下单和支付数量排名前10的品类

  什么意思呢，对于这些session，每个session可能都会对一些品类的商品进行点击、下单和支付等等行为。那么现在就需要获取这些session点击、下单和支付数量排名前10的最热门的品类。也就是说，要计算出所有这些session对各个品类的点击、下单和支付的次数，然后按照这三个属性进行排序，获取前10个品类。

  **这个功能，很重要，就可以让我们明白，就是符合条件的用户，他最感兴趣的商品是什么种类。这个可以让公司里的人，清晰地了解到不同层次、不同类型的用户的心理和喜好。**

* 5、对于排名前10的品类，分别获取其点击次数排名前10的session

  这个就是说，对于top10的品类，每一个都要获取对它点击次数排名前10的session。

  这个功能，可以让我们看到，对某个用户群体最感兴趣的品类，各个品类最感兴趣最典型的用户的session的行为。

### 技术方案设计

* 1、按条件筛选session

  这里首先提出第一个问题，你要按条件筛选session，但是这个筛选的粒度是不同的，比如说搜索词、访问时间，那么这个都是session粒度的，甚至是action粒度的；那么还有，就是针对用户的基础信息进行筛选，年龄、性别、职业。。所以说筛选粒度是不统一的。

  第二个问题，就是说，我们的每天的用户访问数据量是很大的，因为user_visit_action这个表，一行就代表了用户的一个行为，比如点击或者搜索；那么在国内一个大的电商企业里面，如果每天的活跃用户数量在千万级别的话。那么可以告诉大家，这个user_visit_action表，每天的数据量大概在至少5亿以上，在10亿左右。

  那么针对这个筛选粒度不统一的问题，以及数据量巨大（10亿/day），可能会有两个问题；首先第一个，就是，如果不统一筛选粒度的话，那么就必须得对所有的数据进行全量的扫描；第二个，就是全量扫描的话，量实在太大了，一天如果在10亿左右，那么10天呢（100亿），100呢，1000亿。量太大的话，会导致Spark作业的运行速度大幅度降低。极大的影响平台使用者的用户体验。

  所以为了解决这个问题，那么我们选择在这里，对原始的数据，进行聚合，什么粒度的聚合呢？session粒度的聚合。也就是说，用一些最基本的筛选条件，比如时间范围，从hive表中提取数据，然后呢，**按照session_id这个字段进行聚合，那么聚合后的一条记录，就是一个用户的某个session在指定时间内的访问的记录，比如搜索过的所有的关键词、点击过的所有的品类id、session对应的userid关联的用户的基础信息。**

  聚合过后，针对session粒度的数据，按照使用者指定的筛选条件，进行数据的筛选。筛选出来符合条件的用session粒度的数据。其实就是我们想要的那些session了。

* 2、聚合统计

  如果要做这个事情，那么首先要明确，我们的spark作业是分布式的。所以也就是说，每个spark task在执行我们的统计逻辑的时候，**可能就需要对一个全局的变量**，进行累加操作。比如代表访问时长在1s-3s的session数量，初始是0，然后呢分布式处理所有的session，判断每个session的访问时长，如果是1s-3s内的话，那么就给1s-3s内的session计数器，累加1。

  那么在spark中，要实现分布式安全的累加操作，基本上只有一个最好的选择，就是Accumulator变量。但是，问题又来了，如果是基础的Accumulator变量，那么可能需要将近20个Accumulator变量，1s-3s、4s-6s。。。。；但是这样的话，就会导致代码中充斥了大量的Accumulator变量，导致维护变得更加复杂，在修改代码的时候，很可能会导致错误。

  比如说判断出一个session访问时长在4s-6s，但是代码中不小心写了一个bug（由于Accumulator太多了），比如说，更新了1s-3s的范围的Accumulator变量。导致统计出错。所以，对于这个情况，那么我们就可以使用自定义Accumulator的技术，来实现复杂的分布式计算。也就是说，就用一个Accumulator，来计算所有的指标。

* 3、在符合条件的session中，按照时间比例随机抽取1000个session这个呢，需求上已经明确了。那么剩下的就是具体的实现了。具体的实现这里不多说，技术上来说，就是要综合运用Spark的countByKey、groupByKey、mapToPair等算子，来开发一个复杂的按时间比例随机均匀采样抽取的算法。（大数据算法）

* 4、在符合条件的session中，获取点击、下单和支付数量排名前10的品类

  这里的话呢，需要对每个品类的点击、下单和支付的数量都进行计算。然后呢，使用Spark的自定义Key二次排序算法的技术，来实现所有品类，按照三个字段，点击数量、下单数量、支付数量依次进行排序，首先比较点击数量，如果相同的话，那么比较下单数量，如果还是相同，那么比较支付数量。
* 5、对于排名前10的品类，分别获取其点击次数排名前10的session这个需求，需要使用Spark的分组取TopN的算法来进行实现。也就是说对排名前10的品类对应的数据，按照品类id进行分组，然后求出每组点击数量排名前10的session。

可以掌握到的技术点：

```
1、通过底层数据聚合，来减少spark作业处理数据量，从而提升spark作业的性能（从根本上提升spark性能的技巧）
2、自定义Accumulator实现复杂分布式计算的技术
3、Spark按时间比例随机抽取算法
4、Spark自定义key二次排序技术
5、Spark分组取TopN算法
6、通过Spark的各种功能和技术点，进行各种聚合、采样、排序、取TopN业务的实现
```

### JDBC原理

JDBC，Java Database Connectivity，Java数据库连接技术。

JDBC，其实只是代表了JDK提供的一套面向数据库的一套开发接口，注意，这里大部分仅仅是接口而已换句话说，你的Java应用程序，光有JDBC，是操作不了数据库的，更不用谈所谓的CRUD，增删改查。因为JDBC只是一套接口，接口，接口而已

**JDBC真正的意义在于通过接口统一了java程序对各种数据库的访问的规范**

数据库厂商提供的JDBC驱动，JDBC Driver。**即JDBC的实现类**

数据库厂商，比如说，MySQL公司，或者Oracle公司，会针对JDBC的一套接口，提供完整的一套接口的实现类在这套实现类中，不同的数据库厂商就实现了针对自己数据库的一套连接、执行SQL语句等等实际的功能

 实际上，在项目中，我们一般不会直接使用JDBC；而是会使用J2EE的一些开源框架，比如MyBatis，也可以是Hibernate而且为了方便框架的整合使用，我们通常都会在spark作业中，使用Spring开源框架，进行各种技术的整合 比如Kafka、Redis、ZooKeeper、Thrift

MyBatis/Hibernate这种操作数据库的框架，其实底层也是基于JDBC进行封装的，只不过提供了更加方便快捷的使用大大提升了我们的开发效率

总结一下JDBC的最基本的使用过程

```
1、加载驱动类：Class.forName()
2、获取数据库连接：DriverManager.getConnection()
3、创建SQL语句执行句柄：Connection.createStatement()
4、执行SQL语句：Statement.executeUpdate()
5、释放数据库连接资源：finally，Connection.close()
```

### 数据库连接池原理

每一次java程序要在MySQL中执行一条SQL语句，那么就必须建立一个Connection对象，代表了与MySQL数据库的连接。然后在通过连接发送了你要执行的SQL语句之后，就会调用Connection.close()来关闭和销毁与数据库的连接。

为什么要立即关闭呢？

**因为数据库的连接是一种很重的资源，代表了网络连接、IO等资源。所以如果不使用的话，就需要尽早关闭，以避免资源浪费。**

* 劣势/不足：

如果要频繁地操作MySQL的话，那么就势必会频繁地创建Connection对象，底层建立起与MySQL的占用了网络资源、IO资源的连接。此外呢，每次使用完Connection对象之后，都必须将Connection连接给关闭，又涉及到频繁的网络资源、IO资源的关闭和释放。

如上所述，如果频繁的开关Connection连接，那么会造成大量的对网络、IO资源的申请和释放的无谓的时间的耗费

对于特别频繁的数据库操作，比如100次/s，那么可能会导致性能急剧下降。

数据库连接池，**会自己在内部持有一定数量的数据库连接，比如通常可能是100-1000个左右。然后每次java程序要通过数据库连接往MySQL发送SQL语句的时候，都会从数据库连接池中获取一个连接，然后通过它发送SQL语句。SQL语句执行完之后，不会调用Connection.close()，而是将连接还回数据库连接池里面去。**

下一次，java程序再需要操作数据库的时候，就还是重复以上步骤，获取连接、发送SQL、还回连接。

* 数据库连接池的好处：

  1、java程序不用自己去管理Connection的创建和销毁，代码上更加方便。

  2、程序中只有固定数量的数据库连接，不会一下子变得很多，而且也不会进行销毁。那么对于短时间频繁进行数据库操作的业务来说。就有很高的意义和价值。也就是说，如果短时间内，频繁操作10000次，不需要对数据库连接创建和销毁10000次。这样的话，可以大幅度节省我们的数据库连接的创建和销毁的资源开销以及时间开销。

  3、最终可以提升整个应用程序的性能。

在spark作业中，是非常适合使用数据库连接池的，为什么呢？因此spark计算出来的结果，可能数据量还是会比较大的。比如说10万条。那么如果用普通的数据库操作方式，就必须创建和销毁数据库连接10万次，那么会大大降低整个spark作业的性能。数据库的操作变成整个spark作业的瓶颈。

如果可以善用数据库连接池的话，那么就大大节省数据库连接的创建和销毁的时间和性能开销。大大提升我们的spark作业的整体性能。

###  单例模式

* 单例模式是指的什么意思？

我们自己定义的类，其实默认情况下，都是可以让外界的代码随意创建任意多个实例的

但是有些时候，我们不希望外界来随意创建实例，而只是希望一个类，在整个程序运行期间，只有一个实例

任何外界代码，都不能随意创建实例



* 那么，要实现单例模式，有几个要点：

```
1、如果不想让外界可以随意创建实例，那么类的构造方法就必须用private修饰，必须是私有的

2、既然类的构造方法被私有化了，外界代码要想获取类的实例，不能够随意地去创建，那么就只能通过调用类的静态方法，去获取类的实例

3、所以类必须有一个静态方法，getInstance()，来提供获取唯一实例的功能。getInstance()方法，必须保证类的实例创建，且仅创建一次，返回一个唯一的实例
```



* 单例模式的应用场景有哪几个呢？

```
1、配置管理组件，可以在读取大量的配置信息之后，用单例模式的方式，就将配置信息仅仅保存在一个实例的实例变量中，这样可以避免对于静态不变的配置信息，反复多次的读取

2、JDBC辅助组件，全局就只有一个实例，实例中持有了一个内部的简单数据源使用了单例模式之后，就保证只有一个实例，那么数据源也只有一个，不会重复创建多次数据源（数据库连接池）
```

### 内部类及匿名内部类

外部类：最普通的，我们平时见到的那种类，就是在一个后缀为.java的文件中，直接定义的类，比如

```java
public class Student {  
    private String name;  
    private int age;
}
```

内部类：内部类，顾名思义，就是包含在外部类中的类，就叫做内部类。

内部类有两种，一种是静态内部类，一种是非静态内部类。

```java
public class School {  
	private static School instance = null;  
	static class Teacher {}
}

public class School {  
	private String name;  
    class Teacher {}
}
```

* 静态内部类和非静态内部类之间的区别主要如下：

* 1、内部原理的区别：

  静态内部类是属于外部类的类成员，是一种静态的成员，是属于类的，就有点类似于private static Singleton instance = null；

  非静态内部类，是属于外部类的实例对象的一个实例成员，也就是说，每个非静态内部类，不是属于外部类的，是属于外部类的每一个实例的，创建非静态内部类的实例以后，非静态内部类实例，是必须跟一个外部类的实例进行关联和有寄存关系的。

* 2、创建方式的区别：

  创建静态内部类的实例的时候，只要直接使用“外部类.内部类()”的方式，就可以，

  比如new School.Teacher()；

  创建非静态内部类的实例的时候，必须要先创建一个外部类的实例，然后通过外部类的实例，再来创建内部类的实例，new School().Teader()

  通常来说，我们一般都会为了方便，会选择使用静态内部类。

匿名内部类：

```java
public interface ISayHello {  
    String sayHello(String name);
}
public class SayHelloTest {    
    public static void main(String[] args) {    
        ISayHello obj = new ISayHello() {      
            public String sayHello(String name) { 
                return "hello, " + name 
            }    
        }    
        System.out.println(obj.sayHello("leo"))  
    }
}
```

匿名内部类的使用场景，通常来说，就是在一个内部类，只要创建一次，使用一次，以后就不再使用的情况下，就可以。

那么，此时，通常不会选择在外部创建一个类，而是选择直接创建一个实现了某个接口、或者继承了某个父类的内部类，而且通常是在方法内部，创建一个匿名内部类。

在使用java进行spark编程的时候，如果使用的是java7以及之前的版本，那么通常在对某个RDD执行算子，并传入算子的函数的时候，通常都会传入一个实现了某个Spark Java API中Function接口的匿名内部类。

### JavaBean

JavaBean，虽然就是一个类，但是是有特殊条件的一个类，不是所有的类都可以叫做JavaBean的首先，它需要有一些field，这些field，都必须用private来修饰，表示所有的field，都是私有化的，不能随意的获取和设置其次，需要给所有的field，都提供对应的setter和getter方法，什么叫setter和getter？setter，就是说setX()方法，用于给某个field设置值；getter，就是说getX()方法，用于对某个field获取值

```java
public class Student {
  
  private String name;
  private int age;

  public void setName(String name) {
    this.name = name;
  }
  public String getName() {
    return name;
  }
  public void setAge(int age) {
    this.age = age;
  }
  public int getAge() {
    return age;
  }

}

```

* JavaBean通常怎么用？

通常来说，会将一个JavaBean，与数据库中的某个表一一对应起来比如说，有一个student表，

```
create table student(name varchar(30), age integer)
```

那么这个表，如果要操作的话，通常来说，会在程序中，建立一个对应的JavaBean，这个JavaBean中，所有的field，都是和表中的字段一一对应起来的。

然后在执行增删改查操作的时候，其实都是面向JavaBean来操作的，比如insertStudent()方法，就应该接收一个参数，Student对象；

findAllStudent()方法，就应该将返回类型设置为List<Student>列表

* domain的概念：

在系统中，通常会分很多层，比如经典的三层架构，控制层、业务层、数据访问层（DAO层）

此外，还有一个层，就是domain层

domain层，通常就是用于放置这个系统中，与数据库中的表，一一对应起来的JavaBean的

三层架构+domain层+model层（J2EE web系统）

浏览器->后台->控制层->业务层->数据访问层->数据库 	 

domain->domain->domain->SQL		

domain/model<-domain和model可能都是JavaBean；**之间的区别，只是用途不太一样，domain通常就代表了与数据库表一一对应的JavaBean；model通常代表了不与数据库一一对应的JavaBean，但是封装的数据，是前端的JS脚本，需要使用的一些数据。**

### DAO模式

> Data Access Object：数据访问对象

引入了DAO模式以后，就大大降低了业务逻辑层和数据访问层的耦合，大大提升了后期的系统维护的效率，并降低了时间成本。我们自己在实现DAO模式的时候，通常来说，会将其分为两部分，

**一个是DAO接口；一个是DAO实现类。我们的业务的代码，通常就是面向接口进行编程；那么当接口的实现需要改变的时候，直接定义一个新的实现即可。但是对于我们的业务代码来说，只要面向接口开发就可以了。DAO的改动对业务代码应该没有任何的影响。**

### 工厂模式

* 如果没有工厂模式，可能会出现的问题：

ITaskDAO接口和TaskDAOImpl实现类；实现类是可能会更换的；那么，如果你就使用普通的方式来创建DAO，

比如ITaskDAO taskDAO = new TaskDAOImpl()

那么后续，如果你的TaskDAO的实现类变更了，那么你就必须在你的程序中，所有出现过TaskDAOImpl的地方，去更换掉这个实现类。这是非常非常麻烦的。

如果说，你的TaskDAOImpl这个类，在你的程序中出现了100次，那么你就需要修改100个地方。这对程序的维护是一场灾难。

* 工厂设计模式

对于一些种类的对象，使用一个工厂，来提供这些对象创建的方式，外界要使用某个类型的对象时，就直接通过工厂来获取即可。不用自己手动一个一个地方的去创建对应的对象。

那么，假使我们有100个地方用到了TaskDAOImpl。不需要去在100个地方都创建TaskDAOImpl()对象，只要在100个地方，都使用TaskFactory.getTaskDAO()方法，获取出来ITaskDAO接口类型的对象即可。

**如果后面，比如说MySQL迁移到Oracle，我们重新开发了一套TaskDAOImpl实现类，那么就直接在工厂方法中，更换掉这个类即可。不需要再所有使用到的地方都去修改。**

### JSON数据格式

* 什么是JSON？

就是一种数据格式；比如说，我们现在规定，有一个txt文本文件，用来存放一个班级的成绩；然后呢，我们规定，这个文本文件里的学生成绩的格式，是第一行，就是一行列头（姓名 班级 年级 科目 成绩），接下来，每一行就是一个学生的成绩。**那么，这个文本文件内的这种信息存放的格式，其实就是一种数据格式。**

```
学生 班级 年级 科目 成绩
张三 一班 大一 高数 90
李四 二班 大一 高数 80
```

对应到JSON，它其实也是代表了一种数据格式，所谓数据格式，就是数据组织的形式。比如说，刚才所说的学生成绩，用JSON格式来表示的话，如下：

```json
[{"学生":"张三", "班级":"一班", "年级":"大一", "科目":"高数", "成绩":90}, {"学生":"李四", "班级":"二班", "年级":"大一", "科目":"高数", "成绩":80}]
```

其实，JSON，很简单，一点都不复杂，就是对同样一批数据的，不同的一种数据表示的形式。JSON的数据语法，其实很简单：

**如果是包含多个数据实体的话，比如说多个学生成绩，那么需要使用数组的表现形式，就是[]。对于单个数据实体，比如一个学生的成绩，那么使用一个{}来封装数据，对于数据实体中的每个字段以及对应的值，使用key:value的方式来表示，多个key-value对之间用逗号分隔；多个{}代表的数据实体之间，用逗号分隔。**

* 扩展一下：

JSON在企业级项目开发过程中，扮演的角色是无比重要的。最常用的地方，莫过于基于Ajax的前端和后端程序之间的通信。比如说，在前端页面中，可以不刷新页面，直接发送一个Ajax异步请求到后端，后端返回一个JSON格式的数据，然后前端使用JSON格式的数据，渲染页面中的对应地方的信息。

* 在我们的项目中，JSON是起到了什么作用呢？

我们在task表中的task_param字段，会存放不同类型的任务对应的参数。

比如说，用户访问session分析模块与页面单跳转化率统计模块的任务参数是不同的，但是，使用同一张task表来存储所有类型的任务。那么，你怎么来存储不同类型的任务的不同的参数呢？你的表的字段是事先要定好的呀。

所以，我们采取了，用一个task_param字段，来存储不同类型的任务的参数的方式。task_param字段中，实际上会存储一个任务所有的字段，使用JSON的格式封装所有任务参数，并存储在task_param字段中。就实现了非常灵活的方式。

如何来操作JSON格式的数据？比如说，要获取JSON中某个字段的值。

**我们这里使用的是阿里的fastjson工具包。**

**使用这个工具包，可以方便的将字符串类型的JSON数据，转换为一个JSONObject对象，然后通过其中的getX()方法，获取指定的字段的值。**

### session聚合统计之自定义聚合函数 SessionAggrStatAccumulator.java

* session聚合统计：

统计出来之前通过条件过滤的session，访问时长在0s-3s的session的数量，占总session数量的比例；4s-6s。。。。；

访问步长在1-3的session的数量，占总session数量的比例；4-6。。。；

Accumulator 1s_3s = sc.accumulator(0L);。。。。。。十几个Accumulator

可以对过滤以后的session，调用foreach也可以，遍历所有session；计算每个session的访问时长和访问步长；

访问时长：把session的最后一个action的时间，减去第一个action的时间

访问步长：session的action数量

计算出访问时长和访问步长以后，根据对应的区间，找到对应的Accumulator，1s_3s.add(1L)同时每遍历一个session，就可以给总session数量对应的Accumulator，加1最后用各个区间的session数量，除以总session数量，就可以计算出各个区间的占比了

* 这种传统的实现方式，有什么不好？？？

**最大的不好，就是Accumulator太多了，不便于维护**

首先第一，很有可能，在写后面的累加代码的时候，比如找到了一个4s-6s的区间的session，但是却代码里面不小心，累加到7s-9s里面去了；

第二，当后期，项目如果要出现一些逻辑上的变更，比如说，session数量的计算逻辑，要改变，就得更改所有Accumulator对应的代码；或者说，又要增加几个范围，那么又要增加多个Accumulator，并且修改对应的累加代码；维护成本，相当之高（甚至可能，修改一个小功能，或者增加一个小功能，耗费的时间，比做一个新项目还要多；甚至于，还修改出了bug，那就耗费更多的时间）

所以，我们这里的设计，不打算采用传统的方式，用十几个，甚至二十个Accumulator，因为维护成本太高这里的

**实现思路是，我们自己自定义一个Accumulator，实现较为复杂的计算逻辑，一个Accumulator维护了所有范围区间的数量的统计逻辑低耦合，如果说，session数量计算逻辑要改变，那么不用变更session遍历的相关的代码；只要维护一个Accumulator里面的代码即可；如果计算逻辑后期变更，或者加了几个范围，那么也很方便，不用多加好几个Accumulator，去修改大量的代码；只要维护一个Accumulator里面的代码即可；维护成本，大大降低**

自定义Accumulator，也是Spark Core中，属于比较高端的一个技术使用自定义Accumulator，大家就可以任意的实现自己的复杂分布式计算的逻辑如果说，你的task，分布式，进行复杂计算逻辑，那么是很难实现的（借助于redis，维护中间状态，借助于zookeeper去实现分布式锁）但是，**使用自定义Accumulator，可以更方便进行中间状态的维护，而且不用担心并发和锁的问题**

### session聚合统计之重构实现思路与重构session聚合

如果不进行重构，直接来实现，思路：

```
1、actionRDD，映射成<sessionid,Row>的格式
2、按sessionid聚合，计算出每个session的访问时长和访问步长，生成一个新的RDD
3、遍历新生成的RDD，将每个session的访问时长和访问步长，去更新自定义Accumulator中的对应的值
4、使用自定义Accumulator中的统计值，去计算各个区间的比例5、将最后计算出来的结果，写入MySQL对应的表中
```

* 普通实现思路的问题：

1、为什么还要用actionRDD，去映射？其实我们之前在session聚合的时候，映射已经做过了。多此一举
2、是不是一定要，为了session的聚合这个功能，单独去遍历一遍session？其实没有必要，已经有session数据
之前过滤session的时候，其实，就相当于，是在遍历session，那么这里就没有必要再过滤一遍了

* 重构实现思路：

1、不要去生成任何新的RDD（处理上亿的数据）
2、不要去单独遍历一遍session的数据（处理上千万的数据）
3、可以在进行session聚合的时候，就直接计算出来每个session的访问时长和访问步长
4、在进行过滤的时候，本来就要遍历所有的聚合session信息，此时，就可以在某个session通过筛选条件后
将其访问时长和访问步长，累加到自定义的Accumulator上面去
5、就是两种截然不同的思考方式，和实现方式，在面对上亿，上千万数据的时候，甚至可以节省时间长达
半个小时，或者数个小时

* 开发Spark大型复杂项目的一些经验准则：

1、尽量少生成RDD
2、尽量少对RDD进行算子操作，如果有可能，尽量在一个算子里面，实现多个需要做的功能
3、尽量少对RDD进行shuffle算子操作，比如groupByKey、reduceByKey、sortByKey（map、mapToPair）
shuffle操作，会导致大量的磁盘读写，严重降低性能
有shuffle的算子，和没有shuffle的算子，甚至性能，会达到几十分钟，甚至数个小时的差别
有shfufle的算子，很容易导致数据倾斜，一旦数据倾斜，简直就是性能杀手（完整的解决方案）
4、无论做什么功能，性能第一
在传统的J2EE或者.NET后者PHP，软件/系统/网站开发中，我认为是架构和可维护性，可扩展性的重要
程度，远远高于了性能，大量的分布式的架构，设计模式，代码的划分，类的划分（高并发网站除外）
在大数据项目中，比如MapReduce、Hive、Spark、Storm，我认为性能的重要程度，远远大于一些代码
的规范，和设计模式，代码的划分，类的划分；

大数据，大数据，最重要的，就是性能

主要就是因为大数据以及大数据项目的特点，决定了，大数据的程序和项目的速度，都比较慢
如果不优先考虑性能的话，会导致一个大数据处理程序运行时间长度数个小时，甚至数十个小时
此时，对于用户体验，简直就是一场灾难

所以，推荐大数据项目，在开发和代码的架构中，优先考虑性能；其次考虑功能代码的划分、解耦合

我们如果采用第一种实现方案，那么其实就是代码划分（解耦合、可维护）优先，设计优先
如果采用第二种方案，那么其实就是性能优先

### session随机抽取之实现思路分析

每一次执行用户访问session分析模块，要抽取出100个session

**session随机抽取：**按每天的每个小时的session数量，占当天session总数的比例，乘以每天要抽取的session数量，计算出每个小时要抽取的session数量；然后呢，在每天每小时的session中，随机抽取出之前计算出来的数量的session。

举例：10000个session，100个session；0点~1点之间，有2000个session，占总session的比例就是0.2；按照比例，0点~1点需要抽取出来的session数量是100 * 0.2 = 20个；在0点~点的2000个session中，随机抽取出来20个session。

我们之前有什么数据：session粒度的聚合数据（计算出来session的start_time）

session聚合数据进行映射，将每个session发生的yyyy-MM-dd_HH（start_time）作为key，value就是session_id对上述数据，使用countByKey算子，就可以获取到每天每小时的session数量

（按时间比例随机抽取算法）每天每小时有多少session，根据这个数量计算出每天每小时的session占比，以及按照占比，需要抽取多少session，可以计算出每个小时内，从0~session数量之间的范围中，获取指定抽取数量个随机数，作为随机抽取的索引

把之前转换后的session数据（以yyyy-MM-dd_HH作为key），执行groupByKey算子；然后可以遍历每天每小时的session，遍历时，遇到之前计算出来的要抽取的索引，即将session抽取出来；抽取出来的session，直接写入MySQL数据库













## 各区域最热门top3商品

* 用户的商品点击行为

* UDAF函数

* RDD转换DataFrame，注册临时表

* 开窗函数

* Spark SQL数据倾斜解决

---

## 广告流量实时统计

* 用户的广告点击行为 



## 技术点和知识点

* 大数据项目的架构（公共组件的封装，包的划分，代码的规范）

* 复杂的分析需求（纯spark作业代码）

* Spark Core 算子的综合应用：map reduce count group

* 算定义Accumulator，按时间比例随机抽取算法，二次排序，分组取TopN算法

* 大数据项目开发流程：数据调研 需求分析 技术方案设计 数据库设计 编码实现 单元测试 本地测试

##  性能调优

### 常规调优

* 性能调优

executor, cpu per executor, memory per executor, driver memory

```bash
spark-submit \
--class com.soto.....  \
--num-executors 3 \  
--driver-memory 100m \
--executor-memory 1024m \
--executor-cores 3 \
/usr/local/......jar  \
```

* Kryo 序列化

```html
1. 算子函数中用到了外部变量，会序列化，会使用Kyro
2. 使用了序列化的持久化级别时，在将每个RDD partition序列化成一个在的字节数组时，就会使用Kryo进一步优化序列化的效率和性能
3. stage中task之间 执行shuffle时，文件通过网络传输，会使用序列化
```

###  JVM调优


### shuffle调优

### spark算子调优

* 数据倾斜解决
* troubleshotting

## 8. 生产环境测试
* Hive表测试

```bash
hive> create table user_visit_action( 
        date string,    
        user_id bigint, 
        session_id string,  
        page_id bigint, 
        action_time string, 
        search_keyword string,  
        click_category_id bigint,   
        click_product_id bigint,    
        order_category_ids string,  
        order_product_ids string,
        pay_category_ids string,    
        pay_product_ids string,
        city_id bigint  
        );

hive> load data local inpath '/home/sotowang/user/aur/ide/idea/idea-IU-182.3684.101/workspace/sparkhomework/user_visit_action.txt' overwrite into table user_visit_action;

hive> create table user_info(
    user_id bigint,
    username string,
    name string,
    age int,
    professional string,
    city string,
    sex string
    );

hive> load data local inpath '/home/sotowang/user/aur/ide/idea/idea-IU-182.3684.101/workspace/sparkhomework/user_info.txt' into table user_info;


hive> create table product_info(
      product_id bigint,
      product_name string,
      extend_info string
      ); 
      
hive> load data local inpath '/home/sotowang/user/aur/ide/idea/idea-IU-182.3684.101/workspace/sparkhomework/product_info.txt' into table product_info;


```

* 启动zookeeper

```bash
nohup zookeeper-server-start.sh $KAFKA_HOME/config/zookeeper.properties &
```

*启动kafka

```bash
nohup kafka-server-start.sh  -daemon $KAFKA_HOME/config/server.properties &
```

* 创建topic

```bash
kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic AdRealTimeLog

```

* 发送消息

```bash
kafka-console-producer.sh --broker-list localhost:9092 --topic AdRealTimeLog
```

*消费消息

```bash
kafka-console-consumer.sh --zookeeper localhost:2181 --topic AdRealTimeLog
```


