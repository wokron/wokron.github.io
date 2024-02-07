+++
title = "Java 和 C# 中的反射机制"
tags = ["C#", "Java"]
categories = ["编程语言"]
aliases = ["/posts/afa82b33"]
date = "2023-02-19T13:59:43+08:00"
+++
## 一、反射简介
反射是一种程序动态访问修改其状态或行为的机制。具体来说，反射提供了在程序运行时对代码进行操作的手段。

反射提高了程序的灵活性和扩展性，能使程序员通过字符串动态地实现程序的修改。它降低了代码间的耦合度，可以避免硬编码，实现代码的组件化。反射的这些特点在对灵活性和扩展性要求很高的框架上有巨大的作用。

当然，也需要注意到，反射通过字符串进行操作的方式是解释性的，这将导致性能的降低。

## 二、反射的原理
对于拥有虚拟机（或者类似的东西，虚拟机只是一个不正规的名词）的语言，都存在加载字节码（同样不正规的名词）到虚拟机的过程。在这个过程中，虚拟机获取了有关类的信息，包括继承关系、包含的字段和方法等等。对于java来说，局部变量放在栈中，类实例放在堆中，程序方法放在方法区中。

对不使用反射的一般情况来说，方法调用更加直接。调用时虚拟机将根据编译时即确定的方法地址进行跳转。而对于反射，则需要根据字符串查找对应的类或方法。这样查找而非硬编码的过程就会影响性能。

## 二、java中的反射
通过反射我们能获取java语言中的如下组成部分：包、类（和接口）、方法（一般/构造）、字段、注解。反射不能操作方法中的内容。

类是面向对象的基本单元，我们通过获取类来实现对程序的动态操作。

### （1）类和包
“类” `Class<?>` 是一个用于表示类的类型。它有三种获取方式
```java
ClassName.class // 类型名.class
obj.getClass(); // 对象.getClass
Class.forName(String className); // Class.forName("包含包路径的类型全称")
```

前两种较好理解，相当于获取一个类型，我们可以用这编写一个类似于 `instanceof` 的函数。
```java
boolean typeof(Object obj, Class<?> cls)
{
    return obj.getClass() == cls;
}
```
> 当然这无法将子类也判断为真。值得注意的是这里我们直接使用了 `==` 而不是 `equals`。因为对于每一个类，Class<?>只用一个实例。

但第三种才最为重要，它通过字符串获取了一个类型。这就为程序提供了灵活性。

获取了一个 `Class` 对象后，我们可以进一步获取其父类和实现的接口
```java
Class<?> cls = Class.forName("somepackages.someClass");
Class<?> superCls = cls.getSuperclass();
Class<?>[] interfaces = cls.getInterfaces();
```
> 值得注意的是获取接口方法返回的是一个 `Class` 数组。这说明对于虚拟机来说本没有类和接口之分。

调用 `getPackage` 可以获取包
```java
Package p = cls.getPackage();
```

`Package` 类型保存了该包的一些信息，如名称、版本等等，不一一列举。

### （2）方法
通过反射机制可以获取类的方法。既可以获取方法集合也可以获取指定的方法。

获取方法集合的方法为
```java
// Class<?> cls = Class.forName("somepackages.someClass");
Method[] ms = cls.getMethods(); // 只获取 public 方法
Method[] ms = cls.getDeclaredMethods(); // 获取所有方法
```

获取特定的方法，需要指定方法签名。也就是方法的名称和其参数类型。
```java
Method m = cls.getMethod("func1", int.class, double.class);
Method m = cls.getDeclaredMethod("func2", int.class, cls);
// 参数类型 (String name, Class<?>... parameterTypes)
```

`Method` 类提供了一些方法以访问方法的相关信息，如名称、参数类型、返回值类型等等。通过函数名很容易理解，这里不一一列举。

如果想要调用方法的话，则需要调用 `Method` 类的 `invoke` 方法。
```java
m.invoke(obj, param1, param2)
// 参数类型 (Object obj, Object... args) 其中第一个参数为要调用该方法的实例
```

对于私有方法，不能直接调用，还需要通过 `setAccessible` 方法设置其为允许调用。
```java
m.setAccessible(true); // is accessible
```

对于构造函数的获取也有类似的方法
```java
Constructor<?>[] cs = cls.getConstructors();
Constructor<?>[] cs = cls.getDeclaredConstructors();
Constructor<?> c = cls.getConstructor(int.class);
Constructor<?> c = cls.getDeclaredConstructor(double.class);
```

