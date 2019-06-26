---
layout:     post
title:      "MySQL事务及注意事项"
subtitle:   "MySQL transaction and notes"
date:       2019-06-26 15:30:00
author:     "David"
header-img: "img/mysql.png"
tags:
    - MySQL
---

> “温故而知新”


## 背景

在公司listing诊断项目中，后端对3张不同的MySQL数据表进行插入或更新操作，一旦插入或更新操作出错，3张表均回退到本次操作前状态。这种操作需要引进数据库事务。

*什么是数据库事务？*

数据库事务特性（ACID）：
    - 原子性（Atomicity）：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。
    - 一致性（Consistency）：事务应确保数据库的状态从一个一致状态转变为另一个一致状态。一致状态的含义是数据库中的数据应满足完整性约束。
    - 隔离性（Isolation）：多个事务并发执行时，一个事务的执行不应影响其它事务的执行。
    - 持久性（Durability）：一个事务一旦提交，它对数据库的修改应该永久保存在数据库中。


---
## 代码实现

回到pymysql库的操作上，对数据库事务，采用如下代码实现：

```python
import pymysql
conn = pymysql.connect(host, port, db, user, password)
try:
    cursor = conn.cursor()
    cursor.execute(query1, args1)
    cursor.execute(query2, args2)
    ......
    cursor.close()
    conn.commit()
except Exception as e:
    conn.rollback()
```

如果多个execute操作无异常发生的话，执行commit操作，提交数据库事务；如果发生异常，执行rollback操作，事务回滚至上一条commit执行之后的位置。

接下来说说使用pymysql的一些注意事项。  


## 注意事项


1. 给execute函数传参推荐采用绑定变量（bind variables）的方式，不推荐采用格式化query语句的方式。

因为格式化query语句的方式需要提前对某些特殊字符转义（escape），如下所示：

```python
import pymysql
conn = pymysql.connect(host, port, db, user, password)
cursor = conn.cursor()

text = conn.escape('my name is: \nDavid.')
query = "SELECT * FROM `Codes` WHERE `ShortCode` = {}".format(text)
cursor.execute(query)
```

并且该方式无法采用executemany函数，效率上大打折扣。


2. 采用绑定变量的方式，无需提前转义，在execute函数中自动转义，只需提供query语句和多个变量值组成的tuple作为参数传入execute函数， 如下所示：

```python
...

query = "SELECT * FROM `listing` WHERE `site` = %s and `asin_id` = %s"
args = ('US', '123456')
cursor.execute(query, args)
```

若是需要对同一张表执行多次操作，采用executemany函数：

```python
query = "SELECT * FROM `listing` WHERE `site` = %s and `asin_id` = %s"
args = [('US', '123456'), ('DE', '987654')]
cursor.executemany(query, args)
```

在《Python Cookbook》一书6.8节 Interacting with a Relational Database中提到,

> “You should never use Python string formatting operators (e.g., %) or the .format() method to create such strings.”

**特殊通配符%s**指示数据库后台去采用它自身的字符串替代机制。

综上，采用绑定变量而不是格式化query语句，是更好的实践方式。


3. execute是I/O操作，大批量执行很耗时，可采用gevent库，执行异步操作，提升效率。

execute执行命令应放入gevent.queue中，否则*gevent killed greenlet causes pymysql to continue on the broken connection*，导致异常RuntimeError: reentrant call inside <_io.BufferedReader> exception thrown。


## 参考资料

> [彻底理解数据库事务](https://www.hollischuang.com/archives/898)

> [How can I escape the input to a MySQL db in Python3?](https://stackoverflow.com/questions/11363335/how-can-i-escape-the-input-to-a-mysql-db-in-python3)

> [利用python 实现快速插入300万行数据](https://blog.51cto.com/haowen/2139510)

> [RuntimeError: reentrant call inside <_io.BufferedReader> exception thrown](https://github.com/PyMySQL/PyMySQL/issues/260)

> Python Cookbook, 3th, page 197
