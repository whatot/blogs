# postgres concurrency control

较高的隔离级别能更好地保证数据一致性，但反过来会影响程序的并发性能。
因此，我们往往会根据需要，使用较低的隔离级别完成操作。
这就需要了解各个级别的特性，选择合适的级别，并写出与此级别适应的代码。

* 各种数据库的事务隔离，2016-11-15
http://www.infoq.com/cn/articles/Isolation-Levels
* PG Transaction Isolation
https://www.postgresql.org/docs/current/static/transaction-iso.html
* 数据库事务隔离级别浅析，2015-03-16
http://anjianshi.net/post/yan-jiu-bi-ji/db_isolation

## ACID性质

并非任意的对数据库的操作序列都是数据库事务。
数据库事务拥有以下四个特性，习惯上被称之为ACID特性。

* 原子性（Atomicity）
    事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。
* 一致性（Consistency）
    事务应确保数据库的状态从一个一致状态转变为另一个一致状态。
    一致状态的含义是数据库中的数据应满足完整性约束。
* 隔离性（Isolation）
    多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
* 持久性（Durability）
    已被提交的事务对数据库的修改应该永久保存在数据库中。

## 事务隔离级别 Transaction Isolation

ANSI SQL给出了四种标准的事务隔离级别：

* 可串行化(Serializable)
    此级别下运行的事务是完全隔离的，不会受到其他事务的干扰。
* 可重复读(Repeatable reads)
    此级别中不会出现“不可重复读”的现象，
    也就是其他事务对某行数据的更改无论提交没提交，都不会反应到当前事务中。
    但是允许出现“幻读”，也就是其他事务中插入的新数据，允许出现在当前事务里。
* 提交读(Read committed)
    此级别中，其他事务对数据进行的更改只要一提交，就可以在当前事务中读取到。
    因此会出现“不可重复读”和“幻读”。
* 未提交读(Read uncommitted)
    其他事务对数据进行的任意更改就算没提交，也会立刻反应到当前事务中。
    所有读现象都可能出现在这个级别中。

### 默认事务级别

* MySQL，Repeatable read
* PG，Read committed，（默认没有未提交读Read uncommitted）

## 读现象

这三种读现象是并列、互不相关的，例如在某个隔离等级中出现了“不可重复读”，
并不意味着就一定也能出现“幻读”。

* 幻读（phantom reads）
    一个事务重新执行一个查询，返回一套符合查询条件的行，
    发现这些行因为其它最近提交的事务而发生了改变。(结果集，insert、delete)
* 不可重复读
   一个事务A重新读取前面读取过的数据，发现该数据已经被另一个已提交事务B修改
   （这个事务B的提交是在事务A第一次读之后发生的）。(同一行，update)
* 脏读：一个事务读取了另一个并行的未提交事务写入的数据。(包括insert，update，delete)。

## 标准SQL事务隔离级别对应读现象

| 隔离级别 | 脏读   | 不可重复读 | 幻读   |
| -------- | ------ | ---------- | ------ |
| 读未提交 | 可能   | 可能       | 可能   |
| 读已提交 | 不可能 | 可能       | 可能   |
| 可重复读 | 不可能 | 不可能     | 可能   |
| 可串行化 | 不可能 | 不可能     | 不可能 |

## pg事务的操作命令实例

```
BEGIN;  --开启事务

--事务隔离级别，定义多个事务时间的隔离级别
BEGIN TRANSACTION ISOLATION LEVEl [READ COMMITTED/REPEATABLE READ/SERIALIZABLE];  --一次启动事务并指定事务隔离级别
BEGIN;
TRANSACTION ISOLATION LEVEl [READ COMMITTED/REPEATABLE READ/SERIALIZABLE];  --先启动事务，再设置事务隔离级别


--预备事务，使得事务分阶段可以提交
PREPARE TRANSACTION 'foobar';
......
COMMIT PREPARE TRANSACTION 'foobar';
ROLLBACK PREPARE TRANSACTION 'foobar';


--保存点savepoint,可以支持事务的部分回滚
insert into lyy values(1,'nn');
savepoint svp1;
insert into lyy values(2,'ff');
rollback to savepoint svp1;
--此时提交的话，第二个insert未被插入，但是第一个插入成功。


END; 或者 COMMIT;  --结束并提交事务
或者 ROLLBACK;  --结束并回滚事务
```