需要注意的是，调用构造函数所使用的方法不是 `invoke` 而是 `newInstance`。同时方法不包含对象参数（这是当然的）。
```java
var ins = c.newInstance(param1, param2);
```

### （3）字段
同样使用反射机制获取类的字段，这与获取方法类似。对于获取特定字段，只需要指定字段名即可。
```java
Field[] fs = cls.getFields();
Field[] fs = cls.getDeclaredFields();
Field f = cls.getField("fieldName");
Field f = cls.getDeclaredField("fieldName");
```

我们希望实现字段的读写，这需要通过 `Field` 的 `get` 和 `set` 方法

```java
Object val = f.get(obj);
f.set(obj2, val);
// obj 和 obj2 为拥有被访问字段的实例
```
> 注意进行类型转换。字段的类型是在编译时确定的。

### （4）注解
注解 `Annotation` 是一种特殊的类（就像异常也是一种特殊的类）。它能够被写在类（或接口）、方法、字段或包等等位置，用于为这些元素附加一些内容。如java内置的 `@Override` 注解就表达了 “其所注解的方法重写了父类方法” 的意思。

编译器会在编译期处理一些注解。但在运行阶段注解就只能通过反射访问。如果想要自定义注解，需要继承 `Annotation` 类。
```java
public class MyAnnotation extends Annotation {
}
```
同时还有另一种写法，由此可知注释本质上是接口
```java
public @interface MyAnnotation2 {
}
```
> 但关于注释的具体内容这里还是先打住吧，可能会另写一篇文章详细说明。还是看一下通过反射获取注解的方式吧。

反射中所说明的其他对象都继承了 `AnnotatedElement` 类。这意味着他们都是 “可以被注解的”。他们都具有获取自己拥有的注解的方法。

```java
cls.getAnnotation(MyAnnotation.class);
cls.getAnnotations();
cls.getAnnotationsByType(MyAnnotation2.class);

cls.getDeclaredAnnotation(MyAnnotation.class);
cls.getDeclaredAnnotations();
cls.getDeclaredAnnotationsByType(MyAnnotation2.class);
```
> 注意 `getAnnotationsByType` 获取可重复注解；`DeclaredAnnotation` 指的是直接属于该对象，而非继承得来的注解

不同的反射对象也有特殊的关于注解的方法，如 `Class` 有获取具有注解的方法集合的方法，`Method` 有获取具有注解的参数的方法。不一一列举。

### （5）泛型
java的假泛型设计使得这个语言关于泛型的部分都十分麻烦，这里也是。

反射库中可以看到一些带有 `Generic` 字段的方法。如 `getGenericReturnType`。这些方法返回的类型不是 `Class` 而是 `Type`。`Type` 是 `Class` 的父类。`Type` 类还有用以支持泛型、泛型数组等类型的子类，如 `ParameterizedType`、`GenericArrayType` 这些子类各有方法用于获取其类型参数。

> 也就是说，没有 `Generic` 字段的方法返回的只是类型擦除后的类型。

`ParameterizedType` 具有 `getActualTypeArguments` 方法，可以获取一个泛型类的泛型类型。如 `List<Integer>` 获取 `Integer` 类型。

`GenericArrayType` 具有 `getGenericComponentType` 方法，可以获取一个泛型数组的泛型类型。

## 三、C#中的反射
C# 中的反射机制与java中类似，这里进行对比介绍。

C# 的反射机制能获取如下组成部分，类型、方法（构造、一般）、属性、字段、事件。

### （1）程序集
程序集是C#自有的概念，指的是C#代码编译后产生的exe文件或dll文件。类似于java中的jar。可以动态地加载程序集，这一点类似于java中的 `ClassLoader`，但似乎更加灵活。

可以通过程序集名或程序集路径来加载程序集。
```csharp
Assembly assembly = Assembly.Load("assemblyName");
Assembly assembly = Assembly.LoadFile("filePath");
Assembly assembly = Assembly.LoadFrom("assemblyFile");
```

之后使用类型全名获取类型或获取程序集中的所有类型
```csharp
Type type = assembly.GetType("namespace.className");
Type[] type = assembly.GetTypes(); // 获取程序集中的所有类型
```

通过类型同样可以反过来获取程序集
```csharp
Assembly assembly = type.Assembly;
```

### （2）类型
类型 `Type` 同java `Type`。但没有假泛型导致的复杂子类。除了通过程序集，还有其他的获取类型的三种方式，类似于java。
```csharp
Type t = typeof(TypeName) // typeof(类型名)
Type t = obj.getType(); // 对象.getClass
Type t = Type.getType(String className); // Class.forName("包含命名空间的类型全称")
```

