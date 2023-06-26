---
title: 利用ASM库实现Java反射
tags:
  - 反射
  - ASM
  - Java
categories: 学习笔记
abbrlink: 3a041e16
date: 2023-02-21 18:10:43
---
## 一、ASM库简介
ASM 库是一款基于 Java 字节码层面的代码分析和修改工具。它能分析二进制的 class 文件并对其进行动态修改。ASM 库侧重性能，设计和实现尽可能小和快。

通过 ASM 库，我们可以方便地获取类信息，并实现类似于反射的功能。ASM 库处理字节码，因此能得到仅仅使用反射无法获取的信息，如方法内的结构。

ASM 库同样具有动态修改和创建类文件的功能，但本篇文章主要使用 ASM 库的字节码读取功能。

> 本篇文章假设读者能自行通过Maven或其他方式添加ASM库依赖。

## 二、利用ASM库访问类文件
ASM 库的核心库提供了读取和修改字节码的基本 API。核心库包含如下几个工具类：
### ClassReader
`ClassReader` 类用于从 class 文件中加载字节码。这样，这一 `ClassReader` 对象就拥有了关于某一类的所有信息。

### ClassVisitor
`ClassVisitor` 类可以从 `ClassReader` 中获取想要的信息。

`ClassVisitor` 是访问者模式的访问类。简单解释访问者模式，就是被访问的对象调用访问者类的方法，从而使得访问者获取希望得到的数据。

还是通过 ASM 的例子来理解吧，`ClassVisitor` 实际是一个抽象类，其中定义了一个方法 `visit`。该方法的签名为：
```java
visit(int version, int access, String name, String signature, String superName, String[] interfaces)
```

假设我们继承 `ClassVisitor` 创建了一个新的访问类，那么获取信息的方式如下：
```java
ClassReader cr = new ClassReader("className");
ClassVisitor cv = new ClassVisitor(Opcodes.ASM5) {
    // this is an anonymous class 
};
cr.accept(cv, 0); // second argument is the parsingOptions
```

如上代码中，`ClassReader` 调用 `accept` 方法允许一个 `ClassVisitor`。在 `accept` 方法内部，会按一定解析规则调用 `ClassVisitor` 中的 `visit` 方法。这样我们就可以获得该类的类名、父类名、接口名等等信息。

`ClassVisitor` 类中还有许多访问方法可以重写，如`visitAnnotation`、`visitField`、`visitMethod` 等等。其中 `visitField`、`visitMethod` 等方法的返回值类似 `FieldVisitor`、`MethodVisitor` 等等 ，同样是访问者模式的抽象访问者类型。使用方法与 `ClassVisitor` 类似。

### ClassWriter
`ClassWriter` 用于修改或创建 class 文件，虽然在本篇文章中不会使用，但因为是ASM核心库的组成部分，还是略微介绍一下。

`ClassWriter` 继承了 `ClassVisitor`。因此也具有 `visit` 等方法。不同于 `ClassVisitor` 通过 `ClassReader` 调用访问函数，`ClassWriter` 应由程序员自行调用，并由此让 `ClassWriter` 构建起类结构。最后，调用 `ClassWriter` 的 `toByteArray` 方法生成字节码，通过文件流生成新的文件。

例子如下，这里不赘述
```java
ClassWriter cw = new ClassWriter(0);
cw.visit(V1_5, ACC_PUBLIC + ACC_ABSTRACT + ACC_INTERFACE,"pkg/Comparable", null, "java/lang/Object",new String[]{"pkg/Mesurable"});
cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "LESS","I", null, new Integer(-1)).visitEnd();
cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "EQUAL","I", null, new Integer(0)).visitEnd();
cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "GREATER","I", null, new Integer(1)).visitEnd();
cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "compareTo","(Ljava/lang/Object;)I", null, null).visitEnd();
cw.visitEnd();
byte[] b = cw.toByteArray();

//输出
FileOutputStream fileOutputStream = new FileOutputStream(new File("F:/asm/Comparable.class"));
fileOutputStream.write(b);
fileOutputStream.close();
```

