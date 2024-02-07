+++
title = "后端开发入门笔记之脚本"
tags = ["Shell", "MySQL", "Python"]
categories = ["开发"]
series = ["后端开发入门笔记"]
aliases = ["/posts/2594d393"]
date = "2023-04-25T18:10:33+08:00"
+++
## 一、前言
这篇文章只是记录一些自己编写脚本时用到的零碎知识而已。这也是这一系列的最后一篇。
<!-- more -->

## 二、shell 的异常处理
有时我们希望在 shell 脚本命令执行出现异常时进行处理，比如说输出异常情况或退出等等。我们知道发生异常时返回值不为 0，如果是在 c 语言中我们可以这样处理
```c
if (!do_something()) {
  // handle exception
}
```

类似的在 shell 中
```sh
dosomething arg1 arg2
if [ $? -ne 0 ]
then
  # handle exception
if
```

但是这样编写起来太过麻烦了，我们可以采用另一种方法，那就是使用短路逻辑运算符。

我们还先以 c 语言为例，假设现在我们有两个函数，对他们取逻辑或
```c
int r = func1() && func2()
```

那么在 `func1` 的返回值为 0 时，就会发生短路，不执行 `func2`。而在返回值为；而当 `func1` 的返回值不为 0 时，则会继续执行 `func2`。我们可以让 `func2` 完成 `func1` 的异常处理。

当然，在 c 语言中这种方法很是牵强，因为不同函数的栈帧并不一样，很难跨函数进行处理。可是 shell 中就不同了，所有的变量都是全局变量。

但还需要注意一点，shell 中 0 表示真，1 表示假。（因为 0 表示程序正常结束，所以为真。）所以在 shell 中就需要使用 `||` 而非 `&&`。

```sh
cmd1 arg1 arg2 || cmd2 arg3 arg4
```

举我写的脚本中的例子。
```sh
echo "try to import csv files in $source_path, execute insert.py script to do this"
python3 ./insert.py $host $port $username $password $source_path $database || ! echo "error: fail to import all csv files" || exit
echo "import csv files success!!!"
```

需要注意这里用 `||` 连接了三条命令，其中第二条输出错误提示，我们为了在该命令正常执行后依旧执行后面的 `exit` 操作，在该命令前面加上 `!` 进行取反。需要注意这里不能将 `!` 和 `echo` 连接起来 `!echo`。

另外在学习使用命令行时你一定见过这样的操作，只是使用的是 `&&`。
```sh
cmd1 arg1 arg2 && cmd2 arg3 arg4
```

虽然各位应该理解两者的区别，但这里还是说明一下：
- `||` 表示如果发生错误，才执行下一条
- `&&` 表示只要不发生错误，就不断依次执行

## 三。执行 sql 脚本
当进行项目维护的时候，可能需要使用大量的 sql 语句，这时将这些语句编写成 sql 脚本，批量执行更有效率。

我自己的数据导入脚本就有用到了 sql 脚本。实际上执行脚本不过就是进行了一个文件重定向而已。
```sh
echo "try to create tables by create.sql..."
mysql -h $host -P $port -u $username -p$password < ./create.sql || ! echo "error: fail to execute create.sql" || exit
echo "tables are now created"
```

当然还有使用管道的方法，如果想要执行多个 sql 脚本，可以使用通配符将文件连接起来统一执行
```sh
cat ./*.sql | mysql -h $host -P $port -u $username -p$password
```

还可以在命令行中执行单一语句，这需要在使用 `mysql` 命令时设置 `-e` 参数
```sh
mysql -h $host -P $port -u $username -p$password -e "SHOW DATABASES"
```

## 四、csv 数据导入
mysql 中有 `LOAD DATA INFILE` 语句用于文件导入，不过其他数据库似乎并不支持这种方法。更普适的方法还是通过 `INSERT` 插入数据。这里我写了一个 python 脚本用于通过 `insert` 从 csv 文件中导入数据。我个人认为还是编写地比较灵活，可以通过简单修改导入不同数据。

```py
from sys import argv

import pymysql

insert_template = """
insert into {}({})
values ({})
"""

tables = ['departments', 'employees', 'titles', 'dept_emp', 'dept_manager']

_, host, port, username, password, source_path, database = argv
port = int(port)

error = False

db = pymysql.connect(
    host=host,
    user=username,
    passwd=password,
    database=database,
    port=port,
)

cursor = db.cursor()

for table in tables:
    print(f"start to import csv file to table {table}...")
    with open(f'{source_path}/{table}.csv', 'r') as f:
        args = list(map(lambda line: tuple(line.strip().split(',')), f.readlines()))
        fields = ', '.join(args[0])
        argc = len(args[0])
        del args[0]

    sql = insert_template.format(table, fields, ', '.join(['%s']*argc))
    try:
        cursor.executemany(sql, args)
        db.commit()
        print(f"import csv file to table {table} success!!!")
    except Exception as e:
        db.rollback()
        print(f"error: fail to import into table {table}, {e}")
        error = True

db.close()

if error:
    exit(1)
```

这里使用了 `executemany` 用于一次执行许多条类似的语句，相较于逐个插入数据有很大的速度提升。这似乎是因为 `INSERT` 语句执行后默认会直接提交到数据库，写入磁盘。过多的 `INSERT` 语句就会增加大量的磁盘写入时间。

直接使用 sql 插入数据也有同样的问题，一个解决办法是在进行多次插入之前禁用自动提交 `SET AUTOCOMMIT = 0;`，再插入结束后再开启 `SET AUTOCOMMIT = 1;`。

这里我们举一例子，如下的 sql 语句创建了一个存储过程，该存储过程会向表 `table_random` 中插入 400000 条随机生成的数据。在 `WHILE` 循环的前后，我们设置了 `AUTOCOMMIT` 的值以禁用和开启自动提交。可以试着注释这两条语句，在注释前和注释后分别执行该存储过程，看一下完成所需时间。

```sql
DELIMITER $$
CREATE PROCEDURE random_insert()
BEGIN
	DECLARE i INTEGER DEFAULT 0;
	DECLARE sparse_val INTEGER;
	DECLARE dense_val INTEGER;
	DECLARE num INTEGER;

	SET AUTOCOMMIT = 0;

	WHILE i < 400000 DO
		SET sparse_val = ROUND(RAND()*5000001 - 0.5);
		SET dense_val = ROUND(RAND()*10 - 0.5);

		INSERT INTO table_random(sparse, dense)
		VALUES (sparse_val, dense_val);

		SET i = i + 1;
	END WHILE;

  SET AUTOCOMMIT = 1;
	
END $$
DELIMITER ;
```