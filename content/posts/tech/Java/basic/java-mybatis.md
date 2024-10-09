---
title: "Java后端数据持久化层技术选型与初始化" #标题
date: 2023-04-15T22:12:25+08:00 #创建时间
lastmod: 2023-04-15T22:12:25+08:00 #更新时间
author: ["River"] #作者
keywords: #关键词
- mybatis
- java
categories: #类别
- 
tags: #标签
- mybatis
- java
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
cover:
    image: "" #图片路径：posts/tech/文章1/picture.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
    
reward: true # 打赏
---

# Java后端数据持久化层技术选型与初始化

## 最终选型

选型：MyBatis + MyBatis Generator + Mybatis-PageHelper

理由：

- 非侵入性，将数据持久化用sql实现，完全将数据持久化和service的逻辑解耦
- 灵活，可扩展性强，高速迭代
- 性能取决于sql，锻炼sql能力，倒逼自己熟悉sql
- 国内使用广泛，适用面广
- 出现bug好定位和fix
- generator也是官方维护
- 最方便使用的分页插件

## 选型比较

### Mybatis

- 半自动化orm，不完全遵守jpa，其实还是要控制sql去和数据库交互，只是把sql结果用java对象封装
- 优点：
  - 几乎完全将数据持久化和service的逻辑解耦，对sql的可操控性强
  - 性能较高
  - 灵活，简单高效，高速迭代。
- 缺点：
  - 但是要手动编写sql，单表重复性元素太高，如果数据库修改了要重新generate很麻烦。
  - 数据库移植性差，不能随意更换数据库

### Mybatis-plus

- 基于mybatis实现jpa，封装的太多，总之不用写sql了
- 性能差点
- 灵活性差，问题也不好定位
- 入侵性太强了直接到了service层
- 还一堆自定义注解

### Tk-mybatis、通用mapper

- 将generator生成的example只缩成了一个，简单的单表查询不用写了，仅仅是对dao层封装
- 我觉得主要还是解决了数据库表结构变化带来的修改成本
- 但是是个人开发者造的轮子，使用不广泛，有学习价值但是生产不敢用怕bug，侵入性也不小

### Spring data jpa

- 完全实现jpa，更好实现DDD驱动的设计思想
- 复杂很多
- 性能略低一点
- 不好调试

国外其实更注重开发效率和面向对象的分析和设计（DDD）

### Querydsl

- 平等的不喜欢每种DSL风格操作数据库的框架，维护起来很烦

## MyBatis基本原理

> 参考：
>
> - Mybatis 官方文档：https://mybatis.org/mybatis-3/zh/getting-started.html

### 使用逻辑

1. **从 XML / Java配置类 中构建 SqlSessionFactory**

   - 每个基于 MyBatis 的应用都是以一个 **SqlSessionFactory** 的实例为核心的。
   - SqlSessionFactory 的实例可以通过 **SqlSessionFactoryBuilder** 获得。
   - 而 SqlSessionFactoryBuilder 则可以从 **XML** 配置文件或一个预先配置的 **Configuration** 实例来构建出 **SqlSessionFactory** 实例。
   - XML 配置文件中包含了对 MyBatis 系统的核心设置，包括**获取数据库连接实例**的**数据源（DataSource）**以及**决定事务作用域和控制方式**的**事务管理器（TransactionManager）**。

2. **从 SqlSessionFactory 中获取 SqlSession**

   - **SqlSession 提供了在数据库执行 SQL 命令所需的所有方法**

     ```java
     try (SqlSession session = sqlSessionFactory.openSession()) {
       BlogMapper mapper = session.getMapper(BlogMapper.class);
       Blog blog = mapper.selectBlog(101);
     }
     ```

   - 一个语句既可以通过 XML 定义，也可以通过注解定义

     ```xml
     <?xml version="1.0" encoding="UTF-8" ?>
     <!DOCTYPE mapper
       PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
       "https://mybatis.org/dtd/mybatis-3-mapper.dtd">
     <mapper namespace="org.mybatis.example.BlogMapper">
       <select id="selectBlog" resultType="Blog">
         select * from Blog where id = #{id}
       </select>
     </mapper>
     ```

     ```java
     package org.mybatis.example;
     public interface BlogMapper {
       @Select("SELECT * FROM blog WHERE id = #{id}")
       Blog selectBlog(int id);
     }
     ```

### 不同类的对象的作用域和生命周期

**SqlSessionFactoryBuilder**

这个类可以被实例化、使用和**丢弃**，一旦创建了 SqlSessionFactory，就不再需要它了。 因此 SqlSessionFactoryBuilder 实例的最佳作用域是方法作用域（也就是局部方法变量）。 你可以重用 SqlSessionFactoryBuilder 来创建多个 SqlSessionFactory 实例，但最好还是不要一直保留着它，以保证所有的 XML 解析资源可以被释放给更重要的事情。

**SqlSessionFactory**

SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在，没有任何理由丢弃它或重新创建另一个实例。 使用 SqlSessionFactory 的最佳实践是在应用运行期间不要重复创建多次，多次重建 SqlSessionFactory 被视为一种代码“坏习惯”。因此 SqlSessionFactory 的最佳作用域是**应用作用域**。 有很多方法可以做到，最简单的就是使用**单例模式或者静态单例模式**。

**SqlSession**

**每个线程都应该有它自己的 SqlSession 实例**。SqlSession 的实例**不是线程安全的，因此是不能被共享的**，所以它的**最佳的作用域是请求或方法作用域**。 **绝对不能将 SqlSession 实例的引用放在一个类的静态域，甚至一个类的实例变量也不行**。 也绝不能将 SqlSession 实例的引用放在任何类型的托管作用域中，比如 Servlet 框架中的 HttpSession。 如果你现在正在使用一种 Web 框架，考虑将 SqlSession 放在一个和 HTTP 请求相似的作用域中。 换句话说，每次收到 HTTP 请求，就可以打开一个 SqlSession，返回一个响应后，就关闭它。 这个关闭操作很重要，为了确保每次都能执行关闭操作，你应该把这个关闭操作放到 finally 块中。 下面的示例就是一个确保 SqlSession 关闭的标准模式：

```
try (SqlSession session = sqlSessionFactory.openSession()) {
  // 你的应用逻辑代码
}
```

在所有代码中都遵循这种使用模式，可以保证所有数据库资源都能被正确地关闭。

**mapper映射器实例**

映射器是一些绑定映射语句的接口。映射器接口的实例是从 SqlSession 中获得的。虽然从技术层面上来讲，**任何映射器实例的最大作用域与请求它们的 SqlSession 相同。但方法作用域才是映射器实例的最合适的作用域。** 也就是说，**映射器实例应该在调用它们的方法中被获取，使用完毕之后即可丢弃**。 映射器实例并不需要被显式地关闭。尽管在整个请求作用域保留映射器实例不会有什么问题，但是你很快会发现，在这个作用域上管理太多像 SqlSession 的资源会让你忙不过来。 因此，最好将映射器放在方法作用域内。就像下面的例子一样：

```
try (SqlSession session = sqlSessionFactory.openSession()) {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  // 你的应用逻辑代码
}
```

## Mybatis + Spring 逻辑

> 参考：
>
> - Mybatis-spring 官方文档：https://mybatis.org/spring/zh/getting-started.html

MyBatis-Spring 会帮助你将 MyBatis 代码无缝地整合到 Spring 中。它将允许 MyBatis 参与到 Spring 的事务管理之中，创建映射器 mapper 和 `SqlSession` 并注入到 bean 中，以及将 Mybatis 的异常转换为 Spring 的 `DataAccessException`。 最终，可以做到应用代码不依赖于 MyBatis，Spring 或 MyBatis-Spring。

要和 Spring 一起使用 MyBatis，需要在 Spring 应用上下文中定义至少两样东西：**一个 `SqlSessionFactory` 和至少一个数据映射器类**。

### SqlSessionFactory 何去何从

