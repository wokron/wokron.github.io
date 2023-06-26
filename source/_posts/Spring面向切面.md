---
title: Spring面向切面
tags:
  - Spring
  - 面向切面
  - AOP
categories: 学习笔记
abbrlink: b452423a
date: 2023-01-10 16:47:00
---
## 一、面向切面（AOP）
有时，在按数据的处理流程编写程序时，我们不得不关心流程之外的情况，比如异常处理安全或日志。这些部分与主要事务交织在一起，使得代码功能不清，造成了强耦合。

面向切片（Aspect-Oriented Programming）就是为解决这样的问题产生的技术。该技术把那些横向影响了应用多处的功能从被其影响的主要事物流程中分离开来，作为切面。使流程只需要关注其本身，而切片则通过其他方式织入程序。

## 二、面向切面的术语
正如面向对象有其术语一样，面向切面也有用于描述其技术的相关概念，在介绍 Spring 的面向切面前需要加以解释。

### 通知（Advice）
通知是切面所具有的行为，也就是不采用面向切面编程时，那些与主要事务无关，应该被抽离出来的代码段。

### 连接点（Join Point）
连接点是可以应用通知的地方，也就是能够执行切面所具有的行为的地方。

### 切点（Poincut）
切点是真正应用通知的地方，切点一定是连接点。

### 切面（Aspect）
切面是通知和切点的总和。当程序执行到切点所在的位置时，就会执行对应的通知。

### 引入（Introduction）
引入是作为切面的类作用到处理主要事务的类的过程。这一过程为处理主要事物的类引入了新的方法和属性，但却没有对这个类本身进行修改。

### 织入（Weaving）
织入是为了实现切面的引入而采取的操作。织入将切面引入目标对象，创建了融合切面和目标的代理对象。这一操作可以发生在编译期、类加载期和运行期，需要看具体的实现。Spring 会在运行期完成切面的织入。

## 三、Spring的AOP：利用切点表达式选择切点
通知和切点共同组成了切面，在这一部分将讲述如何确定切点的位置。我们使用的是称为切点表达式的语法规范，用这一表达式确定我们所指定的切点，以便之后通知的编写。

Spring AOP 使用的是 AspectJ 切点指示器中的一部分，包括如下的内容

| AspectJ指示器| 描述|
|---|---|
| arg()       |限制连接点匹配参数为指定类型的执行方法|
| @args()     |限制连接点匹配参数由指定注解标注的执行方法|
| execution() |用于匹配是连接点的执行方法|
| this()      |限制连接点匹配AOP代理的bean引用为指定类型的类|
| target      |限制连接点匹配目标对象为指定类型的类|
| @target()   |限制连接点匹配特定的执行对象，这些对象对应的类要具有指定类型的注解|
| within()    |限制连接点匹配指定的类型|
|@within()   |限制连接点匹配指定注解所标注的类型（当使用Spring AOP时，方法定义在由指定的注解所标注的类里）|
|@annotation |限定匹配带有指定注解的连接点|

通过组合使用这些类似于函数的指示器的，就可以确定我们想要引入通知的连接点究竟在哪里。接下来我们通过一些示例展示切点表达式的使用。如
```c
execution(* concert.Performance.perform(..))
```
表示匹配一个在方法执行时触发的连接点，这个方法是 `concert` 包下的 `Performance` 类中名为 `perform` 的方法，不考虑方法的返回值和参数（匹配所有同名的 `perform` 方法）。

切点表达式中还可以使用与（&&）或（||）非（!）操作。如
```c
execution(* concert.Performance.perform(..)) && within(concert.*)
```
表示同时满足是 concert 包下的类方法调用，并且满足上一条的条件的切点。

另外因为 Spring 具有依赖注入的功能，因此添加了一个新的切点指示器 `bean()` 用来通过 bean id 选择切点。
```c
execution(* concert.Performance.perform()) && bean('woodstock')
```

## 四、Spring的AOP：基于注释的切面创建
### （1）创建第一个切面
我们还以 CDPlayer 为例子，创建一个 Listener 作为切片。
我们首先要为切面类添加注释 `@Aspect`。
```java
@Aspect
public class Listener
```

之后我们定义一些方法作为通知。并通过注释将其引入 `CDPlayer` 类。我们会在如下注释中选择：

| 注解            | 通知调用位置|
| ---             | --- |
| @After          | 通知方法会在目标方法返回或抛出异常后调用|
| @AfterReturning | 通知方法会在目标方法返回后调用|
| @AfterThrowing  | 通知方法会在目标方法抛出异常后调用|
| @Around         | 通知方法会将目标方法封装起来|
| @Before         | 通知方法会在目标方法调用之前执行|

