---
title: Java注解与C#特性
tags:
  - C#
  - Java
categories: 学习笔记
abbrlink: 99f60659
date: 2023-02-20 12:45:30
---
## 一、元数据简介
元数据是指用来描述数据的数据。对编程语言来说，元数据可以为程序中元素添加额外的信息。这一功能可以被用于描述代码间关系，以及代码与其他资源的联系。

元数据可以被用于框架中。通过元数据可以实现在代码上对类的直接配置，避免编写如 xml 的配置文件。

Java 和 C# 都具有为程序中元素，如类、方法等等，添加元数据的方式。Java中称为注解，而C#中称为特性。注解和特性都通过反射获取，关于两者的获取方式，已经在前一篇文章中有所记述。本篇只讨论注解和特性的定义和使用。

## 二、Java注解
### 注解的定义
如果想要自定义注解，需要继承 `Annotation` 类。
```java
public class MyAnnotation extends Annotation {
}
```
同时还有另一种写法，由此可知注释本质上是接口
```java
public @interface MyAnnotation2 {
}
```
> 和接口一样，注解可以添加静态变量

我们可以为注解添加“参数”，比如 Spring 中的注解 `@Profile(value = "dev")`。我们为其设置了 `value` 属性。

但是注解本是接口，不可能有保存数据的能力，要如何设置注解的属性呢？实际上，注解的属性在编写 `Annotation` 时是一个方法。即

```java
public @interface MyProfile {
    String value();
}
```

还可以为参数添加默认值
```java
public @interface MyAnnotation3 {
    String value();
    int number() default 8; // set default number
}
```

注解经常和枚举一起使用。

```java
public @interface MyAnnotation4 {
    enum SELECT_TYPE {TYPE_A, TYPE_B}
    SELECT_TYPE selectType() default TYPE_A;
}
```

### 元注解
元注解是注解的元数据，也就是对注解的注解。元注解为注解设置额外的信息，如设置注解的作用对象等等。

#### @Documented
设置@Documented表明该注解能够出现在javadoc中。默认注解不会出现在javadoc中。

#### @Target
设置注解的作用对象，如注解类、方法、构造方法等等。使用方式如下
```java
@Target({ElementType.METHOD, ElementType.CONSTRUCTOR})
public @interface MyAnnotation5 {
}
```
`@Target` 表明注解 `@MyAnnotation5` 只能修饰方法和构造函数。

#### @Retention
@Retention 设置注解被保留的时间。分为“仅在源文件”、“在class文件中保留”和“直到运行时依旧存在”。对应枚举 `RetentionPolicy.SOURCE`、`RetentionPolicy.CLASS`和`RetentionPolicy.RUNTIME`。

#### @Inherited
@Inherited 表示该注解所修饰的类，其子类也具有该注解。即注解也可被继承。

#### @Repeatable
@Repeatable 表示该注解可以多次修饰同一个地方。例如

```java
@RepeatableAnnotation("123")
@RepeatableAnnotation("456")
public class TestClass {
}
```
### 注解的使用
注解可以看做存储有一定信息的类型。使用注解就是获取其存储的信息（或者特定的注解类型本身便可作为一种信息）。获取注解信息只需要调用注解接口定义的方法即可。

## 三、C#特性
### 特性的定义
在C#中，特性就是一个类。可以定义类的所有成员种类，如方法、属性、字段等等，获取后也可以像一般类实例一样使用。自定义特性只需要继承 `Attribute` 类即可。

```csharp
public MyTestAttribute : Attribute
{
    public string Value { get; set; }

    public MyTestAttribute() {}

    public MyTestAttribute(string value)
    {
        Value = value;
    }
}
```
约定特性类名称以 Attribute 结尾。要使用特性时，既可以使用特性类名，也可以使用去掉末尾 Attribute 后的名称。
```csharp
[MyTestAttribute("123")]
public TestClass
{ }

[MyTest("456")]
public TestClass2
{ }
```
> 注意特性类作为特性使用构造函数时传入的参数只能是简单类型如 `bool, int, double, string, Type, enums` 等等。

同时特性类型中的属性也可以在特性修饰时设置，还举 `MyTest` 为例。使用方法为 `propertyName = constPropertyValue`。
```csharp
[MyTest(Value = "789")]
public TestClass3
{ }
```

### 特性设置
类似于Java注解的元注解，也有对特性进行设置的方式。只需要使用 `AttributeUsage` 注解。
```csharp
[AttributeUsage(AttributeTargets.Class|AttributeTargets.Method, AllowMultiple = true, Inherited = true)]
public class MyTest2Attribute : Attribute
{ }
```

上面的例子展示了可以对特性进行的所有设置：作用对象、是否可重复、是否可继承。