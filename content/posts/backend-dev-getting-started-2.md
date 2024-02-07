+++
title = "后端开发入门笔记之 Flask 简介"
tags = ["Flask"]
categories = ["开发"]
series = ["后端开发入门笔记"]
aliases = ["/posts/b7c97beb"]
date = "2023-04-22T22:40:12+08:00"
+++

## 一、前言
这一篇文章中主要讲解了为完成后端项目所使用的 Flask 框架。对于自己所使用的功能和特性进行了着重的讨论。当然说是着重讨论，其实还是皮毛罢了。想要更加系统地了解 Flask，还请查看[官方文档](https://dormousehole.readthedocs.io/en/latest/index.html)

<!-- more -->
## 二、Flask
### （1）Flask 介绍
Flask 是一个 Python Web 框架，十分轻量灵活，可以用于开发小型 Web 应用。Flask 高度可扩展，可以通过添加不同的组件来实现定制化的功能。对于简单的后端任务来说，Flask 十分合适。

### （2）Flask 用于后端
虽然 Flask 可以实现经典的 MVC 架构，但是本项目只进行后端开发，提供一些用于数据库操作的 RESTful 接口，并不涉及显示的部分。因此只讨论 Flask 用于后端开发的方面。

对于后端项目，我们希望其向下管理数据库，向上为前端提供接口。这就需要用到 ORM 来通过对象管理数据库关系；同时通过 Route 提供 api 接口。

另外，我们还需要对项目进行管理，以合理的结构组织项目。我们需要了解 Flask 的 BluePrint。

## 三、ORM
### （1）数据库设置
Flask 使用 Flask-SQLAlchemy 组件实现 ORM。SQLAlchemy 本是独立于 Flask 的 ORM 库，但在 Flask 中使用时，又针对 Flask 进行了一定封装。

我们希望创建到数据库的连接，这需要在后端项目中进行设置。对 Flask 来说，设置以字符串的形式保存于项目的配置变量 `SQLALCHEMY_DATABASE_URI` 中。
```py
app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = f'mysql+pymysql://{username}:{password}@{host}:{port}/{name}?charset=utf8mb4'
```
> 配置存储于 `app.config` 中。其中 `app` 是一个 `Flask` 类型的对象，用于表示 Flask 项目这个整体。

通过查看上面的代码，应该可以理解该配置的具体格式是样子的了。比如说
```py
'mysql+pymysql://root:123456@localhost:3306/db_test?charset=utf8mb4'
```

表示连接的数据库为 mysql，用户名为 root，密码为 123456，连接地址为 localhost:3306，选用的数据库名为 db_test，使用字符集为 utf8mb4。

我们通过这样的设置创建数据库连接，命名为 `db`。接下来我们所有的 ORM 操作都要通过 `db` 实现。

```py
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy(app)
```
### （2）字段设置
在数据库中有着不同的表格，有着各自的列，记录着一条条数据。反映到 SQLAlchemy 中，表格对应着类，所有表示着表格的类都是 `db.Model` 的子类；而表的列对应着类的字段，类的字段的类型都为 `db.Column`，通过设置 `db.Column` 的属性来表示不同的列。

举例来说，我们想要设置 departments 表，它有 `dept_no` 和 `dept_name` 作为列。那么对应的类表示如下
```py
class Department(db.Model):
    __tablename__ = "departments"
    dept_no = db.Column(db.String(4), nullable=False, primary_key=True)
    dept_name = db.Column(db.String(40), nullable=False, unique=True)
```

我们将 `Department` 类作为 departments 表的对应（`__tablename__` 字段用于表示表名，必须设定，否则以类名作为表名）。通过 `db.Column` 设置列的约束，如类型约束、是否可空、是否主键、是否唯一，以及索引等等内容。

需要注意 `db.String`，Flask-SQLAlchemy 使用这种方式设定字段的类型。类似的还有 `db.Integer`，`db.Enum`，`db.Date` 等等。它们的使用可看如下的例子
```py
emp_no = db.Column(db.Integer, primary_key=True, nullable=False)
gender = db.Column(db.Enum('M', 'F'), nullable=False)
birth_date = db.Column(db.Date, nullable=False)
```

### （3）关系设置
我们还需要为数据库对象间设定关系，即一对一、一对多、多对多关系。一对一、一对多关系只需要将某一列设定为外键即可，较为简单，因此这里只考虑多对多关系。我们先看如下的例子，其中展示了如何实现多对多关系
```py
class Department(db.Model):
    __tablename__ = "departments"
    dept_no = db.Column(db.String(4), nullable=False, primary_key=True)
    dept_name = db.Column(db.String(40), nullable=False, unique=True)
    # relation
    employees = db.relationship('Dept2Emp', back_populates='department')


class Employee(db.Model):
    __tablename__ = "employees"
    emp_no = db.Column(db.Integer, nullable=False, primary_key=True)
    birth_date = db.Column(db.Date, nullable=False)
    first_name = db.Column(db.String(14), nullable=False, index=True)
    last_name = db.Column(db.String(16), nullable=False)
    gender = db.Column(db.Enum('M', 'F'), nullable=False)
    hire_date = db.Column(db.Date, nullable=False)
    # relation
    departments = db.relationship('Dept2Emp', back_populates='employee')


class Dept2Emp(db.Model):
    __tablename__ = "dept_emp"
    emp_no = db.Column(db.Integer, db.ForeignKey('employees.emp_no', ondelete='cascade'), primary_key=True,
                       nullable=False)
    dept_no = db.Column(db.String(4), db.ForeignKey('departments.dept_no', ondelete='cascade'), primary_key=True,
                        nullable=False, index=True)
    from_date = db.Column(db.Date, nullable=False)
    to_date = db.Column(db.Date, nullable=False)
    # relation
    department = db.relationship('Department', back_populates='employees')
    employee = db.relationship('Employee', back_populates='departments')
```

首先我们创建关系表 dept_emp，对应类 `Dept2Emp`。在作为外键的字段 `emp_no` 和 `dept_no` 上用 `db.ForeignKey` 加以标识，指示外键对应的表和字段、以及删除时的操作 `on_delete`。
```py
class Dept2Emp(db.Model):
    __tablename__ = "dept_emp"
    emp_no = db.Column(db.Integer, db.ForeignKey('employees.emp_no', ondelete='cascade'), primary_key=True,
                       nullable=False)
    dept_no = db.Column(db.String(4), db.ForeignKey('departments.dept_no', ondelete='cascade'), primary_key=True,
                        nullable=False, index=True)
```

之后通过 `db_relationship` 创建关系。在 `Dept2Emp` 中我们需要分别建立到 `Department` 和 `Employee` 的联系
```py
    # relation
    department = db.relationship('Department', back_populates='employees')
    employee = db.relationship('Employee', back_populates='departments')
```

同样分别对于 `Department` 和 `Employee`，都需要建立到 `Dept2Emp` 的联系
```py
class Department(db.Model):
    __tablename__ = "departments"
    # relation
    employees = db.relationship('Dept2Emp', back_populates='department')


class Employee(db.Model):
    __tablename__ = "employees"
    # relation
    departments = db.relationship('Dept2Emp', back_populates='employee')
```

唯一需要注意的是对参数 `back_populates` 的设置。该参数指示了当前设定的关系要与对应类的哪个字段发生联系，在具有关系的两个字段间需要同时设置。

### （4）事件设置
有时我们会希望实现类似触发器的功能。SQLAlchemy 提供了事件，用于在 Python 代码而非数据库中实现事件处理。

比如说，如果想要在某一表中插入数据的同时在另一个表中插入对应的数据，只需要这样
```py
@db.event.listens_for(Dept2Manager, 'after_insert')
def insert_into_dept_manager_title_simultaneously(mapper, connection, target):
    new_elm = DeptManagerTitle(target.emp_no, target.from_date, target.to_date)
    db.session.add(new_elm)
    db.session.commit()
```
`@db.event.listens_for(Dept2Manager, 'after_insert')` 设定了监听的事件为对 `Dept2Manager` 的插入操作，此时我们会获取新插入的数据 `target`，并进行一系列后续处理。对本处理函数来说，即为在 `DeptManagerTitle` 中同时插入数据。

### （5）MDL 语句
#### insert
因为 ORM 技术，我们只需要创建新的对象，并将该对象添加到数据库中即可完成插入操作，如
```py
department = Department(dept_no="A001", dept_name="first")
# departments = [department]
db.session.add(department)
# db.session.add_all(departments)
db.commit()
```

注意所有的数据库修改，都只有在调用 `db.commit()` 后才会执行。

> 有可能存在通过字典创建对象的操作，这时只需要调用 `YourClass(**d)` 即可。

#### delete
删除类似，只需要将对象从数据库中删除即可
```py
db.session.delete(target)
db.session.commit()
```

#### update
对于修改，只需要直接修改对象的字段，并调用 `db.session.commit()` 即可。
```py
target.count += 1
db.session.commit()
```

当然还有基于查询的更新，这需要对于查询结果执行 `update` 方法，传入的参数为一个字典，其中包含要修改的字段名和修改后的值。
```py
update_dict = {"count": 10}
Department.query.update(update_dict)
db.session.commit()
```

### query
SQLAlchemy 的查询通过 `filter` 实现 where，通过 `group_by` 实现 group by，通过 `order_by` 实现 order by。对于查询结果，使用 `first`、`all` 等进行输出。
```py
result = Employee.query.filter(Employee.emp_no >= 10100).order_by(Employee.emp_no).first()
```

对于只进行等值查询的情况，还可使用 `filter_by` 替换 `filter`
```py
result = Employee.query.filter(first_name="Tom").all()
```

当然查询部分还有很多内容，不过由于本文只是简介，因此就像其他部分没有讲解的内容一样，就此打住吧。

## 四、Route
### （1）路由简介
路由是路径到函数的映射过程。通过路由，我们就将路径请求转化为了函数调用。Flask 实现了路由机制，能让我们通过简单地编写函数就提供了 api。

```text
/api/v1/xxx/yyy?x1=a&x2=b =====> func(xxx, yyy)
```

### （2）路由的使用
路由的使用十分简单，我们只需要为作为接口的函数增加一个修饰器即可，例如
```py
@app.route('/api/v1/<table_name>', methods=['POST'])
def insert(table_name):
```

我们为 insert 函数增加了修饰器`@app.route`（其中 `app = Flask(__name__)`），将路径 `/api/v1/<table_name>` 映射到该函数，同时设定只响应 POST 类型请求。

注意这里我们使用了路径参数 `table_name`，这需要添加 Flask_RESTful 组件。路径参数使用时在路径中需要用尖括号声明，同时函数中也要声明同名的参数。

同一个函数可以有多个路由，如
```py
@app.route('/api/v1/<table_name>', methods=['GET'])
@app.route('/api/v1/<table_name>/<int:id>', methods=['GET'])
@app.route('/api/v1/<table_name>/<int:id>/<int:id2>', methods=['GET'])
def query(table_name, id=None, id2=None):
```

这里 `<int:id>` 设定 `id` 参数的类型为 `int`，如果不设定的话，默认为字符串。另外我们也可以为参数赋予默认值，以适应路径参数缺省的情况。

在处理完成后，函数可以进行返回。返回的结果有 5 种。分别为：字符串、重定向、html页面、文件或json。对后四种，需要有对应的函数进行处理。

**重定向**
```py
    return redirect('redirect/url')
```

**html**
```py
    return render_template('index.html')
```

**文件**
```py
    return send_file('images/img1.jpg')
```

**json**
```py
    dict_obj = {
      'a': 1,
      'b': "123"
    }
    return jsonfy(dict_obj)
```

## 五、BluePrint
### （1）项目的组织
我们的项目并不只有一个部分，而是由功能不同的模块组成的。不同模块会提供不同的接口，用于不同的服务。这样的话将所有文件堆在一起就很不合适了。如果不及时整理，随着项目的增大，将会造成很大的软件工程灾难。

Flask 提供了蓝图，能够将项目拆封成不同的功能模块，同时在使用时将各个模块结合起来，形成统一的整体。

### （2）蓝图的使用
在之前曾两次出现 `app = Flask(__name__)` 的语句，我们知道 `app` 表示着整个项目。现在蓝图的创建也是类似的，只不过这次只表示着项目的一个部分。
```py
api = Blueprint('api', __name__, url_prefix="/api/v1")
```

这里我们创建了一个蓝图，名为 `api`，该模块所在的路径为 /api/v1，所有处于该模块下的路由都会以 /api/v1 作为前缀。

蓝图的使用也和 `app` 十分类似。现在我们可以把第四节中列出的路由稍微修改一下
```py
@api.route('/<table_name>', methods=['POST'])
def insert(table_name):
    pass

@api.route('/<table_name>', methods=['GET'])
@api.route('/<table_name>/<int:id>', methods=['GET'])
@api.route('/<table_name>/<int:id>/<int:id2>', methods=['GET'])
def query(table_name, id=None, id2=None):
    pass
```

这样这些路由就变为属于 `api` 蓝图的路由了。我们可以将路由和蓝图的声明放在同一个文件中或者同一个软件包中，这样就可以组织起项目的结构了。Flask 并无固定的项目结构，因此可以按照自己的需求进行安排。我自己的项目因为比较简单，安排的很是随意
```sh
.
├── Dockerfile
├── config.ini
├── readme.md
├── src
│   ├── __init__.py
│   ├── api.py
│   ├── database
│   │   ├── __init__.py
│   │   ├── dbs.py
│   │   ├── events.py
│   │   ├── models.py
│   │   └── utils.py
│   ├── main.py
│   └── requirements.txt
└── start.sh
```

最后我们还要将 `api` 模块启用。只需要调用 `app` 的 `register_blueprint` 方法即可
```py
app.register_blueprint(api)
```