**在 MyBatis-Spring 中，可使用 `SqlSessionFactoryBean`来创建 `SqlSessionFactory`。** 要配置这个工厂 bean，只需要把下面代码放在 Spring 的 XML 配置文件中：

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
</bean>
```

```java
@Configuration
public class MyBatisConfig {
  @Bean
  public SqlSessionFactory sqlSessionFactory() throws Exception {
    SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
    factoryBean.setDataSource(dataSource());
    return factoryBean.getObject();
  }
}
```

需要注意的是 `SqlSessionFactoryBean` 实现了 Spring 的 `FactoryBean` 接口（参见 Spring 官方文档 3.8 节 [通过工厂 bean 自定义实例化逻辑](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-factory-extension-factorybean) ）。 这意味着由 Spring 最终创建的 bean **并不是** `SqlSessionFactoryBean` 本身，而是工厂类（`SqlSessionFactoryBean`）的 getObject() 方法的返回结果。这种情况下，Spring 将会在应用启动时为你创建 `SqlSessionFactory`，并使用 `sqlSessionFactory` 这个名字存储起来。

通常，在 MyBatis-Spring 中，你不需要直接使用 `SqlSessionFactoryBean` 或对应的 `SqlSessionFactory`。 相反，session 的工厂 bean 将会被注入到 `MapperFactoryBean` 或其它继承于 `SqlSessionDaoSupport` 的 DAO（Data Access Object，数据访问对象）中。（详细看下面一节）

**DataSource**

`SqlSessionFactory` 有一个**唯一的必要属性：用于 JDBC 的 `DataSource`。**这可以是任意的 `DataSource` 对象，它的配置方法和其它 Spring 数据库连接是一样的。

**configLocation** / **configuration**

一个常用的属性是 `configLocation`，它用来指定 MyBatis 的 XML 配置文件路径。它在需要修改 MyBatis 的基础配置非常有用。通常，基础配置指的是 `<settings>` 或 `<typeAliases>` 元素。

需要注意的是，这个配置文件**并不需要**是一个完整的 MyBatis 配置。确切地说，任何环境配置（`<environments>`），数据源（`<DataSource>`）和 MyBatis 的事务管理器（`<transactionManager>`）都会被**忽略**。 `SqlSessionFactoryBean` 会创建它自有的 MyBatis 环境配置（`Environment`），并按要求设置自定义环境的值。

新增的 `configuration` 属性能够在没有对应的 MyBatis XML 配置文件的情况下，直接设置 `Configuration` 实例。例如：

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
  <property name="configuration">
    <bean class="org.apache.ibatis.session.Configuration">
      <property name="mapUnderscoreToCamelCase" value="true"/>
    </bean>
  </property>
</bean>
```

**mapperLocations**

如果 MyBatis 在映射器类对应的路径下找不到与之相对应的映射器 XML 文件，那么也需要配置文件。这时有两种解决办法：第一种是手动在 MyBatis 的 XML 配置文件中的 `<mappers>` 部分中指定 XML 文件的类路径；第二种是**设置工厂 bean 的 `mapperLocations` 属性**。

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
  <property name="mapperLocations" value="classpath*:sample/config/mappers/**/*.xml" />
</bean>
```

**transactionFactoryClass**

在容器管理事务的时候，你可能需要的一个属性是 `transactionFactoryClass`。详见Mybatis-spring的事务管理



### SqlSesstion、Mapper 何去何从

**通过 `MapperFactoryBean` ，来创建MapperFactoryBean，将对应的MapperFactoryBean加入到 Spring 中**

使用 MyBatis-Spring 之后，你不再需要直接使用 `SqlSessionFactory` 了，因为你的 bean 可以被注入一个线程安全的 `SqlSession`，它能基于 Spring 的事务配置来自动提交、回滚、关闭 session。

```java
public interface UserMapper {
  @Select("SELECT * FROM users WHERE id = #{userId}")
  User getUser(@Param("userId") String userId);
}
```

```xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
  <property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

```java
@Configuration
public class MyBatisConfig {
  @Bean
  public UserMapper userMapper() throws Exception {
    SqlSessionTemplate sqlSessionTemplate = new SqlSessionTemplate(sqlSessionFactory());
    return sqlSessionTemplate.getMapper(UserMapper.class);
  }
}
```

需要注意的是：所指定的映射器类**必须**是一个接口，而不是具体的实现类。

配置好之后，你就可以像 Spring 中普通的 bean 注入方法那样，将映射器注入到你的业务或服务对象中。

**`MapperFactoryBean` 将会负责 `SqlSession` 的创建和关闭**。 

如果使用了 Spring 的事务功能，那么当事务完成时，session 将会被提交或回滚。最终任何异常都会被转换成 Spring 的 `DataAccessException` 异常。

不需要一个个地注册你的所有映射器。你可以让 MyBatis-Spring 对类路径进行扫描来发现它们。

- **使用 `@MapperScan` 注解**
- 使用 `<mybatis:scan/>` 元素

### 事务管理

一个使用 MyBatis-Spring 的其中一个主要原因是它允许 MyBatis 参与到 Spring 的事务管理中。而不是给 MyBatis 创建一个新的专用事务管理器，MyBatis-Spring 借助了 Spring 中的 `DataSourceTransactionManager` 来实现事务管理。

一旦配置好了 Spring 的事务管理器，你就可以在 Spring 中按你平时的方式来配置事务。并且支持 `@Transactional` 注解和 AOP 风格的配置。在事务处理期间，一个单独的 `SqlSession` 对象将会被创建和使用。当事务完成时，这个 session 会以合适的方式提交或回滚。

事务配置好了以后，MyBatis-Spring 将会透明地管理事务。这样在你的 DAO 类中就不需要额外的代码了。

详细的还得去学习Spring事务管理

**配置事务交由JDBC管理事务**

要开启 Spring 的事务处理功能，在 Spring 的配置文件中创建一个 `DataSourceTransactionManager` 对象：

```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <constructor-arg ref="dataSource" />
</bean>
```

```java
@Configuration
public class DataSourceConfig {
  @Bean
  public DataSourceTransactionManager transactionManager() {
    return new DataSourceTransactionManager(dataSource());
  }
}
```

传入的 `DataSource` 可以是任何能够与 Spring 兼容的 JDBC `DataSource`。包括连接池和通过 JNDI 查找获得的 `DataSource`。

注意：为事务管理器指定的 `DataSource` **必须**和用来创建 `SqlSessionFactoryBean` 的是同一个数据源，否则事务管理器就无法工作了。

**配置事务交由容器管理事务**

如果你正使用一个 JEE 容器而且想让 Spring 参与到容器管理事务（Container managed transactions，CMT）的过程中，那么 Spring 应该被设置为使用 `JtaTransactionManager` 或由容器指定的一个子类作为事务管理器。最简单的方式是使用 Spring 的事务命名空间或使用 `JtaTransactionManagerFactoryBean`：

```xml
<tx:jta-transaction-manager />
```

```java
@Configuration
public class DataSourceConfig {
  @Bean
  public JtaTransactionManager transactionManager() {
    return new JtaTransactionManagerFactoryBean().getObject();
  }
}
```

在这个配置中，MyBatis 将会和其它由容器管理事务配置的 Spring 事务资源一样。Spring 会自动使用任何一个存在的容器事务管理器，并注入一个 `SqlSession`。 如果没有正在进行的事务，而基于事务配置需要一个新的事务的时候，Spring 会开启一个新的由容器管理的事务。

`DataSourceTransactionManager`：用于支持本地事务，事实上，其内部也是通过操作java.sql.Connection来开启、提交和回滚事务。

