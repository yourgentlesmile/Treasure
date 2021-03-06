# MySQL



## 事务

### ACID属性

#### 原子性(Atomic)

原子性是指事务是一个不可分割的工作单位，事务中的操作要么都发生，要么都不发生。

#### 一致性(Consistent)

事务必须使数据库从一个一致性状态变换到另一个一致性状态

#### 隔离性(Isolation)

事务的隔离性是指一个事务的执行不能被其他事务干扰，即一个事务内部的操作及使用的数据对并发的其他事务是隔离的，并发执行的各个事务之间不能互相干扰。

#### 持久性(Durability)

持久性是指一个事务一旦被提交吗，它对数据库中数据的改变就是永久性的，接下来的其他操作和数据库故障不应该对其有任何影响。

### 分类

#### 隐式事务

事务没有明显的开启和结束标记。  

比如insert、update、delete语句。  

#### 显示事务

> 前提：必须先设置自动提交功能为禁用 set autocommit=0

启动事务：`start transaction;`可写可不写  

结束提交事务：`commit;`  

回滚事务：`rollback;`

### 隔离级别

> 查看事务的隔离级别：`select @@tx_isolation;`
>
> 设置隔离级别(例如read uncommitted): `set session transaction isolation level read uncommitted;`  注意：这里设置的是session级别的

#### 出现的并发问题

##### 脏读

对于两个事务T1,T2。T1读取了已经被T2更新但还**没有被提交**的字段之后，若T2回滚，T1读取的内容就是临时且无效的。

##### 不可重复读

对于两个事务T1，T2。T1读取了一个字段，然后T2**更新**了该字段之后，T1再次读取同一个字段，值就不同了。

##### 幻读

对于两个事务T1，T2。T1从一个表中读取了一个字段，然后T2在该表中**插入**了一些新的行中后，如果T1再次读取同一个表，就会多出几行  

#### 支持的隔离级别

| 隔离级别        | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| READ UNCOMMITED | 允许事务读取未被其他事务提交的变更，**脏读、幻读、不可重复读**的问题都会出现 |
| READ COMMITED   | 只允许事务读取已经被其他事务提交的变更，可以**避免脏读**，但**幻读和不可重复读**问题仍然可能出现 |
| REPEATABLE READ | 确保事务可以多次从一个字段中读取相同的值，在事务持续期间，禁止其他事务对这个字段进行更新，可以避免脏读和不可重复读。但幻读的问题依然存在。 |
| SERIALIZABLE    | 确保事务可以从一个表中读取相同的行，在这个事务持续期间，禁止其他事务对该表执行插入、更新和删除操作，所有并发问题都可以避免，但性能十分地下。 |

### 回滚点

> 执行 `savepoint;`

```sql
SET autocommit=0
START TRANSACTION;
DELETE FROM ACCOUNT WHERE ID = 25;
SAVEPOINT a; #设置保存点
DELETE FROM ACCOUNT WHERE ID = 26;
ROLLBACK TO a; # 回滚到保存点处
```

回滚点之前的SQL的执行动作会被保留，之后的动作会被回滚。  

## SQL_MODE

> 作用是规范SQL语句的书写方式

5.7新增ONLY_FULL_GROUP_BY，规定了如果select中的字段不出现在group by中，则必须以在聚合函数中的形式出现。

例子：  

```sql
mysql> select department_id,employee_name from employee group by department_id;
ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'offcndb.employee.employee_name' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
```

## 字符集

> 主要说UTF8与UTF8mb4  

UTF8：最大存储长度，单个字符最多3个字节  

UTF8mb4：最大存储长度，单个字符最多4个字节  

差别：  

utf8mb4支持的编码比utf8更多，例如：emoji字符UTF8mb4是支持的，而UTF8不支持。因为emoji表情字符，1个字符占4个字节。  

> 建库的时候指定字符集：create database baseee charset utf8mb4;

每个字符集有多种校对规则(排序规则)

**校对规则**：

影响排序操作。

## 约束

Primary key: 主键约束，作用：唯一 + 非空，每张表只能有一个主键，作为**聚簇索引**  

not null  

unique key : 唯一约束

unsigned ： 针对数字列非负数

