---
title: Spring高级装配
tags:
  - Spring
  - 依赖注入
categories: 学习笔记
abbrlink: fe377a4
date: 2023-01-10 15:05:26
---
上一篇文章已经讲解了 spring 依赖注入装配的方法。但是为了处理一些特殊的问题，或者为了更好地实现某些功能，spring 还提供了更多的装配设置。

## 一、条件化声明 Bean
让我们回到上一篇第一节的例子，假设这次我们需要在开发时使用 SQLite，而在生产环境使用 Oracle 要怎么办呢？

或许可以这样，我们在生产环境注意将 `DBUtil` Bean 改成 Oracle，把原来的部分注释掉，就像是这样。
```java
@Bean
public DBUtil databaseUtil() {
    // return new SQLiteUtil(); // for dev
    return new OracleUtil(); // for prod
}
```
但这样做很明显是不合适的。如果许多组件都需要进行调整的话，修改上就会十分复杂，且很容易出错。

我们可以使用注释 `@Profile` 实现不同环境条件下选择不同的装配方式。这需要给带有 `@Bean` 或 `@Configuration` 注释的方法或类添加注释 `@Profile("某某环境")`。于是数据库的选择就可以改为如下形式：
```java
@Bean
@Profile("dev")
public DBUtil sqliteUtil() {
    return new SQLiteUtil();
}

@Bean
@Profile("prod")
public DBUtil oracleUtil() {
    return new OracleUtil();
}
```

想要启用某个 Bean 需要设置环境。具体来说，这通过两个环境变量来实现，`spring.profiles.default` 和 `spring.profiles.active`。这两个值可以在许多地方定义。一种方式是在 properties 或 yaml 文件中定义，如
```properties
#application.properties
spring.profiles.default=dev #默认环境为 dev
spring.profiles.active=prod #当前环境为 prod
```

为了指明所使用的配置文件，还需在配置类添加注释 `@PropertySource`。
```java
@Configuration
@PropertySource("classpath:application.properties")
public class DBUtilConfig
```

最终配置类代码为
```java
@Configuration
@PropertySource("classpath:application.properties")
public class DBUtilConfig {
    @Bean
    @Profile("dev")
    public DBUtil sqliteUtil() {
        return new SQLiteUtil();
    }

    @Bean
    @Profile("prod")
    public DBUtil oracleUtil() {
        return new OracleUtil();
    }
}
```

更一般的，还可以自定义使 Bean 或配置生效的条件，这需要用到 `@Conditional` 注释。

注释的使用形式如下
```java
@Configuration
@Conditional(SomethingIsTrueCondition.class)
public class ConditionalConfig
```
我们把一个类作为 `@Conditional` 注释的参数，这个类要继承 `Condition` 类，实现 `boolen matches(ConditionContext context, AnnotatedTypeMetadata metadata)` 方法。只有 `matches` 方法的返回值为真，才会使由 `@Conditional` 修饰的部分生效。

> 其实 `@Profile` 注释也是由 `@Conditional` 所实现的。查看源码，就会发现`@Profile` 也有一个 `@Conditional` 注释。
> ```java
> @Target({ElementType.TYPE, ElementType.METHOD})
> @Retention(RetentionPolicy.RUNTIME)
> @Documented
> @Conditional({ProfileCondition.class})
> public @interface Profile {
>     String[] value();
> }
> ```

## 二、消除自动装配的歧义
在自动装配时，如果在 `@ComponentScan` 的扫描范围内有多个可以满足装配条件的组件（`@Component`），就会产生装配的歧义。自动装配无法决定应该装配哪一个。

还举数据库作为例子，有 `OracleDBUtil` `MySQLDBUtil` `SQLiteDBUtil` 都可以作为自动装配的组件，那么就无法确定应该装配哪一个。
```java
@Component
public class OracleDBUtil extends DBUtil {}

@Component
public class MySQLDBUtil extends DBUtil {}

@Component
public class SQLiteDBUtil extends DBUtil {}

public class MyDAO {
    @Autowired
    private DBUtil databaseUtil;
}
```

这种情况可以选择一个组件添加 `@Primary` 注释，表示若出现歧义时，优先选择该组件作为装配对象。
```java
@Component
@Primary
public class MySQLDBUtil extends DBUtil {}
```

另外也可以在注释的地方增加 `@Qualifier` 并添加参数，这将在注入处指定依赖。`@Qualifier` 的参数是 Bean 的 id，默认是将对应的类名的首字母变为小写作为 id，当然也可以通过在 `@Bean` 注释添加参数指定一个 id。

