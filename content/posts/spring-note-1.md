+++
title = "Spring 依赖注入"
tags = ["Spring"]
categories = ["开发"]
series = ["Spring 笔记"]
aliases = ["/posts/94547693"]
date = "2023-01-09T17:42:35+08:00"
+++
## 一、依赖注入
考虑这样一个例子，`CustomerDAO` 类使用了 `DBUtil` 类中的方法连接到 MySQL 数据库
```java
public class CustomerDAO
{
    private DBUtil databaseUtil = new DBUtil();

    public Customer findCustomerByName(String name)
    {
        databaseUtil.doSomething()
    }
}
```

假设现在想要改为使用 Oracle 数据库，需要怎么做？可以使用 `OracleDBUtil` 继承 `DBUtil`，并重写其相关方法。但是由于 `CustomerDAO` 中将 DBUtil 实例添加到其初始化过程中。因此想要修改的话还是需要对 `CustomerDAO` 本身进行修改。这违反了开-闭原则。
```java
public class CustomerDAO
{
    private DBUtil databaseUtil = new OracleDBUtil();

    public Customer findCustomerByName(String name)
    {
        databaseUtil.doSomething()
    }
}
```

为了使对 `dbUtil` 的修改不影响 `CustomerDAO` 本身，我们可以将实例的创建移到 `CustomerDAO` 之外。
```java
public class CustomerDAO
{
    private DBUtil databaseUtil;

    public CustomerDAO(DBUtil databaseUtil)
    {
        this.databaseUtil = databaseUtil;
    }

    public Customer findCustomerByName(String name)
    {
        databaseUtil.doSomething()
    }
}
```
这样，我们就可以自主选择 DBUtil 的具体实现，同时不会造成对 CustomerDAO 的修改。保证了开闭原则，降低了代码的耦合程度，有助于更有效的代码复用。
```java
DBUtil databaseUtil = new OracleDBUtil();
var customerDAO = new CustomerDAO(databaseUtil) 
```

这样的方法称为依赖注入。也就是，一个类对其他类的依赖，并不存在于这个类中，而是通过构造函数或其他方法传入这个类中。这样就使得编程时只需要关注抽象的依赖，而不需要考虑具体的实现。

## 二、spring的依赖注入
依赖注入是 spring 的核心之一。spring 提供了通过 java 注释或 xml 进行依赖注入配置的方法。这里只介绍通过 java 注释实现的手动装配和自动装配。

接下来本文会举一个经典例子来比较手动装配和自动装配的不同。考虑我们需要一个 CDPlayer，其中包含 CD。CD 具有 `play()` 方法，需要 CDPlayer 调用才能播放。

## 三、自动化装配
不管自动或手动装配，都需要创建配置类，并为其添加注释 `@Configuration`。这一配置类将提供可供 spring 维护的用于装配的类。这些类称为 Bean（爪哇咖啡豆！！）。
```java
@Configuration
public class CDPlayerConfig
```

自动化装配方式中，用于指示类与类的依赖关系的标志分散在各个类中。给类添加注释 `@Component` 表示该类为一个 Bean。或者可以认为这表示配置类中会自动生成一个创建该类型 Bean 的方法。
```java
@Component
public class SgtPeppers implements CompactDisc
```

同时在需要被注入的位置，需要添加 `@Autowired` 方法。表示该属性是自动装配的。
```java
@Autowired
private CompactDisc cd;
```

最后，在配置类中再添加一条注释 `@ComponentScan`，表示该配置类会扫描对应路径下所有带 `@Component` 注释的类，为这些类创建生成 Bean 的方法。
```java
@Configuration
@ComponentScan
public class CDPlayerConfig
```

> `@ComponentScan` 默认扫描与配置类在同一包或在子包下的所有 Component，可以通过 `@ComponentScan("包名")` 或 `@ComponentScan({"包名1", "包名2"})` 来设置扫描路径。