同样的，第三种方式最为关键。另外C#中也可以获取到父类和接口，这些内容不再重复。

### （3）方法
C#同样可以获取构方法集合和特定方法。
```csharp
ConstructorInfo[] cs = t.GetConstructors();
ConstructorInfo? c = t.GetConstructor(new Type[]{type1, type2}); // 参数类型(Type[] types)

MethodInfo[] ms = t.GetMethods();
MethodInfo? m = t.GetMethod("func1", new Type[]{type1, type2});
```

值得注意的是，`ConstructorInfo` 和 `MethodInfo` 都继承了 `MethodBase`。这意味着构造函数和一般函数在C#反射中都是函数，可以使用同样的 `Invoke` 方法调用。
```csharp
object newObj = c.Invoke(new object[]{param1, param2});
m.Invoke(obj, new object[]{param1, param2});
```

### （4）属性
属性相当于C#中的getter、setter函数。也是类的组成部分，也可以通过反射获取。
```csharp
PropertyInfo? p = t.GetProperty("propertyName");
PropertyInfo[] ps = t.GetProperties();
```

通过 `GetValue` 和 `SetValue` 进行属性的修改。
```csharp
object? val = p.GetValue(obj);
p.setValue(obj2, val);
```

### （5）字段
字段类似于java的字段
```csharp
FieldInfo? f = t.GetField("fieldName");
FieldInfo[] fs = t.GetFields();
```

同样通过 `GetValue` 和 `SetValue` 进行属性的修改。
```csharp
object? val = f.GetValue(obj);
f.setValue(obj2, val);
```

### （6）事件
从C#的角度讲，事件是只属于某一类的委托。因此单独作为反射的一部分。
```csharp
EventInfo? e = t.GetEvent("eventName");
EventInfo[] es = t.GetEvents();
```

每个事件具有 `Add` 和 `Remove` 方法。用于注册或删除方法。我们可以获取某一事件对应的方法
```csharp
MethodInfo addHandler = e.GetAddMethod();
MethodInfo removeHandler = e.GetRemoveMethod();
```

之后，创建代理并将其添加到事件或从事件中删除。
```csharp
Delegate d = Delegate.CreateDelegate(tDelegate, this, miHandler); // 创建代理
Object[] handlerArgs = { d };
addHandler.Invoke(eventObj, handlerArgs);
removeHandler.Invoke(eventObj, handlerArgs);
```

### （7）类型成员
如上除程序集和类型之外的组成部分都是类型的成员。我们可以直接获取成员
```csharp
MemberInfo[] members = t.GetMember("memberName"); // 获取该名称的成员
MemberInfo[] members = t.GetMembers(); // 获取类型的所有public成员
```

### （8）成员筛选
我们有时希望之查找具有某些特征的成员，如只搜索私有成员或公共成员。这需要在如上获取成员的方法（不管是如第3节获取特定的成员，如方法。或是如上一节获取一般的成员）上设定参数 `bindingAttr:`。

```csharp
var m = type.GetMethod("func", new Type[]{type1, type2},
    bindingAttr: BindingFlags.Public);
var members = t.GetMembers(
    bindingAttr: BindingFlags.NonPublic);
```

`bindingAttr:` 参数的类型是一个枚举类 `BindingFlags`。包含有许多设置参数，如

|选项|含义|
|---|---|
|IgnoreCase|忽略名称大小写|
|Public|查找公共成员|
|NonPublic|查找非公共成员|
|Static|查找静态成员|
|...|...|

> 如果想要采取不止一种选项，可以对这些枚举值求按位或。如 `bindingAttr: BindingFlags.Public|BindingFlags.NonPublic`

### （9）特性
c#的特性（Attribute）与java的注解几乎一模一样。只不过声明方法略有区别，是用方括号括起来的。
```csharp
[MyAttribute("123")]
public class MyClass {
}
```

C#中获取特性的方法更为优雅。
```csharp
Attribute[] attrs = Attribute.GetCustomAttributes(t);
Attribute attr = Attribute.GetCustomAttribute(
    t, attributeType: typeof(MyAttribute));
```

### （10）泛型
C#中反射的泛型十分简单。
```csharp
Type[] gArgs = t.GetGenericArguments();
```
这样就可以获取类型的泛型参数了（如果不是泛型类则数组长度为0）。对于类内的成员同样适用。