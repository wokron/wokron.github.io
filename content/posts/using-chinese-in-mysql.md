+++
title = "MySQL 的中文插入问题"
tags = ["MySQL"]
categories = ["开发"]
aliases = ["/posts/4754b2bd"]
date = "2022-12-31T15:24:00+08:00"
+++
## 一、问题描述
在执行一个sql脚本的时候出现了这样的问题：
```text
ERROR 1366 (HY000): Incorrect string value: '\xA3\x8E\xE6\x9C\x89\xE5...' for column 'name_zh' at row 1
```

问题很明显是由于中文引起的。查看报错的语句：
```sql
INSERT INTO `admin_menu` VALUES ('1', '/admin', 'AdminIndex', '首页', 'el-icon-s-home', 'AdminIndex', '0');
```
是 “首页” 无法被识别。考虑是字符集的问题。

## 二、问题处理
### 情况一：数据库字符集与脚本所用字符集不一致
第一种可能是数据库所用的字符集和脚本字符集不一致，比如说建表时使用的字符集为 `latin1`，而脚本使用的却是 `utf-8`，这样就会导致错误。

输入命令，查看建库语句
```sql
SHOW CREATE DATABASE wj;
```
```text
+--------------+----------------------------------------------------------------------------------------------------------------------------------------+
| Database     | Create Database                                                                                                                        |
+--------------+----------------------------------------------------------------------------------------------------------------------------------------+
| white_jotter | CREATE DATABASE `wj` /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci */ /*!80016 DEFAULT ENCRYPTION='N' */ |
+--------------+----------------------------------------------------------------------------------------------------------------------------------------+
```
显示所用字符集为 `utf8mb4`，说明数据库字符集与脚本所用字符集一致。

> 若不一致，应该使用如下指令来修改
> ```sql
> ALTER DATABASE 数据库名 CHARACTER SET utf8mb4;
> ```


> 注意平常所说的 utf8 指 sql 中的 utf8mb4，而不是 sql 中的 utf8 或 utf8md3。sql 将 utf8md3 视为 utf8 似乎有历史的原因。utf8mb4 兼容 utf8md3，并可以表示更多的符号，一些表情符号也包含在 utf8mb4 中。

### 情况二：表字符集或列字符集与脚本所用字符集不一致
同样输入命令，查看建表语句
```sql
SHOW CREATE TABLE admin_menu;
```
```text
+------------+------------------------------------------------------------------------+
| admin_menu | CREATE TABLE `admin_menu` (
  `id` int NOT NULL AUTO_INCREMENT,
  `path` varchar(64) DEFAULT NULL,
  `name` varchar(64) DEFAULT NULL,
  `name_zh` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `icon_cls` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `component` varchar(64) DEFAULT NULL,
  `parent_id` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=17 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
+------------+------------------------------------------------------------------------+
```

显示表字符集和列字符集所用字符集依旧为 `utf8mb4`。

> 类似的，如果不一致，应该使用如下指令来修改
> ```sql
> ALTER TABLE 表名 CHARACTER SET utf8mb4;
> ```

### 情况三：客户端字符集与脚本所用字符集不一致
所谓的客户端字符集指的是用户输入的命令所用的字符集。

通过如下指令查看：
```sql
SHOW variables LIKE '%char%';
```
```text
+--------------------------+-------------------------------------------+
| Variable_name            | Value                                     |
+--------------------------+-------------------------------------------+
| character_set_client     | gbk                                       |
| character_set_connection | gbk                                       |
| character_set_database   | utf8mb4                                   |
| character_set_filesystem | binary                                    |
| character_set_results    | gbk                                       |
| character_set_server     | utf8mb4                                   |
| character_set_system     | utf8mb3                                   |
| character_sets_dir       | D:\MySQL\MySQL Server 8.0\share\charsets\ |
+--------------------------+-------------------------------------------+
```
第一行 `character_set_client` 便是用户字符集，为 `gbk` 而非 `utf8mb4`。这就是 sql 指令中出现中文会产生错误的原因。

使用指令
```sql
set character_set_client=utf8mb4;
```
可以修改 `character_set_client` 为 `utf8mb4`。

类似的命令也可以修改 `character_set_connection` 和 `character_set_results`。这两个字符集分别是连接数据库的字符集和数据库向客户端返回时的字符集。另外这三个字符集也可以通过一条命令同时修改：
```sql
SET NAMES utf8mb4;
```

注意这样的设置在重启后将会还原，永久设置字符集可以通过修改配置文件 `my.ini` 实现（[永久修改字符集](https://stackoverflow.com/questions/20570246/change-database-variable-character-set-client-in-mysql)）。
