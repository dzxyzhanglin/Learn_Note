# Mysql



## 事务

### mysql事务的隔离级别

* `REPEATABLE READ` ：**mysql默认。读取 的数据是事务开始状态时的数据，多次读取数据结果永远不会改变。**

* `READ UNCOMMITTED` ：**可脏读。读取到其他已修改，但是还未提交的数据。**

* `READ COMMITTED` ：**读取已经提交的数据，在同一个事务中，多次读取数据结果可能会改变。**

* `SERIALIZABLE` ：**串行。只能同时执行一个事务。但是不用事务的查询可以执行。**

  

### 查询mysql事务隔离级别

`select @@global.tx_isolation, @@tx_isolation`

> `@@global.tx_isolation`全局隔离级别；
>
> `@@tx_isolation`当前会话（session）隔离级别



### 修改mysql事务隔离级别

`set session transaction isolation level <隔离级别>`