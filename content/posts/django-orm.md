+++
title = "Django 创建表结构"
tags = ["Django"]
categories = ["开发"]
aliases = ["/posts/c837c423"]
date = "2023-01-09T13:44:16+08:00"
+++
## 一、对象关系映射（ORM）
面向对象与关系型数据库中的数据存储方式并不匹配，因此为了在编程中实现对数据库的操作，需要 ORM 技术将数据库中数据映射到对象之中。通过对对象的操作，做到对数据库的操作，从而实现增删改查等功能。

> 面向对象编程中，“关系”体现在类与其属性之间以及同一类的不同属性之间；而在关系型数据库中，“关系”体现在同一表中的不同列字段中。

## 二、django 的 ORM 实现
django 通过 `models.Model` 类来对应数据库中的表结构，如
```py
class User(models.Model):
```
即表示了 `User` 类，也对应了数据库中的 user 表。django 会根据类的声明自动在数据库中建表。

django 用类中的属性来表示表的列字段。如
```py
class User(models.Model):
    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=50)
    password = models.CharField(max_length=50)
```
定义了 user 表的主键、名字、密码信息。

表间关系通过类与其属性表示，如
```py
class User(models.Model):
    id = models.IntegerField(primary_key=True)
    name = models.CharField(max_length=50)
    password = models.CharField(max_length=50)
    courses = models.ManyToManyField(to="Course")
```
`courses` 属性表示用户具有的课程，反映了用户与课程之间的关系。

## 三、类属性与数据库字段的对应
如上一节所述，django 建立了类属性与数据库字段之间的对应关系。常见的如
```py
models.CharField
models.TextField
models.IntegerField
models.FloatField
```
就对应了数据库中的 char(n)，text，int，float 类型。不一一详述。

## 四、主键与外键
在表中要定义主键，作为该表一条记录的唯一标识。相应的，在 django 中定义主键需要在对应属性上添加 `primary_key=True` 参数。例如
```py
id = models.IntegerField(primary_key=True)
```
未定义主键时，django 也会自动创建一个主键。

通过外键可以设置表间的关系。外键通过属性 `models.ForeignKey` 定义。如
```py
class Task(models.Model):
    id = models.IntegerField(primary_key=True, null=False)
    name = models.CharField(max_length=50, null=False)
    course = models.ForeignKey(to="Course", on_delete=models.CASCADE)
```
就定义了一个外键，对数据库来说，保存的是 course 表中的主键信息。但对 Task 类来说，则是定义了一个 Course 类型的属性。但这两种观点的本质是一样的，都是建立了 task 到 course 的关系。更准确地说，是多到一的关系（不同的 task 可以对应同一个 course）。

> 注意建立表间关系的时候，需要设置 on_delete 参数，表示外键所对应的记录被删除时，本记录的行为。在例子中 CASCADE 表示外键对应的删除后，本记录也删除。

## 五、表间关系的创建
外键是创建表间关系的基础。这一节将更深入讨论表间关系。表间关系可以分为三类，分别是一对一、多对一和多对多。可以通过如下例子进行理解：
- 一对一：一个人对应唯一的身份证，一张身份证对应唯一的一个人。
- 多对一：一个班级有许多学生，而一个学生只属于某个班级。
- 多对多：一门课有许多的学生，一个学生会上不同的课。

### 一对一的创建
在数据库中，一对一的关系比较随意，可以在 A 表中包含 B 表的主键，也可以在 B 表中包含 A 表的主键。只需要保证外键的唯一性即可。

django 设置一对一关系，需要定义 `models.OneToOneField` 的属性。如在 `Person` 类中有
```py
id_card = models.OneToOneField(to="IDCard", on_delete=models.RESTRICT)
```
这样就可以通过访问 `Person` 类的属性 `id_card` 来访问对应的表格数据了。

> models.RESTRICT 表示如果存在关系，则删除会产生异常

### 多对一的创建
建立多对一关系，在数据库中需要让多的一方设置外键，包含一的一方的主键。

而在 django 中，则只需要添加一个外键即可实现。如第五节中的例子
```py
course = models.ForeignKey(to="Course", on_delete=models.CASCADE)
```

### 多对多的创建
多对多关系不能通过在任何一方或两方中添加外键实现。需要建立第三个表，称中间表或关系表，在该表中同时包含原来两个表的主键。

django 通过添加 `models.ManyToManyField` 创建多对多关系。如
```py
class Course(models.Model):
    id = models.CharField(max_length=10, primary_key=True, null=False)
    name = models.CharField(max_length=50, null=False)
    users = models.ManyToManyField(to=User)
```
这样将自动创建一个中间表 course_user，包含 course 表和 user 表的主键。

但是这样只能创建由 Course 到 User 的单向联系。通过 Course 可以访问 User，但从 User 却不能访问 Course。可以自行定义中间表来实现双向多对多关系。

> 注意，如果在 User 表中添加属性 `courses = models.ManyToManyField(to=Courses)` 并不能解决问题。这只会创建另一个表 user_course，最终形成两个单向多对多关系。

正确的方式是通过 `through=` 参数指定中间表。
```py
class Course(models.Model):
    id = models.CharField(max_length=10, primary_key=True, null=False)
    name = models.CharField(max_length=50, null=False)
    users = models.ManyToManyField(to=User, through="Course2User")


class Course2User(models.Model):
    id = models.IntegerField(primary_key=True, null=False)
    course = models.ForeignKey(to="Course", on_delete=models.CASCADE)
    user = models.ForeignKey(to=User, on_delete=models.CASCADE)
```

这样就可以通过中间表查询某个 `User` 对应的 `Course` 了。