示例出自[java - ASM入门篇 - ksfzhaohui技术专栏 - SegmentFault 思否](https://segmentfault.com/a/1190000040160637)

## 三、利用ASM Tree实现反射
ASM 将各种不同功能的 API 组织在不同的库中。Tree库就是其中之一，能够将解析后的类信息组织成树的形式。即，类用ClassNode表示，ClassNode中包含类的相关信息，以及包含的字段，用FieldNode表示；和方法，用MethodNode表示。

首先，我们需要获取字节码。字节码可能是项目中已经被加载的文件，也可能是未被加载的其他文件。我们可以通过包括包名的类全称获取已加载的类的字节码，也可以通过文件流加载字节码。
```java
var cr = new ClassReader("package.name.ClassName");
```
```java
var in = new FileInputStream("fileName");
var cr = new CLassReader(in);
```

接着我们创建一个 `ClassNode` 实例作为访问和存储类信息的对象。并使用该对象获取类信息。
```java
var cls = new ClassNode();
cr.accept(cls, 0);
```

接着我们就可以获取 `ClassNode` 中保存的类信息。`ClassNode` 的所有字段都是公共的。这样做是因为 `ClassNode` 类只作为保存数据的对象，而没有其他功能。因此使用公共字段是合理的。

```java
    // ClassNode中的一些字段：
    public List<Attribute> attrs;
    public List<InnerClassNode> innerClasses;
    public String nestHostClass;
    public List<String> nestMembers;
    public List<String> permittedSubclasses;
    public List<RecordComponentNode> recordComponents;
    public List<FieldNode> fields;
    public List<MethodNode> methods;
```

这样我们就利用 ASM Tree 库实现了类似反射的功能。

## 四、自定义访问器实现反射
在这一部分，我们自定义一个简单的访问器，实现对类、方法、字段信息的获取。以此为例演示 ASM 核心库的使用方法。

我们创建 `MyClassVisitor` 类继承 `ClassVisitor`。
```java
public class MyClassVisitor extends ClassVisitor
```

添加希望获取的字段
```java
public String className;
public String superClassName;
public List<String> interfacesName = new ArrayList<>();
```

重写 `visit` 方法。这样我们就获取到了类名、父类名和接口名等信息。
```java
@Override
public void visit(int version, int access, String name, String signature, String superName, String[] interfaces)
{
    super.visit(version, access, name, signature, superName, interfaces);
    className = name;
    superClassName = superName;
    interfacesName.addAll(List.of(interfaces));
}
```

类似的，我们创建 `MyMethodVisitor` 和 `MyFieldVisitor` 类，并添加字段
```java
public class MyMethodVisitor extends MethodVisitor {
    public String methodName;
    public String descriptor;
}
public class MyFieldVisitor extends FieldVisitor {
    public String fieldType;
    public String fieldName;
}
```

然后重写 `ClassVisitor` 的 `visitMethod` 和 `visitField`。
```java
@Override
public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions)
{
    var mv = new MyMethodVisitor(Opcodes.ASM5,
            super.visitMethod(access, name, descriptor, signature, exceptions));
    mv.methodName = name;
    mv.descriptor = descriptor;
    methods.add(mv);
    return mv;
}

@Override
public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value)
{
    var fv = new MyFieldVisitor(Opcodes.ASM5,
            super.visitField(access, name, descriptor, signature, value));
    fv.fieldName = name;
    fv.fieldType = descriptor;
    fields.add(fv);
    return fv;
}
```

这样就可以利用 `MyMethodVisitor` 获取 `ClassReader` 中的字节码信息了。
```java
var cr = new ClassReader("org.objectweb.asm.ClassVisitor");
var cv = new MyClassVisitor(Opcodes.ASM5);
cr.accept(cv, 0);
```