最终各文件代码如下（去除导入语句，各文件在同一包目录下）：
```java
// CompactDisc.java
public interface CompactDisc {
    void play();
}

// CDPlayerConfig.java
@Configuration
@ComponentScan
public class CDPlayerConfig {
}

// SgtPeppers.java
@Component
public class SgtPeppers implements CompactDisc {
    private String title = "Sgt. Pepper's Lonely Hearts Club Band";
    private String artist = "The Beatles";

    @Override
    public void play() {
        System.out.println("Playing " + title + " by " + artist);
    }
}

// CDPlayer.java
@Component
public class CDPlayer {
    @Autowired
    private CompactDisc cd;

    public void playCD() {
        cd.play();
    }
}
```

注意此时若直接通过 `new` 创建 `CDPlayer` 实例，`cd` 属性不会被装配。正确的使用方法是让 spring 托管对象的创建。这将在第五节介绍。

## 四、注释手动装配
手动装配 Bean 需要自行编写配置类。但是不需要在其他类中添加注释。

配置类只需要添加注释 `@Configuration`，而不需要 `@ComponentScan`。
```java
@Configuration
public class CDPlayerConfig
```

接着在配置类中编写方法，返回值为要创建 Bean 的类型。在例子中，也就是要 `SgtPeppers` 和 `CDPlayer`。同时要对方法添加 `@Bean` 注释。
```java
@Bean
public CompactDisc compactDisc() {
    return new SgtPeppers();
}

@Bean
public CDPlayer cdPlayer() {
    return new CDPlayer(sgtPeppers());
}
```
> 注意在 `cdPlayer()` 方法中使用了 `CDPlayer` 的一个参数的构造函数。为了通过java 代码实现配置，实现对希望注入依赖的属性的访问（如构造函数，或 `setter`）是必要的。
> ```java
> public CDPlayer(CompactDisc cd) {
>     this.cd = cd;
> }
> ```
通过 `new CDPlayer(sgtPeppers())`，我们就建立了 CDPlayer 和 SgtPeppers 的依赖关系。另外一种写法更加简洁，也更能体现组件之间的依赖关系。
```java
@Bean
public CompactDisc compactDisc() {
    return new SgtPeppers();
}

@Bean
public CDPlayer cdPlayer(CompactDisc cd) {
    return new CDPlayer(cd);
}
```

最终各文件代码如下：
```java
// CompactDisc.java
public interface CompactDisc {
    void play();
}

// CDPlayerConfig.java
@Configuration
public class CDPlayerConfig {
    @Bean
    public CompactDisc sgtPeppers() {
        return new SgtPeppers();
    }

    @Bean
    public CDPlayer cdPlayer(CompactDisc cd) {
        return new CDPlayer(cd);
    }
}

// SgtPeppers.java
public class SgtPeppers implements CompactDisc {
    private String title = "Sgt. Pepper's Lonely Hearts Club Band";
    private String artist = "The Beatles";

    @Override
    public void play() {
        System.out.println("Playing " + title + " by " + artist);
    }
}

// CDPlayer.java
public class CDPlayer {
    private CompactDisc cd;

    public CDPlayer(CompactDisc cd) {
        this.cd = cd;
    }

    public void playCD() {
        cd.play();
    }
}
```

## 五、Bean 的使用
终于到了 Bean 的使用部分。被 `@Bean` 或 `@Component` 注释，作为 Bean 的类，并不能直接通过 `new` 来创建实例，而只能通过被 Spring 托管的方式创建，这似乎类似于 jvm。

具体来说，要使用 Bean，需要创建一个 Spring 应用上下文（Application Context）的实例，并设定需要用到的 Bean 所在的配置类或 xml。通过这个实例获取各 Bean 对象实例，才能进行操作。

以 java 配置类为例，我们需要使用 CDPlayer 类，不能直接通过 `new` 创建实例。而是先创建上下文，并将配置类传入。
```java
ApplicationContext context = new AnnotationConfigApplicationContext(CDPlayerConfig.class);
```

接着才能获取 CDPlayer 实例，并调用其方法。
```java
CDPlayer cdPlayer = context.getBean(CDPlayer.class);
cdPlayer.playCD();
```