`JtaTransactionManager`：用于支持分布式事务，其实现了JTA规范，使用XA协议进行两阶段提交。需要注意的是，这只是一个代理，我们需要为其提供一个JTA provider，一般是Java EE容器提供的事务协调器(Java EE server's transaction coordinator)，也可以不依赖容器，配置一个本地的JTA provider。

方式一：给SqlSessionFactory注入Spring事务管理器ManagedTransactionFactory：

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
  <property name="transactionFactory">
    <bean class="org.apache.ibatis.transaction.managed.ManagedTransactionFactory" />
  </property>  
</bean>
```

或：

```java
@Bean
public SqlSessionFactory sqlSessionFactory() {
  SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
  factoryBean.setDataSource(dataSource());
  factoryBean.setTransactionFactory(new ManagedTransactionFactory());
  return factoryBean.getObject();
}
```

方式二：结合AOP实现事务的织入（通常使用）

```xml
<!--Spring配置声明式事务-->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <constructor-arg ref="dataSource" />
</bean>

<!--基于AOP实现事务织入-->
<!--配置事务通知-->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <!--给哪些方法配置事务-->
    <!--配置事务的传播特性： propagation-->
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED"/>
    </tx:attributes>
</tx:advice>

<!--配置事务切入点-->
<aop:config>
    <aop:pointcut id="txPointCut" expression="execution(* com.spong.mapper.*.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="txPointCut"/>
</aop:config>
```



### 实践方式

要调用 MyBatis 的数据方法，只需**一行代码**：

```java
public class FooServiceImpl implements FooService {

  private final UserMapper userMapper;

  public FooServiceImpl(UserMapper userMapper) {
    this.userMapper = userMapper;
  }

  public User doSomeBusinessStuff(String userId) {
    return this.userMapper.getUser(userId);
  }
}
```

## MyBatis + SpringBoot 配置

> 参考：
>
> - MyBatis 官方文档：https://mybatis.org/mybatis-3/zh/index.html
> - MyBatis with springboot 官方文档：http://mybatis.org/spring-boot-starter/mybatis-spring-boot-autoconfigure/zh/index.html
> - 比较好的文章：https://tobebetterjavaer.com/springboot/mybatis.html

### SpringBoot做了什么

正如你已经知道的， 要与 Spring 一起使用 MyBatis，你至少需要一个 `SqlSessionFactory` 和一个 mapper 接口。

MyBatis-Spring-Boot-Starter 将会：

- 自动探测存在的 `DataSource`
- 将使用 `SqlSessionFactoryBean` 创建并注册一个 `SqlSessionFactory` 的实例，并将探测到的 `DataSource` 作为数据源
- 将创建并注册一个从 `SqlSessionFactory` 中得到的 `SqlSessionTemplate` 的实例
- 自动扫描你的 mapper，将它们与 `SqlSessionTemplate` 相关联，并将它们注册到Spring 的环境（context）中去，这样它们就可以被注入到你的 bean 中

### 配置步骤

1. **使用idea的 springboot initializr 初始化项目，至少包含mybatis framework和mysql driver两个项目**

2. **添加其他依赖**：比如引入druid数据源，mybatis-pagehelper分页工具

3. **配置数据源**

   ```yml
   spring:
     datasource:
       druid:
         driver-class-name: com.mysql.cj.jdbc.Driver
         username: root
         password: xxxxxxxx
         url: jdbc:mysql://124.222.233.153:3306/power_grid?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai&useSSL=false
   
   ```

4. **初始化数据库的数据（比如运行初始化sql脚本）**

5. **新建对应的实体类**（可以添加lombok的@Data和@Builder）

6. **新建对应的mapper接口和mapper xml文档**

7. **（上述两步可以用generator自动生成）**

8. **在启动类 CodingmoreMybatisApplication 上添加 [@MapperScan](https://tobebetterjavaer.com/MapperScan) 注解来扫描 mapper**

9. **在application.yml中配置mapper xml文件的位置等相关mybatis配置**

   ```yml
   mybatis:
     mapper-locations: classpath:mapper/*.xml
   ```

10. **如果mapper xml文件在resource里面，还要在maven的pom里配置build选项，保证resource里的xml可以被打包构建( Maven 打包的时候默认会忽略 xml 文件)**

    ```xml
    <build>
        <resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                </includes>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
            </resource>
        </resources>
    </build>
    
    ```

### 配置参数选项

像其他的 Spring Boot 应用一样，配置参数在 `application.properties` （或 `application.yml` )。

MyBatis 在它的配置项中，使用 `mybatis` 作为前缀。

可用的配置项如下：

| 配置项（properties）                     | 描述                                                         |
| :--------------------------------------- | :----------------------------------------------------------- |
| `config-location`                        | MyBatis XML 配置文件的路径。                                 |
| `check-config-location`                  | 指定是否对 MyBatis XML 配置文件的存在进行检查。              |
| `mapper-locations`                       | XML 映射文件的路径。                                         |
| `type-aliases-package`                   | 搜索类型别名的包名。（包使用的分隔符是 “`,; \t\n`”）         |
| `type-aliases-super-type`                | 用于过滤类型别名的父类。如果没有指定，MyBatis会将所有从 `type-aliases-package` 搜索到的类作为类型别名处理。 |
| `type-handlers-package`                  | 搜索类型处理器的包名。（包使用的分隔符是 “`,; \t\n`”）       |
| `executor-type`                          | SQL 执行器类型： `SIMPLE`, `REUSE`, `BATCH`                  |
| `default-scripting-language-driver`      | 默认的脚本语言驱动（模板引擎），此功能需要与 mybatis-spring 2.0.2 以上版本一起使用。 |
| `configuration-properties`               | 可在外部配置的 MyBatis 配置项。指定的配置项可以被用作 MyBatis 配置文件和 Mapper 文件的占位符。更多细节 见 [MyBatis 参考页面](https://mybatis.org/mybatis-3/zh/configuration.html#properties)。 |
| `lazy-initialization`                    | 是否启用 mapper bean 的延迟初始化。设置 `true` 以启用延迟初始化。此功能需要与 mybatis-spring 2.0.2 以上版本一起使用。 |
| `mapper-default-scope`                   | 通过自动配置扫描的 mapper 组件的默认作用域。该功能需要与 mybatis-spring 2.0.6 以上版本一起使用。 |
| `inject-sql-session-on-mapper-scan`      | 设置是否注入 `SqlSessionTemplate` 或 `SqlSessionFactory` 组件 （如果你想回到 2.2.1 或之前的行为，请指定 `false` ）。如果你和 spring-native 一起使用，应该设置为 `true` （默认）。 |
| `configuration.*`                        | MyBatis Core 提供的`Configuration` 组件的配置项。有关可用的内部配置项，请参阅[MyBatis 参考页面](http://www.mybatis.org/mybatis-3/zh/configuration.html#settings)。注：此属性不能与 `config-location` 同时使用。 |
| `scripting-language-driver.thymeleaf.*`  | MyBatis `ThymeleafLanguageDriverConfig` 组件的 properties keys。有关可用的内部配置项，请参阅 [MyBatis Thymeleaf 参考页面](http://www.mybatis.org/thymeleaf-scripting/user-guide.html#_configuration_properties)。 |
| `scripting-language-driver.freemarker.*` | MyBatis `FreemarkerLanguageDriverConfig` 组件的 properties keys。有关可用的内部配置项，请参阅 [MyBatis FreeMarker 参考页面](http://www.mybatis.org/freemarker-scripting/#Configuration)。这个特性需要与 mybatis-freemarker 1.2.0 以上版本一起使用。 |
| `scripting-language-driver.velocity.*`   | MyBatis `VelocityLanguageDriverConfig` 组件的 properties keys。有关可用的内部属性，请参阅 [MyBatis Velocity 参考页面](http://www.mybatis.org/velocity-scripting/#Configuration)。这个特性需要与 mybatis-velocity 2.1.0 以上版本一起使用。 |

### 使用 Java Config 来自定义 MyBatis 配置

MyBatis-Spring-Boot-Starter 将自动寻找实现了 `ConfigurationCustomizer` 接口的组件，调用自定义 MyBatis 配置的方法。( 1.2.1 及以上的版本可用）

例如：

```java
@Configuration
public class MyBatisConfig {
  @Bean
  ConfigurationCustomizer mybatisConfigurationCustomizer() {
    return new ConfigurationCustomizer() {
      @Override
      public void customize(Configuration configuration) {
        // customize ...
      }
    };
  }
}
```

## MyBatis Generator with Maven 配置

> 参考：
>
> - 官方文档：http://mybatis.org/generator/running/runningWithMaven.html
> - 优质文章清晰步骤：https://zhuanlan.zhihu.com/p/495673409
> - 优质文章详细配置：https://zhuanlan.zhihu.com/p/373736998

1. **加入配置文件**

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE generatorConfiguration
           PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
           "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
   <!-- 配置生成器 -->
   <generatorConfiguration>
       <!-- 可以用于加载配置项或者配置文件，在整个配置文件中就可以使用${propertyKey}的方式来引用配置项
           resource：配置资源加载地址，使用resource，MBG从classpath开始找，比如com/myproject/generatorConfig.properties
           url：配置资源加载地质，使用URL的方式，比如file:///C:/myfolder/generatorConfig.properties.
           注意，两个属性只能选址一个;
           另外，如果使用了mybatis-generator-maven-plugin，那么在pom.xml中定义的properties都可以直接在generatorConfig.xml中使用
       <properties resource="" url="" />
        -->
   
       <!-- 在MBG工作的时候，需要额外加载的依赖包
           location属性指明加载jar/zip包的全路径
      <classPathEntry location="/Program Files/IBM/SQLLIB/java/db2java.zip" />
        -->
   
       <!--
           context:生成一组对象的环境
           id:必选，上下文id，用于在生成错误时提示
           defaultModelType:指定生成对象的样式
               1，conditional：默认，类似hierarchical，但是只有一个主键的时候会合并所有属性生成在同一个类；
               2，flat：所有内容（主键，blob）等全部生成在一个对象中；
               3，hierarchical：主键生成一个XXKey对象(key class)，Blob等单独生成一个对象，其他简单属性在一个对象中(record class)
           targetRuntime:
               1，MyBatis3：提供基本的基于动态SQL的CRUD方法和XXXByExample方法，会生成XML映射文件
               2，MyBatis3Simple：提供基本的基于动态SQL的CRUD方法，会生成XML映射文件
               3，MyBatis3DynamicSql	默认值，兼容JDK8+和MyBatis 3.4.2+，不会生成XML映射文件，忽略<sqlMapGenerator>的配置项，也就是Mapper全部注解化，依赖于MyBatis Dynamic SQL类库
               4，MyBatis3Kotlin	行为类似于MyBatis3DynamicSql，不过兼容Kotlin的代码生成
           introspectedColumnImpl：类全限定名，用于扩展MBG
       -->
       <context id="mysql" defaultModelType="conditional" targetRuntime="MyBatis3" >
   
           <!-- 是否使用分隔符号括住数据库关键字，默认false -->
           <property name="autoDelimitKeywords" value="false"/>
           <!-- 生成的Java文件的编码 -->
           <property name="javaFileEncoding" value="UTF-8"/>
           <!-- 格式化java代码 -->
           <property name="javaFormatter" value="org.mybatis.generator.api.dom.DefaultJavaFormatter"/>
           <!-- 格式化XML代码 -->
           <property name="xmlFormatter" value="org.mybatis.generator.api.dom.DefaultXmlFormatter"/>
   
           <!-- beginningDelimiter和endingDelimiter：指明数据库的用于标记数据库对象名的符号，比如ORACLE就是双引号，MYSQL默认是`反引号； -->
           <property name="beginningDelimiter" value="`"/>
           <property name="endingDelimiter" value="`"/>
   
           <!--        内置的插件实现见Supplied Plugins(http://mybatis.org/generator/reference/plugins.html)-->
           <!--        例如：引入org.mybatis.generator.plugins.SerializablePlugin插件-->
           <!--        会让生成的实体类自动实现java.io.Serializable接口并且添加serialVersionUID属性-->
           <!--        <plugin type=""></plugin>-->
   
           <!--        This plugin adds fluent builder methods to the generated model classes.-->
           <plugin type="org.mybatis.generator.plugins.FluentBuilderMethodsPlugin"></plugin>
   
           <!--        This plugin adds the @Mapper annotation to generated mapper interfaces.-->
           <plugin type="org.mybatis.generator.plugins.MapperAnnotationPlugin"></plugin>
         
         <!--生成mapper.xml时覆盖原文件-->
           <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin" />
   
           <commentGenerator>
               <property name="suppressAllComments" value="true"/>
               <property name="addRemarkComments" value="true"/>
           </commentGenerator>
   
           <!-- 必须要有的，使用这个配置链接数据库-->
           <jdbcConnection
                   driverClass="com.mysql.cj.jdbc.Driver"
                   connectionURL="jdbc:mysql://124.222.233.153:3306/power_grid?useUnicode=true&amp;characterEncoding=utf-8&amp;serverTimezone=Asia/Shanghai&amp;useSSL=false"
                   userId="root"
                   password="Mzd18931226">
               <!-- 这里面可以设置property属性，每一个property属性都设置到配置的Driver上 -->
             <!--            识别主键-->
               <property name="useInformationSchema" value="true"/>
   <!--            mysql 没有 catelog 一直要设置 不然会有问题-->
               <property name="nullCatalogMeansCurrent" value="true"/>
           </jdbcConnection>
   
           <!-- java类型处理器
               用于处理DB中的类型到Java中的类型映射，
               该标签只包含一个type属性，用于指定org.mybatis.generator.api.JavaTypeResolver接口的实现类
               默认使用JavaTypeResolverDefaultImpl；
               注意一点，默认会先尝试使用Integer，Long，Short等来对应DECIMAL和 NUMERIC数据类型；
               <javaTypeResolver>标签支持0或N个<property>标签，<property>的可选属性有
               1，forceBigDecimals	是否强制把所有的数字类型强制使用java.math.BigDecimal类型表示	默认false
               2，useJSR310Types	是否支持JSR310，主要是JSR310的新日期类型	                    默认false
           -->
           <javaTypeResolver type="org.mybatis.generator.internal.types.JavaTypeResolverDefaultImpl">
               <!--
                   true：使用BigDecimal对应DECIMAL和 NUMERIC数据类型
                   false：默认,
                       scale>0;length>18：使用BigDecimal;
                       scale=0;length[10,18]：使用Long；
                       scale=0;length[5,9]：使用Integer；
                       scale=0;length<5：使用Short；
                -->
               <property name="forceBigDecimals" value="false"/>
               <property name="useJSR310Types" value="true"/>
           </javaTypeResolver>
   
   
           <!-- java模型创建器，是必须要的元素
               负责：1，key类（见context的defaultModelType）；2，java类；3，查询类
               targetPackage：生成的类要放的包，真实的包受enableSubPackages属性控制；
               targetProject：目标项目，指定一个存在的目录下，生成的内容会放到指定目录中，如果目录不存在，MBG不会自动建目录
            -->
           <javaModelGenerator targetPackage="cn.river.entities" targetProject="src/main/java">
               <!--  for MyBatis3/MyBatis3Simple
                   自动为每一个生成的类创建一个构造方法，构造方法包含了所有的field；而不是使用setter；
                -->
               <property name="constructorBased" value="false"/>
   
               <!-- 在targetPackage的基础上，根据数据库的schema再生成一层package，最终生成的类放在这个package下，默认为false -->
               <property name="enableSubPackages" value="false"/>
   
               <!-- for MyBatis3 / MyBatis3Simple
                   是否创建一个不可变的类，如果为true，则不会生成Setter方法，所有字段都使用final修饰，提供一个带有所有字段属性的构造函数
                -->
               <property name="immutable" value="false"/>
   
   <!--            exampleTargetPackage	生成的伴随实体类的Example类的包名	-	- -->
   <!--            exampleTargetProject	生成的伴随实体类的Example类文件相对于项目（根目录）的位置-->
   <!--            <property name="exampleTargetPackage" value=""/>-->
   <!--            <property name="exampleTargetProject" value=""/>-->
   
               <!-- 设置一个根对象，
                   如果设置了这个根对象，那么生成的keyClass或者recordClass会继承这个类；在Table的rootClass属性中可以覆盖该选项
                   注意：如果在key class或者record class中有root class相同的属性，MBG就不会重新生成这些属性了，包括：
                       1，属性名相同，类型相同，有相同的getter/setter方法；
                -->
   <!--            <property name="rootClass" value="com._520it.mybatis.domain.BaseDomain"/>-->
   
               <!-- 设置是否在setter/getter方法中，对String类型字段调用trim()方法 -->
               <property name="trimStrings" value="true"/>
           </javaModelGenerator>
   
   
           <!-- 生成SQL map的XML文件生成器，
               注意，在Mybatis3之后，我们可以使用mapper.xml文件+Mapper接口（或者不用mapper接口），
                   或者只使用Mapper接口+Annotation，所以，如果 javaClientGenerator配置中配置了需要生成XML的话，这个元素就必须配置
               targetPackage/targetProject:同javaModelGenerator
            -->
           <sqlMapGenerator targetPackage="cn.river.newmapper" targetProject="src/main/resources">
               <!-- 在targetPackage的基础上，根据数据库的schema再生成一层package，最终生成的类放在这个package下，默认为false -->
               <property name="enableSubPackages" value="false"/>
           </sqlMapGenerator>
   
   
           <!-- 对于mybatis来说，即生成Mapper接口，注意，如果没有配置该元素，那么默认不会生成Mapper接口
               targetPackage/targetProject:同javaModelGenerator
               type：选择怎么生成mapper接口（在MyBatis3/MyBatis3Simple下）：
                   1，ANNOTATEDMAPPER：会生成使用Mapper接口+Annotation的方式创建（SQL生成在annotation中），不会生成对应的XML；
                   2，MIXEDMAPPER：Mapper接口生成的时候复杂的方法实现生成在XML映射文件中，而简单的实现通过注解和SqlProviders实现
                   3，XMLMAPPER：会生成Mapper接口，接口完全依赖XML；
               注意，如果context是MyBatis3Simple：只支持ANNOTATEDMAPPER和XMLMAPPER
           -->
           <javaClientGenerator targetPackage="cn.river.newmapper" type="XMLMAPPER" targetProject="src/main/java">
               <!-- 在targetPackage的基础上，根据数据库的schema再生成一层package，最终生成的类放在这个package下，默认为false -->
               <property name="enableSubPackages" value="false"/>
   
               <!-- 可以为所有生成的接口添加一个父接口，但是MBG只负责生成，不负责检查
               <property name="rootInterface" value=""/>
                -->
           </javaClientGenerator>
   
           <!-- 选择table来生成相关实体类、mapper和mapper xml，
           主要用于配置要生成代码的数据库表格，定制一些代码生成行为等等
           可以有一个或多个table，必须要有table元素
               可选：
               1，schema：数据库的schema；
                   mapperName  表对应的Mapper接口类名称
               2，catalog：数据库的catalog；
               3，alias：为数据表设置的别名，如果设置了alias，那么生成的所有的SELECT SQL语句中，列名会变成：alias_actualColumnName
               4，domainObjectName：生成的domain类的名字，如果不设置，直接使用表名作为domain类的名字；
               5，enableInsert（默认true）：指定是否生成insert语句；
               6，enableSelectByPrimaryKey（默认true）：指定是否生成按照主键查询对象的语句（就是getById或get）；
               7，enableSelectByExample（默认true）：MyBatis3Simple为false，指定是否生成动态查询语句；
               8，enableUpdateByPrimaryKey（默认true）：指定是否生成按照主键修改对象的语句（即update)；
               9，enableDeleteByPrimaryKey（默认true）：指定是否生成按照主键删除对象的语句（即delete）；
               10，enableDeleteByExample（默认true）：MyBatis3Simple为false，指定是否生成动态删除语句；
               11，enableCountByExample（默认true）：MyBatis3Simple为false，指定是否生成动态查询总条数语句（用于分页的总条数查询）；
               12，enableUpdateByExample（默认true）：MyBatis3Simple为false，指定是否生成动态修改语句（只修改对象中不为空的属性）；
               13，modelType：参考context元素的defaultModelType，相当于覆盖；
               14，delimitIdentifiers：参考tableName的解释，注意，默认的delimitIdentifiers是双引号，如果类似MYSQL这样的数据库，使用的是`（反引号，那么还需要设置context的beginningDelimiter和endingDelimiter属性）
               15，delimitAllColumns：设置是否所有生成的SQL中的列名都使用标识符引起来。默认为false，delimitIdentifiers参考context的属性
               注意，table里面很多参数都是对javaModelGenerator，context等元素的默认属性的一个复写；
            -->
           <table schema="power_grid" tableName="decloud_autodevice_account" >
   <!--            (自增主键）generatedKey identity: If true, then the column is flagged as an identity column and the generated <selectKey> element will be placed after the insert (for an identity column). If false, then the generated <selectKey> will be placed before the insert (typically for a sequence).-->
   <!--            Important: Even if you specify the type attribute as "post", you should still specify this value as "true" for identity columns. This will flag MBG to remove the column from the insert list.-->
   <!--            <generatedKey column="PSRID" sqlStatement="Mysql" identity=""></generatedKey>-->
           </table>
   <!--        <table tableName="userinfo" >-->
   
   <!--            &lt;!&ndash; 参考 javaModelGenerator 的 constructorBased属性&ndash;&gt;-->
   <!--            <property name="constructorBased" value="false"/>-->
   
   <!--            &lt;!&ndash; 默认为false，如果设置为true，在生成的SQL中，table名字不会加上catalog或schema； &ndash;&gt;-->
   <!--            <property name="ignoreQualifiersAtRuntime" value="false"/>-->
   
   <!--            &lt;!&ndash; 参考 javaModelGenerator 的 immutable 属性 &ndash;&gt;-->
   <!--            <property name="immutable" value="false"/>-->
   
   <!--            &lt;!&ndash; 指定是否只生成domain类，如果设置为true，只生成domain类，如果还配置了sqlMapGenerator，那么在mapper XML文件中，只生成resultMap元素 &ndash;&gt;-->
   <!--            <property name="modelOnly" value="false"/>-->
   
   <!--            &lt;!&ndash; 参考 javaModelGenerator 的 rootClass 属性-->
   <!--            <property name="rootClass" value=""/>-->
   <!--             &ndash;&gt;-->
   
   <!--            &lt;!&ndash; 参考javaClientGenerator 的  rootInterface 属性-->
   <!--            <property name="rootInterface" value=""/>-->
   <!--            &ndash;&gt;-->
   
   <!--            &lt;!&ndash; 如果设置了runtimeCatalog，那么在生成的SQL中，使用该指定的catalog，而不是table元素上的catalog-->
   <!--            <property name="runtimeCatalog" value=""/>-->
   <!--            &ndash;&gt;-->
   
   <!--            &lt;!&ndash; 如果设置了runtimeSchema，那么在生成的SQL中，使用该指定的schema，而不是table元素上的schema-->
   <!--            <property name="runtimeSchema" value=""/>-->
   <!--            &ndash;&gt;-->
   
   <!--            &lt;!&ndash; 如果设置了runtimeTableName，那么在生成的SQL中，使用该指定的tablename，而不是table元素上的tablename-->
   <!--            <property name="runtimeTableName" value=""/>-->
   <!--            &ndash;&gt;-->
   
   <!--            &lt;!&ndash; 注意，该属性只针对MyBatis3Simple有用；-->
   <!--                如果选择的runtime是MyBatis3Simple，那么会生成一个SelectAll方法，如果指定了selectAllOrderByClause，那么会在该SQL中添加指定的这个order条件；-->
   <!--             &ndash;&gt;-->
   <!--            <property name="selectAllOrderByClause" value="age desc,username asc"/>-->
   
   <!--            &lt;!&ndash; 如果设置为true，生成的model类会直接使用column本身的名字，而不会再使用驼峰命名方法，比如BORN_DATE，生成的属性名字就是BORN_DATE,而不会是bornDate &ndash;&gt;-->
   <!--            <property name="useActualColumnNames" value="false"/>-->
   
   
   <!--            &lt;!&ndash; generatedKey用于生成生成主键的方法，-->
   <!--                如果设置了该元素，MBG会在生成的<insert>元素中生成一条正确的<selectKey>元素，该元素可选-->
   <!--                column:主键的列名；-->
   <!--                sqlStatement：要生成的selectKey语句，有以下可选项：-->
   <!--                    Cloudscape:相当于selectKey的SQL为： VALUES IDENTITY_VAL_LOCAL()-->
   <!--                    DB2       :相当于selectKey的SQL为： VALUES IDENTITY_VAL_LOCAL()-->
   <!--                    DB2_MF    :相当于selectKey的SQL为：SELECT IDENTITY_VAL_LOCAL() FROM SYSIBM.SYSDUMMY1-->
   <!--                    Derby      :相当于selectKey的SQL为：VALUES IDENTITY_VAL_LOCAL()-->
   <!--                    HSQLDB      :相当于selectKey的SQL为：CALL IDENTITY()-->
   <!--                    Informix  :相当于selectKey的SQL为：select dbinfo('sqlca.sqlerrd1') from systables where tabid=1-->
   <!--                    MySql      :相当于selectKey的SQL为：SELECT LAST_INSERT_ID()-->
   <!--                    SqlServer :相当于selectKey的SQL为：SELECT SCOPE_IDENTITY()-->
   <!--                    SYBASE      :相当于selectKey的SQL为：SELECT @@IDENTITY-->
   <!--                    JDBC      :相当于在生成的insert元素上添加useGeneratedKeys="true"和keyProperty属性-->
   <!--            <generatedKey column="" sqlStatement=""/>-->
   <!--             &ndash;&gt;-->
   
   <!--            &lt;!&ndash;-->
   <!--                该元素会在根据表中列名计算对象属性名之前先重命名列名，非常适合用于表中的列都有公用的前缀字符串的时候，-->
   <!--                比如列名为：CUST_ID,CUST_NAME,CUST_EMAIL,CUST_ADDRESS等；-->
   <!--                那么就可以设置searchString为"^CUST_"，并使用空白替换，那么生成的Customer对象中的属性名称就不是-->
   <!--                custId,custName等，而是先被替换为ID,NAME,EMAIL,然后变成属性：id，name，email；-->
   <!--                注意，MBG是使用java.util.regex.Matcher.replaceAll来替换searchString和replaceString的，-->
   <!--                如果使用了columnOverride元素，该属性无效；-->
   <!--            <columnRenamingRule searchString="" replaceString=""/>-->
   <!--             &ndash;&gt;-->
   
   
   <!--            &lt;!&ndash; 用来修改表中某个列的属性，MBG会使用修改后的列来生成domain的属性；-->
   <!--                column:要重新设置的列名；-->
   <!--                注意，一个table元素中可以有多个columnOverride元素哈~-->
   <!--             &ndash;&gt;-->
   <!--            <columnOverride column="username">-->
   <!--                &lt;!&ndash; 使用property属性来指定列要生成的属性名称 &ndash;&gt;-->
   <!--                <property name="property" value="userName"/>-->
   
   <!--                &lt;!&ndash; javaType用于指定生成的domain的属性类型，使用类型的全限定名-->
   <!--                <property name="javaType" value=""/>-->
   <!--                 &ndash;&gt;-->
   
   <!--                &lt;!&ndash; jdbcType用于指定该列的JDBC类型-->
   <!--                <property name="jdbcType" value=""/>-->
   <!--                 &ndash;&gt;-->
   
   <!--                &lt;!&ndash; typeHandler 用于指定该列使用到的TypeHandler，如果要指定，配置类型处理器的全限定名-->
   <!--                    注意，mybatis中，不会生成到mybatis-config.xml中的typeHandler-->
   <!--                    只会生成类似：where id = #{id,jdbcType=BIGINT,typeHandler=com._520it.mybatis.MyTypeHandler}的参数描述-->
   <!--                <property name="jdbcType" value=""/>-->
   <!--                &ndash;&gt;-->
   
   <!--                &lt;!&ndash; 参考table元素的delimitAllColumns配置，默认为false-->
   <!--                <property name="delimitedColumnName" value=""/>-->
   <!--                 &ndash;&gt;-->
   <!--            </columnOverride>-->
   
   <!--            &lt;!&ndash; ignoreColumn设置一个MGB忽略的列，如果设置了改列，那么在生成的domain中，生成的SQL中，都不会有该列出现-->
   <!--                column:指定要忽略的列的名字；-->
   <!--                delimitedColumnName：参考table元素的delimitAllColumns配置，默认为false-->
   <!--                注意，一个table元素中可以有多个ignoreColumn元素-->
   <!--            <ignoreColumn column="deptId" delimitedColumnName=""/>-->
   <!--            &ndash;&gt;-->
   <!--        </table>-->
   
       </context>
   
   </generatorConfiguration>
   ```

   

2. **maven引入plugin，记得要配置好配置文件的位置**

   ```xml
               <plugin>
                   <groupId>org.mybatis.generator</groupId>
                   <artifactId>mybatis-generator-maven-plugin</artifactId>
                   <version>1.4.2</version>
                   <executions>
                       <execution>
                           <id>Generate MyBatis Artifacts</id>
                           <goals>
                               <goal>generate</goal>
                           </goals>
                           <phase>${generator.phase}</phase>
                       </execution>
                   </executions>
                   <configuration>
                       <overwrite>true</overwrite>
                       <configurationFile>src/main/resources/config/generatorConfig.xml</configurationFile>
                   </configuration>
                   <dependencies>
                       <dependency>
                           <groupId>mysql</groupId>
                           <artifactId>mysql-connector-java</artifactId>
                           <version>8.0.20</version>
                       </dependency>
                   </dependencies>
               </plugin>
   ```

3. **运行maven plugin**

**注意：配置详情还是得看官网，然后个性化配置**

## Mybatis-PageHelper + SpringBoot 配置

### 配置过程

1. 引入maven依赖

2. 配置拦截器插件参数

   >**分页插件可选参数如下：**
   >
   >1. `debug`: 调试参数，默认 `false` 关闭，设置为 `true` 启用后，可以排查系统中存在的**不安全调用**，[#查看如何安全调用](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md#3-pagehelper-安全调用)。 通过 `PageHelper.startPage` 等静态方法调用设置分页参数时，会记录当前执行的方法堆栈信息，当执行 MyBatis 的查询方法时，会使用设置好的分页参数， 此时会输出设置时的方法堆栈，通过查看堆栈，如果和当前执行的方法不一致，那么堆栈中对应的调用就是**不安全调用**，需要根据 [#安全调用](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md#3-pagehelper-安全调用) 中的方式调整。输出的堆栈示例如下：
   >
   >   ```
   >   00:19:08.915 [main] DEBUG c.github.pagehelper.PageInterceptor - java.lang.Exception: 设置分页参数时的堆栈信息
   >       at com.github.pagehelper.util.StackTraceUtil.current(StackTraceUtil.java:12)
   >       at com.github.pagehelper.Page.<init>(Page.java:111)
   >       at com.github.pagehelper.Page.<init>(Page.java:126)
   >       at com.github.pagehelper.page.PageMethod.startPage(PageMethod.java:139)
   >       at com.github.pagehelper.page.PageMethod.startPage(PageMethod.java:113)
   >       at com.github.pagehelper.page.PageMethod.startPage(PageMethod.java:102)
   >       at com.github.pagehelper.test.basic.PageHelperTest.testNamespaceWithStartPage(PageHelperTest.java:118)
   >       ...省略
   >   
   >   00:19:09.069 [main] DEBUG c.g.pagehelper.mapper.UserMapper - Cache Hit Ratio [com.github.pagehelper.mapper.UserMapper]: 0.0
   >   00:19:09.077 [main] DEBUG o.a.i.t.jdbc.JdbcTransaction - Opening JDBC Connection
   >   00:19:09.078 [main] DEBUG o.a.i.t.jdbc.JdbcTransaction - Setting autocommit to false on JDBC Connection [org.hsqldb.jdbc.JDBCConnection@6da21078]
   >   00:19:09.087 [main] DEBUG c.g.p.m.UserMapper.selectAll_COUNT - ==>  Preparing: SELECT count(1) FROM user
   >   00:19:09.121 [main] DEBUG c.g.p.m.UserMapper.selectAll_COUNT - ==> Parameters:
   >   00:19:09.131 [main] TRACE c.g.p.m.UserMapper.selectAll_COUNT - <==    Columns: C1
   >   00:19:09.131 [main] TRACE c.g.p.m.UserMapper.selectAll_COUNT - <==        Row: 183
   >   00:19:09.147 [main] DEBUG c.g.p.m.UserMapper.selectAll_COUNT - <==      Total: 1
   >   ```
   >
   >2. `dialect`：默认情况下会使用 PageHelper 方式进行分页，如果想要实现自己的分页逻辑，可以实现 `Dialect`(`com.github.pagehelper.Dialect`) 接口，然后配置该属性为实现类的全限定名称。
   >
   >3. `countSuffix`：根据查询创建或者查找对应的 count 查询时，追加的 msId 后缀，默认 `_COUNT`。
   >
   >4. `countMsIdGen`（5.3.2+）：count 方法的 msId 生成方式，默认是 查询的 msId + countSuffix，想要自己定义时，可以实现 `com.github.pagehelper.CountMsIdGen` 接口，将该参数配置为实现的全限定类名即可。 **一个常见的用途：** 在有Example查询的情况，`selectByExample` 可以使用对应的 `selectCountByExample` 方法进行 count 查询。
   >
   >5. `msCountCache`：自动创建查询的 count 查询方法时，创建的 count `MappedStatement` 会进行缓存，默认会优先查找 `com.google.common.cache.Cache` 的实现，如果项目没有 guava 依赖就会使用 mybatis 内置的 `CacheBuilder` 创建。想要对缓存进行细粒度的配置请参考源码: `com.github.pagehelper.cache.CacheFactory` ，两种默认方案提供了多个属性进行配置，也可以按照这里要求自己扩展实现。
   >
   >**下面几个参数都是针对默认 dialect 情况下的参数。使用自定义 dialect 实现时，下面的参数没有任何作用。**
   >
   >1. `helperDialect`：分页插件会自动检测当前的数据库链接，自动选择合适的分页方式。 你可以配置`helperDialect`属性来指定分页插件使用哪种方言。配置时，可以使用下面的缩写值： `oracle`,`mysql`,`mariadb`,`sqlite`,`hsqldb`,`postgresql`,`db2`,`sqlserver`,`informix`,`h2`,`sqlserver2012`,`derby` （完整内容看 [PageAutoDialect](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/src/main/java/com/github/pagehelper/page/PageAutoDialect.java)） **特别注意：**使用 SqlServer2012 数据库时，需要手动指定为 `sqlserver2012`，否则会使用 SqlServer2005 的方式进行分页，还可以设置 `useSqlserver2012=true` 将2012改为sqlserver的默认方式。 你也可以实现 `AbstractHelperDialect`，然后配置该属性为实现类的全限定名称即可使用自定义的实现方法。
   >
   >2. `dialectAlias`：允许配置自定义实现的 别名，可以用于根据 JDBCURL 自动获取对应实现，允许通过此种方式覆盖已有的实现，配置示例如（多个时分号隔开）：
   >
   >   ```
   >   <property name="dialectAlias" value="oracle=com.github.pagehelper.dialect.helper.OracleDialect"/>
   >   ```
   >
   >   当你使用的 jdbcurl 不在 [PageAutoDialect](https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/src/main/java/com/github/pagehelper/page/PageAutoDialect.java) 默认提供范围时，可以通过改参数实现自动识别。
   >
   >3. `useSqlserver2012`(sqlserver)：使用 SqlServer2012 数据库时，需要手动指定为 `sqlserver2012`，否则会使用 SqlServer2005 的方式进行分页，还可以设置 `useSqlserver2012=true`将2012改为sqlserver的默认方式。
   >
   >4. `defaultCount`：用于控制默认不带 count 查询的方法中，是否执行 count 查询，默认 `true` 会执行 count 查询，这是一个全局生效的参数，多数据源时也是统一的行为。
   >
   >5. `countColumn`：用于配置自动 count 查询时的查询列，默认值`0`，也就是 `count(0)`，`Page` 对象也新增了 `countColumn` 参数，可以针对具体查询进行配置。
   >
   >6. `offsetAsPageNum`：默认值为 `false`，该参数对使用 `RowBounds` 作为分页参数时有效。 当该参数设置为 `true` 时，会将 `RowBounds` 中的 `offset` 参数当成 `pageNum` 使用，可以用页码和页面大小两个参数进行分页。
   >
   >7. `rowBoundsWithCount`：默认值为`false`，该参数对使用 `RowBounds` 作为分页参数时有效。 当该参数设置为`true`时，使用 `RowBounds` 分页会进行 count 查询。
   >
   >8. `pageSizeZero`：默认值为 `false`，当该参数设置为 `true` 时，如果 `pageSize=0` 或者 `RowBounds.limit = 0` 就会查询出全部的结果（相当于没有执行分页查询，但是返回结果仍然是 `Page` 类型）。
   >
   >9. `reasonable`：分页合理化参数，默认值为`false`。当该参数设置为 `true` 时，`pageNum<=0` 时会查询第一页， `pageNum>pages`（超过总数时），会查询最后一页。默认`false` 时，直接根据参数进行查询。
   >
   >10. `params`：为了支持`startPage(Object params)`方法，增加了该参数来配置参数映射，用于从对象中根据属性名取值， 可以配置 `pageNum,pageSize,count,pageSizeZero,reasonable`，不配置映射的用默认值， 默认值为`pageNum=pageNum;pageSize=pageSize;count=countSql;reasonable=reasonable;pageSizeZero=pageSizeZero`。
   >
   >11. `supportMethodsArguments`：支持通过 Mapper 接口参数来传递分页参数，默认值`false`，分页插件会从查询方法的参数值中，自动根据上面 `params` 配置的字段中取值，查找到合适的值时就会自动分页。 使用方法可以参考测试代码中的 `com.github.pagehelper.test.basic` 包下的 `ArgumentsMapTest` 和 `ArgumentsObjTest`。
   >
   >12. `autoRuntimeDialect`：默认值为 `false`。设置为 `true` 时，允许在运行时根据多数据源自动识别对应方言的分页 （不支持自动选择`sqlserver2012`，只能使用`sqlserver`），用法和注意事项参考下面的**场景五**。
   >
   >13. `closeConn`：默认值为 `true`。当使用运行时动态数据源或没有设置 `helperDialect` 属性自动获取数据库类型时，会自动获取一个数据库连接， 通过该属性来设置是否关闭获取的这个连接，默认`true`关闭，设置为 `false` 后，不会关闭获取的连接，这个参数的设置要根据自己选择的数据源来决定。
   >
   >14. `aggregateFunctions`(5.1.5+)：默认为所有常见数据库的聚合函数，允许手动添加聚合函数（影响行数），所有以聚合函数开头的函数，在进行 count 转换时，会套一层。其他函数和列会被替换为 count(0) ，其中count列可以自己配置。
   >
   >15. `replaceSql`(sqlserver): 可选值为 `regex` 和 `simple`，默认值空时采用 `regex` 方式，也可以自己实现 `com.github.pagehelper.dialect.ReplaceSql` 接口。
   >
   >16. `sqlCacheClass`(sqlserver): 针对 sqlserver 生成的 count 和 page sql 进行缓存，缓存使用的 `com.github.pagehelper.cache.CacheFactory` ，可选的参数和前面的 `msCountCache` 一样。
   >
   >17. `autoDialectClass`：增加 `AutoDialect` 接口用于自动获取数据库类型，可以通过 `autoDialectClass` 配置为自己的实现类，默认使用 `DataSourceNegotiationAutoDialect`，优先根据连接池获取。 默认实现中，增加针对 `hikari,druid,tomcat-jdbc,c3p0,dbcp` 类型数据库连接池的特殊处理，直接从配置获取jdbcUrl，当使用其他类型数据源时，仍然使用旧的方式获取连接在读取jdbcUrl。 想要使用和旧版本完全相同方式时，可以配置 `autoDialectClass=old`。当数据库连接池类型非常明确时，建议配置为具体值，例如使用 hikari 时，配置 `autoDialectClass=hikari` ，使用其他连接池时，配置为自己的实现类。
   >
   >18. `boundSqlInterceptors`：增加分页插件的 `BoundSqlInterceptor` 拦截器，可以在3个阶段对 SQL 进行处理或者简单读取， 增加参数 `boundSqlInterceptors`，可以配置多个实现 `BoundSqlInterceptor` 接口的实现类名， 使用英文逗号隔开。PageHelper调用时，也可以通过类似 `PageHelper.startPage(x,x).boundSqlInterceptor(BoundSqlInterceptor boundSqlInterceptor)`针对本次分页进行设置。
   >
   >19. `keepOrderBy`：转换count查询时保留查询的 order by 排序。除全局配置外，可以针对单次操作进行设置。
   >
   >20. `keepSubSelectOrderBy`：转换count查询时保留子查询的 order by 排序。可以避免给所有子查询添加 `/*keep orderby*/`，除全局配置外，可以针对单次操作进行设置。
   >
   >21. `sqlParser`：配置 JSqlParser 解析器，注意是 `com.github.pagehelper.JSqlParser` 接口，用于支持 sqlserver 等需要额外配置的情况。
   >
   >**重要提示：**
   >
   >当 `offsetAsPageNum=false` 的时候，由于 `PageNum` 问题，`RowBounds`查询的时候 `reasonable` 会强制为 `false`。使用 `PageHelper.startPage` 方法不受影响。
   >
   >
   >
   >**如何选择配置这些参数**
   >
   >单独看每个参数的说明可能是一件让人不爽的事情，这里列举一些可能会用到某些参数的情况。
   >
   >##### 场景一
   >
   >如果你仍然在用类似ibatis式的命名空间调用方式，你也许会用到`rowBoundsWithCount`， 分页插件对`RowBounds`支持和 MyBatis 默认的方式是一致，默认情况下不会进行 count 查询，如果你想在分页查询时进行 count 查询， 以及使用更强大的 `PageInfo` 类，你需要设置该参数为 `true`。
   >
   >**注：** `PageRowBounds` 想要查询总数也需要配置该属性为 `true`。
   >
   >##### 场景二
   >
   >如果你仍然在用类似ibatis式的命名空间调用方式，你觉得 `RowBounds` 中的两个参数 `offset,limit` 不如 `pageNum,pageSize` 容易理解， 你可以使用 `offsetAsPageNum` 参数，将该参数设置为 `true` 后，`offset`会当成 `pageNum` 使用，`limit` 和 `pageSize` 含义相同。
   >
   >##### 场景三
   >
   >如果觉得某个地方使用分页后，你仍然想通过控制参数查询全部的结果，你可以配置 `pageSizeZero` 为 `true`， 配置后，当 `pageSize=0` 或者 `RowBounds.limit = 0` 就会查询出全部的结果。
   >
   >##### 场景四
   >
   >如果你分页插件使用于类似分页查看列表式的数据，如新闻列表，软件列表， 你希望用户输入的页数不在合法范围（第一页到最后一页之外）时能够正确的响应到正确的结果页面， 那么你可以配置 `reasonable` 为 `true`，这时如果 `pageNum<=0` 会查询第一页，如果 `pageNum>总页数` 会查询最后一页。
   >
   >##### 场景五
   >
   >如果你在 Spring 中配置了动态数据源，并且连接不同类型的数据库，这时你可以配置 `autoRuntimeDialect` 为 `true`，这样在使用不同数据源时，会使用匹配的分页进行查询。 这种情况下，你还需要特别注意 `closeConn` 参数，由于获取数据源类型会获取一个数据库连接，所以需要通过这个参数来控制获取连接后，是否关闭该连接。 默认为 `true`，有些数据库连接关闭后就没法进行后续的数据库操作。而有些数据库连接不关闭就会很快由于连接数用完而导致数据库无响应。所以在使用该功能时，特别需要注意你使用的数据源是否需要关闭数据库连接。
   >
   >当不使用动态数据源而只是自动获取 `helperDialect` 时，数据库连接只会获取一次，所以不需要担心占用的这一个连接是否会导致数据库出错，但是最好也根据数据源的特性选择是否关闭连接。

3. 在代码中使用：

   ```java
   //第一种，RowBounds方式的调用
   List<User> list = sqlSession.selectList("x.y.selectIf", null, new RowBounds(0, 10));
   
   //第二种，Mapper接口方式的调用，推荐这种使用方式。
   PageHelper.startPage(1, 10);
   List<User> list = userMapper.selectIf(1);
   
   //第三种，Mapper接口方式的调用，推荐这种使用方式。
   PageHelper.offsetPage(1, 10);
   List<User> list = userMapper.selectIf(1);
   
   //第四种，参数方法调用
   //存在以下 Mapper 接口方法，你不需要在 xml 处理后两个参数
   public interface CountryMapper {
       List<User> selectByPageNumSize(
               @Param("user") User user,
               @Param("pageNum") int pageNum,
               @Param("pageSize") int pageSize);
   }
   //配置supportMethodsArguments=true
   //在代码中直接调用：
   List<User> list = userMapper.selectByPageNumSize(user, 1, 10);
   
   //第五种，参数对象
   //如果 pageNum 和 pageSize 存在于 User 对象中，只要参数有值，也会被分页
   //有如下 User 对象
   public class User {
       //其他fields
       //下面两个参数名和 params 配置的名字一致
       private Integer pageNum;
       private Integer pageSize;
   }
   //存在以下 Mapper 接口方法，你不需要在 xml 处理后两个参数
   public interface CountryMapper {
       List<User> selectByPageNumSize(User user);
   }
   //当 user 中的 pageNum!= null && pageSize!= null 时，会自动分页
   List<User> list = userMapper.selectByPageNumSize(user);
   
   //第六种，ISelect 接口方式
   //jdk6,7用法，创建接口
   Page<User> page = PageHelper.startPage(1, 10).doSelectPage(new ISelect() {
       @Override
       public void doSelect() {
           userMapper.selectGroupBy();
       }
   });
   //jdk8 lambda用法
   Page<User> page = PageHelper.startPage(1, 10).doSelectPage(()-> userMapper.selectGroupBy());
   
   //也可以直接返回PageInfo，注意doSelectPageInfo方法和doSelectPage
   pageInfo = PageHelper.startPage(1, 10).doSelectPageInfo(new ISelect() {
       @Override
       public void doSelect() {
           userMapper.selectGroupBy();
       }
   });
   //对应的lambda用法
   pageInfo = PageHelper.startPage(1, 10).doSelectPageInfo(() -> userMapper.selectGroupBy());
   
   //count查询，返回一个查询语句的count数
   long total = PageHelper.count(new ISelect() {
       @Override
       public void doSelect() {
           userMapper.selectLike(user);
       }
   });
   //lambda
           total=PageHelper.count(()->userMapper.selectLike(user));
   ```

### 使用提示

**`PageHelper.startPage`方法重要提示**

只有紧跟在`PageHelper.startPage`方法后的**第一个**Mybatis的**查询（Select）**方法会被分页。

**请不要配置多个分页插件**

请不要在系统中配置多个分页插件(使用Spring时,`mybatis-config.xml`和`Spring<bean>`配置方式，请选择其中一种，不要同时配置多个分页插件)！

**分页插件不支持带有`for update`语句的分页**

对于带有`for update`的sql，会抛出运行时异常，对于这样的sql建议手动分页，毕竟这样的sql需要重视。

**分页插件不支持嵌套结果映射**

由于嵌套结果方式会导致结果集被折叠，因此分页查询的结果在折叠后总数会减少，所以无法保证分页结果数量正确。

## 最终可复用的架子

springboot 集成 mybatis、mybatis generator、mybatis-pagehelper

1. initialize，引入mybatis framework、mysql connect，手动引入druid和mybatis-pagehelper依赖，和mybatis generator插件
2. 配置，配置数据源、mybatis，再配置mybatis generator（重点文件），顺便配置mybatis-pagehelper
3. 初始化数据库后，运行mybatis generator，检查后，在pom里设置mybatis generator在build不再运行



小tip：

insert：  插入n条记录，返回影响行数n。（n>=1，n为0时实际为插入失败）

update：更新n条记录，返回影响行数n。（n>=0）

delete： 删除n条记录，返回影响行数n。（n>=0）