```java
public class MyDAO {
    @Autowired
    @Qualifier("mySQLDBUtil")
    private DBUtil databaseUtil;
}
```

> 指定 bean id
> ```java
> @Bean("renameBean")
> public DBUtil mySQLDBUtil() {
>     return new MySQLDBUtil();
> }
> ```

### 三、Bean 的作用域
默认的情况下，经过 Spring 托管的对象都是以单例的形式存在的。但一些时候，只有单例无法实现想要的效果。我们还需要一些类来完成保存状态、存储信息等操作。在网络应用中，我们也需要为每个会话创建对应的实例。这都需要调整 Bean 的作用域，一部分 Bean 存在于全局，一部分 Bean 存在于每次创建，一部分 Bean 存在于每次请求，等等。

Spring 定义了多种作用域，包括：
- 单例（Singleton）：在整个应用中，只创建bean的一个实例。
- 原型（Prototype）：每次注入或者通过Spring应用上下文获取的时候，都会创建一个新的bean实例。
- 会话（Session）：在Web应用中，为每个会话创建一个bean实例。
- 请求（Rquest）：在Web应用中，为每个请求创建一个bean实例。

设定作用域的方法很简单，只要使用 `@Scope` 注释即可。这一注释可以用在 `@Component` 或 `@Bean` 所修饰的对象上。

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
// @Scope("prototype") // 另一种方式
public class UserInfo
```

### 四、运行时值注入
让我们重新审视一下 `SgtPeppers` 类。
```java
public class SgtPeppers implements CompactDisc {
    private String title = "Sgt. Pepper's Lonely Hearts Club Band";
    private String artist = "The Beatles";

    @Override
    public void play() {
        System.out.println("Playing " + title + " by " + artist);
    }
}
```
就能感觉到，`title` 和 `artist` 的值还是紧密的与 `SgtPeppers` 耦合在了一起，被硬编码到了类中。按照依赖注入的原则，我们利用构造函数将 title 和 artist 的值移到类的定义之外。当然，这样的话这个类就不能称为 `SgtPeppers` 了。
```java
public class BlankDisc implements CompactDisc {
    private String title;
    private String artist;
    
    public SgtPeppers(String title, String artist) {
        this.title = title;
        this.artist = artist;
    }

    @Override
    public void play() {
        System.out.println("Playing " + title + " by " + artist);
    }
}
```

那么自然地，字符串就转而出现在装配的过程中。
```java
@Bean
public CompactDisc sgtPeppers() {
    return new SgtPeppers(
        "Sgt. Pepper's Lonely Hearts Club Band",
        "The Beatles");
}
```

但是这还不够，因为要修改字符串值的话还需要对配置类进行修改。我们希望这些值被保存在配置中，只在运行时才注入到程序中。

方法有两种，**第一种**是通过 `Environment` 访问环境变量。与使用 `@Profile` 时类似，我们要引入配置位置，并通过自动注入获取环境变量类 `Environment`。调用其方法 `getProperty()` 获取对应名称的配置值。
```java
// CDPlayerConfig.java
@Configuration
@PropertySource("classpath:application.properties")
public class CDPlayerConfig {
    @Autowired
    Environment env;

    @Bean
    public CompactDisc blankDisc() {
        return new BlankDisc(
                env.getProperty("disc.title"),
                env.getProperty("disc.artist"));
    }

    @Bean
    public CDPlayer cdPlayer(CompactDisc cd) {
        return new CDPlayer(cd);
    }
}
```

设置的配置如下：
```properties
#application.properties
disc.title=Sgt. Pepper's Lonely Hearts Club Band
disc.artist=The Beatles
```

**第二种**方法是使用 `@Value` 注释。`@Value` 与 `@Autowired` 类似，只不过 `@Value` 注入的是值，而 `@Autowired` 注入的是对象。
```java
@Configuration
@PropertySource("classpath:application.properties")
public class CDPlayerConfig {
    @Bean
    public CompactDisc blankDisc(
            @Value("${disc.title}") String title,
            @Value("${disc.artist}") String artist) {
        return new BlankDisc(title, artist);
    }

    @Bean
    public CDPlayer cdPlayer(CompactDisc cd) {
        return new CDPlayer(cd);
    }
}
```
`@Value` 使用时需要参数，`"${disc.title}"` 是一个属性占位符，表示取 `disc.title` 的值，这个值将作为 `title` 的值。

> 另外 `@Value` 中还可以使用以 `#{...}` 表示的 SpEL 表达式。可以以更灵活且强大的方式取值，在这里不赘述。