```java
@Aspect
public class Listener
{
    @Before("execution(* playCD()) && bean(cdPlayer)")
    public void waitingForMusic() {
        System.out.println("waiting for listening...");
    }

    @AfterReturning("execution(* playCD()) && bean(cdPlayer)")
    public void appreciateTheMusic() {
        System.out.println("this music is amazing!!");
    }

    @AfterThrowing("execution(* playCD()) && bean(cdPlayer)")
    public void handleSomethingWrongWithTheCDPlayer() {
        System.out.println("there is something wrong, I need to fix this player.");
    }

    @After("execution(* playCD()) && bean(cdPlayer)")
    public void leaveAfterPlayingMusic() {
        System.out.println("music finished, I will leave.");
    }
}
```
如上，我们确定了切点为 `CDPlayer` 的 `playCD` 方法，并对调用该方法之前，成功返回之后和抛出异常之后分别设置了通知。这样切面就创建完成了。接下来我们还需在配置类中将切面设置为 Bean，并添加注释 `@EnableAspectJAutoProxy` 以启用切片。这样所有的配置就完成了。
```java
@Configuration
@EnableAspectJAutoProxy // here!!!
@PropertySource("classpath:application.properties")
public class CDPlayerConfig
{
    @Bean
    public CompactDisc blankDisc(
            @Value("${disc.title}") String title,
            @Value("${disc.artist}") String artist)
    {
        return new BlankDisc(title, artist);
    }

    @Bean
    public CDPlayer cdPlayer(CompactDisc cd)
    {
        return new CDPlayer(cd);
    }

    // here!!!
    @Bean
    public Listener listener() {
        return new Listener();
    }
}
```

### （2）切点的直接声明和使用
注意在定义切面的时候，切点表达式 `"execution(* playCD()) && bean(cdPlayer)"` 出现了许多次，这无疑是代码的重复。我们可以利用注释 `@Pointcut` 声明该表达式，以便重用。
```java
@Aspect
public class Listener
{
    @Pointcut("execution(* playCD()) && bean(cdPlayer)")
    public void playCDPointcut() {}

    @Before("playCDPointcut()")
    public void waitingForMusic() {
        System.out.println("waiting for listening...");
    }

    @AfterReturning("playCDPointcut()")
    public void appreciateTheMusic() {
        System.out.println("this music is amazing!!");
    }

    @AfterThrowing("playCDPointcut()")
    public void handleSomethingWrongWithTheCDPlayer() {
        System.out.println("there is something wrong, I need to fix this player.");
    }

    @After("playCDPointcut()")
    public void leaveAfterPlayingMusic() {
        System.out.println("music finished, I will leave.");
    }
}

```
注意 `playCDPointcut()` 函数的声明和其他方法注释中值的变化。

### （3）环绕通知的使用
Spring AOP 中最强大的是环绕通知，由注释 `@Around` 声明。其他的通知调用位置注释不过是环绕通知的一部分。环绕通知的使用方法有些特殊，因此在这一节中单独讲解。

首先需要明确的是，切面的本质不过是一个代理（proxy），代理类的方法调用委托类中的对应方法，并在调用的前后添加一些额外的代码，从而实现更多功能。AOP 的作用不过是去除了声明代理类的过程，减少了事务与切面间的耦合而已。

环绕通知的使用也类似，如下就是通过环绕通知实现的与上一节等价的代码：
```java
@Aspect
public class Listener
{
    @Pointcut("execution(* playCD()) && bean(cdPlayer)")
    public void playCDPointcut() {}

    @Around("playCDPointcut()")
    public void listenCDPlayer(ProceedingJoinPoint pjp) {
        try {
            System.out.println("waiting for listening...");
            pjp.proceed();
            System.out.println("this music is amazing!!");
        }
        catch (Throwable e) {
            System.out.println("there is something wrong, I need to fix this player.");
        }
        finally {
            System.out.println("music finished, I will leave.");
        }
    }
}
```

### （4）处理参数
调用方法时，可能会传入参数，这些参数也可以被通知所处理。

我们可以在 CDPlayer 中添加一个方法 `playCD(int)` 设定播放的次数。然后获取该参数。
```java
public void playCD(int times) {
    for (int i = 0; i < times; i++) {
        playCD();
    }
}
```

接着我们重写 `Listener` 切面。如下的代码我们将 times 传入了通知方法 `listenCDPlayer` 中。
```java
@Aspect
public class Listener
{
    @Pointcut("execution(* playCD(int)) && bean(cdPlayer) && args(times)")
    public void playCDPointcut(int times) {}

    @Before("playCDPointcut(times)")
    public void listenCDPlayer(int times) {
        System.out.printf("this music will play %d times.\n", times);
    }
}
```

> 当使用多个参数时，函数中的类型名和 args() 中的参数名都用逗号隔开，如 `args(arg1, arg2)`