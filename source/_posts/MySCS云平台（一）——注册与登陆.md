---
title: MySCS云平台（一）——注册与登陆
tags: MySCS云平台
categories: 软件开发
abbrlink: 74b7a513
date: 2022-09-19 18:16:07
---
## 一、注册与登陆模块
该模块执行注册和登陆操作。从本质上说，也就是进行用户信息的添加，以及更改当前登陆的用户信息。因此我们还要定义一个储存用户信息的类
## （1）用户信息（UserInfo）
为什么是UserInfo而不是User呢？因为用户的操作是读入的命令行，所以并不需要一个User类来执行操作。在注册登录模块只需要显示用户的信息而已。（或许多用户的时候User类会出现？）

### 方法
重写toString方法
### 字段
1. 学工号
2. 姓、名
3. 邮箱
4. 密码
   
### 算法
暂无。注意构造函数只能有包括全部信息的。

## （2）注册与登陆控制器（LoginController）
用来控制注册与登陆，在执行不正确的操作时会抛出异常。

### 方法
注册：无返回值，参数为用户的各项信息，用来创建一个新的用户。当用户信息不合法、已存在、密码不一致等情况发生时抛出异常
```java
void register(String ID, String firstName, String lastName, String email, String password, String passwordConfirm);
```

登陆：无返回值，参数为一个学工号和一个密码，用来登陆一个用户。当用户不存在、密码不正确或已经登录的时候抛出异常
```java
void login(String ID, String password);
```

打印信息：无返回值，无参数或有一个学工号作为参数，打印当前登陆的信息或学号对应的人的信息。当未登录、输入的学号不合法、不存在等等情况发生时抛出异常
```java
void printInfo();
void printInfo(String ID);
```

退出登陆：无返回值、无参数。退出当前登陆，如果当前没有登陆则抛出异常。
```java
void logout();
```

### 字段
1. 储存用户信息的列表（`List<UserInfo>`）
2. 当前登陆的用户`(UserInfo`)
3. 当前程序所处的状态（`LoginStatus`枚举）

### 算法
一个有限状态机，包含这样几种状态：未登录、学生登陆、教师登录。利用函数进行状态转换。不合理的转换会抛出异常。