# MySQL笔记

## 架构

### 逻辑架构

<img src="./asset/MySQL_Architecture.png" alt="MySQL_Architecture" style="zoom:80%;" />

MySQL服务器主要分为三层

1. **连接层**
   1. 连接处理
   2. 授权认证
   3. 安全性
2. **核心服务层**
   1. 查询解析
   2. 分析
   3. 优化
   4. 缓存
   5. 内置函数
   6. 跨存储引擎功能
      1. 存储过程
      2. 触发器
      3. 视图
3. **存储引擎层** 详见下述

#### 连接管理与安全性

**每个客户端都有相应的MySQL线程连接 MySQL服务器连接池会根据线程连接认证/验证客户端身份/权限**

#### 优化与执行

##### 执行过程

MySQL核心功能层提供了查询接口、可**解析查询->创建解析树->优化查询(重写/决定表读取顺序)->选择索引->执行语句(与存储引擎交互)**

Note：**SELECT查询解析前服务器先查缓存 如果缓存有不进行解析优化直接执行**

##### 优化器

- **CBO：基于成本优化** 服务器自行优化 不能人为控制 (e.g. **关联表不一定按人为顺序执行 强制人为顺序可用straight_join**)
- RBO：基于规则优化

- 用户可以通过**关键字提示(hint)优化器影响其决策**
- 也可以**让优化器解释(explain)**优化过程
- **存储引擎对优化查询有影响**

### 事务

事务是**一组原子性的SQL语句** 为一个**不可分割的最小工作单元** 

用户可**根据业务判断是否需要事务来选存储引擎** **不需要事务时选非事务型引擎(可以LOCK TABLES提供一定保护)可获得高性能** 

#### ACID

实现了ACID的数据库**CPU处理能力强、内存和磁盘空间需求大**

##### 原子性(Atomicity)

事务中的所有操作要么全部成功 要么全部失败回滚 一个事务**不可能只执行其中一部分操作**

**undo log：保存的是跟执行操作相反的操作（回溯）**

##### 一致性(Consistency)

几个并行执行的事务，其执行结果必须与按某一顺序串行执行的结果相一致 

**事务必须是使数据库从一个一致性状态变到另一个一致性状态**

ACID中**其他三种属性的存在都是为了达成一致性**

##### 隔离性(Isolation)

一个事务所做的**修改在最终提交之前对其他事务不可见**

###### 一致性锁定读

**用户对查询显式加锁**保证数据逻辑一致性

```mysql
# 对读取的行记录加一个X(独占)锁
SELECT...FOR UPDATE;
# 对读取的行记录加一个S(共享)锁
SELECT...LOCK IN SHARE MODE;
```

###### 一致性非锁定读

**通过MVCC在不上锁的情况下读取当前执行时间数据库的行数据** 

读取的行正在执行DELETE/UPDATE时**读取操作不会等待行锁的释放而是读快照数据** 不同事务隔离级别下的快照数据不同(详见MVCC节)

###### 隔离级别

| 事务隔离级别                     | 脏读   | 不可重复读 | 幻读   |
| -------------------------------- | ------ | ---------- | ------ |
| **读未提交（read-uncommitted）** | **是** | **是**     | **是** |
| **不可重复读（read-committed）** | 否     | **是**     | **是** |
| **可重复读（repeatable-read）**  | 否     | 否         | **是** |
| **串行化（serializable）**       | 否     | 否         | 否     |

- **DIRTY READ (脏读)**：**事务可以读取另一事务未提交的脏数据**
- **NONREPEATABLE READ (不可重复读)**：**同一事务执行两次同样的查询 可能会获得不同的结果(读到了其他事务已经提交的更改)**
- **PHANTOM READ (幻读)**
  - **事务在读取某个范围内记录时其他事务又在该范围内并发插入新记录**
  - **此时之前事务再次读取该范围记录时会产生幻行**

1. **READ UNCOMMITED (未提交读)**：**事务中的修改没提交也可以被其他事务读到**
2. **READ COMMITED (提交读/不可重复读)**
   1. **事务开始时只能看到已被提交的修改**
   2. **事务开始直到提交前 所做修改对其他事务不可见**
3. **REPEATABLE READ (可重复读)**
   1. **同一事务中多次读取同样记录结果一致** 
   2. MySQL**默认事务隔离级别** 
   3. **InnoDB通过MVCC+可重复读解决幻读**
4. **SERIALIZABLE (串行化)**：**强制事务串行执行** 读取每一行数据都加锁->大量超时和锁争用

##### 持久性(Durability)

一旦事务**提交** 其所做的**修改就永久保存在数据库中** 即使系统崩溃修改也不会丢失 

**redo log：记录事务对数据的修改 防止事务提交后内存数据丢失**

#### 事务日志

事务日志可以帮助**提高事务效率** 存储引擎可**只修改其内存拷贝 再把修改行为记录到持久硬盘的事务日志中** 不用每次都持久到磁盘

日志采用**追加**方式：**写日志=磁盘块顺序IO**

**预写式日志(WAL)：在数据写入到数据库之前先写入到日志 再将日志记录变更到存储器中 内存中被修改的数据在后台可以慢慢刷回磁盘**

#### 事务一致性

- **强一致性：读操作可以立即读到提交的更新操作**
- **弱一致性：提交的更新操作，不一定立即会被读操作读到** 此种情况会存在一个**不一致窗口，指的是读操作可以读到最新值的一段时间**
- **最终一致性**
  - **弱一致性的特例 事务更新一份数据**
  - **最终一致性保证在没有其他事务更新同样的值的话，最终所有的事务都会读到之前事务更新的最新值**
  - **如果没有错误发生，不一致窗口的大小依赖于：通信延迟，系统负载**等
- 其他一致性变体
  - **单调一致性：如果一个进程已经读到一个值，那么后续不会读到更早的值**
  - **会话一致性：保证客户端和服务器交互的会话过程中，读操作可以读到更新操作后的最新值**

#### MySQL事务

##### 保证失败回滚

- MySQL在开始事务时会**切换到一个延缓操作的状态**(**延缓状态下的操作并不都是立即执行的** 通常情况下语句是立即执行的)
- **commit 时会将延缓执行的操作都执行，并将状态回归到及时写入状态**
- **rollback 时会把延缓写入的操作抛弃掉，此间申请的锁释放掉，并将状态回归到及时写入状态**
  - **释放申请的锁**
  - **回归及时写入状态**
  - **不是放弃未写入的操作（未写入的操作：执行与不执行 rollback 都没有操作写进去 感觉执行或不执行都没什么区别）**
- **显示使用 savepoint，rollback 到之前设置的 savepoint**

##### 自动提交(AUTOCOMMIT)

MySQL**默认采用自动提交**模式 **每个查询都被当作一个事务执行提交操作**

```mysql
SHOW VARIABLES LIKE 'AUTOCOMMIT';
SET AUTOCOMMIT = 0; # 禁用自动提交
```

以下命令**执行之前强制执行COMMIT提交事务**

```mysql
ALTER TABLE ...
LOCK TABLES ...
```

**更改会话隔离级别**

```mysql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**自动提交对非事务性表无效(相当于AUTOCOMMIT一直启用)**

##### 混合使用存储引擎

**服务器层不管理事务** 而是**由存储引擎实现** 同一事务使用不同存储引擎可能会导致事务结果不确定

##### 两阶段锁定协议

**同一事务对任何数据进行读写之前必须对该数据加锁**，在**释放锁之后该事务不再申请和获得任何其他锁 **（**InnoDB**采用）

- **第一阶段（扩展阶段）：获得锁**
- **第二阶段（收缩阶段）：释放锁**

**隐式锁定**：在**事务执行时随时都可以锁定** **所有锁只能在事务提交/回滚时才同时释放** InnoDB会**根据隔离级别自动加锁**

**显示锁定**：可以使用特定语句实现

```mysql
# InnoDB存储引擎层
SELECT ... LOCK IN SHARE MODE
SELECT ... FOR UPDATE

# 服务器层 不能替代事务处理
LOCK TABLES
UNLOCK TABLES
```

Note：**LOCK TABLES最好只在禁用AUTOCOMMIT时显式执行**

### 锁

#### 并发控制

针对服务器层和存储引擎层进行的并发控制

##### 读写锁

- **共享锁(读锁)：共享 互不阻塞  同一时刻多个用户可同时读取同一资源 互不干扰**
- **排他锁(写锁)：非共享 阻塞其他读写锁 同一时刻只有一个用户执行写入**

##### 锁粒度&锁策略

**高并发性：只对会修改的数据进行精准锁定**

**锁粒度**：指锁定数据量大小 **锁粒度越小并发性越高**

**锁策略**：在**锁开销和数据安全性**间寻求**平衡**  

**MySQL存储引擎可实现自己的锁策略和锁粒度**

页锁/表锁速度快、冲突多 & 行锁速度慢、冲突少

###### 页锁 datapage lock

**锁住一个逻辑页(相邻的一组记录)**

###### 表锁 table lock

**最基本且开销最小**的锁策略 加锁时**锁定整张表 **MySQL**服务器层在执行ALTER TABLE语句时使用表锁**（忽略存储引擎锁机制）

- **用户对表进行写操作(增删改)时先获得写锁 阻塞其他用户对该表的读写操作**
- **没有写锁时其他用户采用读操作才能获得读锁(读锁不相互阻塞)**

**READ LOCAL表锁支持并发写**

**写锁比读锁有更高的优先级**(写锁可以插队到读锁前 读锁不可以插队到写锁前)

###### 行级锁 row lock

行级锁可以**最大程度的支持并发**且带来**最大的锁开销** **仅在存储引擎层实现**

**InnoDB**行级锁具体看**索引优化-->高性能索引策略-->索引和锁**

#### 意向锁

**提高封锁子系统的效率 申请意向锁的动作是数据库完成的**

##### 表锁

```mysql
# 用读锁锁表，会阻塞其他事务修改表数据
LOCK TABLE my_table_name READ; 
# 用写锁锁表，会阻塞其他事务读和写
LOCK TABLE my_table_name WRITE;
```

##### 行锁

**共享锁(读锁)：一个事务对一行的共享只读锁**

**排他锁(写锁)：一个事务对一行的排他读写锁**

##### 锁共存

1. **事务A锁住了表中的一行，让这一行只能读，不能写(共享行锁)**
2. **事务B申请整个表的写锁(排他表锁)** 

如果**事务B申请成功**，那么理论上它就能**修改表中的任意一行，这与A持有的行锁是冲突的**

##### 避免锁冲突

数据库要避免冲突--让**B的申请阻塞直到A释放行锁**

1. 判断表是否已被其他事务用表锁锁表
2. 判断**表中的每一行是否已被行锁锁住(逐行遍历整表 判断效率不高-->意向锁诞生)**

##### 意向锁

**事务A必须申请表的意向共享锁 成功后再申请所需行的行锁**

1. **判断表是否已被其他事务用表锁锁表**
2. **发现表上有意向共享锁(表中某(些)行有共享行锁) 事务B申请排他表锁被阻塞**

#### 死锁

**死锁检测&死锁超时机制**

1. **检测到死锁循环以来立刻返回错误 将持有最少行级排他锁的事务回滚** (InnoDB)
2. **查询时间达到锁等待超时后放弃锁请求**

**死锁产生**

- **数据冲突**
- **存储引擎的实现方式不同**

#### InnoDB的MVCC(多版本并发控制)

**MVCC是行级锁的变种** 大多数情况**避免了加锁操作（实现了非阻塞的读 写操作只锁定必要行） 开销更低** **保存数据在某个时间点的快照**

**每行记录后面保存两个隐藏列**（**行的创建时间**和**行的过期/删除时间**）存**系统版本号** **每开始一个新事务 系统版本号自动递增** 

Note：MVCC **很好的实现了事务隔离性** **只在REPEATABLE READ/READ COMMITTED**工作

##### REPEATABLE READ隔离级别下MVCC工作流程

- **SELECT** InnoDB必须检查**每行都满足**以下标准
  - **行版本号<=当前版本号**
  - **行删除版本号未定义或>=当前版本号**
- **INSERT**：**新行创建的存储号设为当前版本号**
- **DELETE**：**删除行的删除版本设为当前版本号**
- **UPDATE**
  - **转换为INSERT+DELETE** 
  - **复制新行并将行版本号设为当前版本号 将旧行的删除版本设为当前版本号****
  - 旧行没有真正被删除 此时innoDB**开启后台线程执行清理工作**(purge：**清掉删除版本号<当前版本号的行**)


##### REPEATABLE READ MVCC vs READ COMMITTED MVCC

1. **V1 V2 V3 按序开启事务** 
2. 当**V3修改记录时V2再读取数据**时**取决于锁定的数据是哪个版本**
   1. **READ COMMITTED读取最新的一致性快照版本** 即读V3更新版本
   2. **REPEATABLE READ读取事务开启前的版本** 即读V2事务开始前的版本(V1)

<img src="./asset/MVCC_compare.jpeg" alt="MVCC_compare" style="zoom:50%;" />

### 存储引擎

**不同的数据文件在物理磁盘上不同的组织形式**

MySQL在文件系统中

- 将每个数据库(schema)保存为数据目录下的一个子目录
- **创建表**时在子目录下创建和表同名的**.frm文件保存表的定义**

**SHOW TABLE STATUS显示表相关信息**

```mysql
SHOW TABLE STATUS LIKE 'user' \G 
*************************** 1. row *************************** 
Name: user 													# 表名
Engine: MyISAM 											# 存储引擎类型
Row_format: Dynamic 								# 行格式 Dynamic--可变行长 Fixed--固定行长 Compressed--行在压缩表
Rows: 6 														# 表的行数
Avg_row_length: 59 
Data_length: 356 
Max_data_length: 4294967295 
Index_length: 2048 									# 索引大小(字节)
Data_free: 0 
Auto_increment: NULL 
Create_time: 2002-01-24 18:07:17 
Update_time: 2002-01-24 21:56:29 
Check_time: NULL 
Collation: utf8_bin 
Checksum: NULL 
Create_options:
	Comment: Users and global privileges 
1 row in set (0.00 sec)
```

#### InnoDB

**MySQL默认事务型引擎 处理大量短期事务**

##### 概览

- **数据存储在表空间(由一系列数据文件(数据+索引)组成)**
- 使用**MVCC支持高并发**
- 实现四个标准隔离级别
- **默认级别：可重复读**
- **防止幻读：间隙锁**(**查询行+索引中的间隙一起锁定**)
- **基于聚簇索引**建立
- **二级(非主键)索引中必须包含主键列**
- 存储格式平台独立

##### 特性

- 可预测性**磁盘预读**
- **自适应哈希索引：自动在内存中创建hash索引加速读操作**
- **插入缓冲区：加速插入操作**
- 支持**热备份**
- **自动崩溃恢复**
- **排序创建索引**
- **删除/增加索引不需复制全表数据**
- 存储格式支持压缩
- 文件格式管理

#### MyISAM

**MySQL5.1之前默认存储引擎**

##### 概览

- 全文索引
- 压缩
- 空间函数
- 不支持事务和行级锁
- 崩溃后无法安全恢复

##### 存储

- **表存储在两个文件中(非聚簇索引)**
  - **数据文件(,MYD)**
  - **索引文件(.MYI)**
- **包含动态/静态行**
- **行记录数**(受限于磁盘)

##### 特性

###### 加锁与并发

**只支持表锁 读取时对读到的所有表加共享锁 写入时加排他锁**

**也支持并发插入：表读取查询时可以插入新记录**

###### 修复

**手动/自动执行检查和修复操作**

```mysql
CHECK TABLE xxxtable; # 检查表错误
REPAIR TABLE xxxtable; # 修复表错误
```

**服务器关闭时用myisamchk命令行工具**

###### 索引特性

- 支持BLOB/TEXT**长字段前缀索引**
- 支持**全文索引** 复杂查询

###### 延迟更新索引键

指定**DELAY_KEY_WRITE**选项

**每次修改后将修改的索引数据写入内存键缓冲区**（不立刻写入磁盘）**清理键缓冲区/关闭表时才将索引块写入磁盘** 极大**提升写入性能**

缺点：数据库/主机崩溃时导致索引损坏

##### MyISAM压缩表

压缩表适用于**不进行修改操作的数据** 极大减少磁盘内存占用/磁盘IO **提升查询性能 支持只读索引**

**表记录独立压缩 读取单行不需要解压表**

##### 性能

**紧密格式**存储数据

**性能问题主要源于表锁**（查询长期处于Locked）

#### InnoDB vs MyISAM

|                    | InnoDB                                                       | MyISAM                                                |
| ------------------ | ------------------------------------------------------------ | ----------------------------------------------------- |
| **适用场景**       | 写多读少<br />可靠性要求高/要求事务<br />表更新频繁、表锁定情况多 | 读多写少<br />不需事务 <br />插入不频繁 查询频繁      |
| **查询速度**       | 慢<br />缓存索引块和数据块<br />寻址先映射块再到行<br />维护MVCC | 快<br />只缓存索引块<br />记录的是文件的偏移量 定位快 |
| **表存储结构**     | .frm表结构文件<br />.ibd文件（存表相关数据、索引、缓存）     | .frm表结构文件<br />.MYD数据文件<br />.MYI索引文件    |
| **数据导出**       | 需导出再导入,sql文件                                         | 只需对应表的存储文件就可运行                          |
| **索引和数据关系** | 索引和数据捆绑 没有压缩 内存占用率大                         | 压缩索引且和数据分开 内存占用率小                     |
| **存储空间**       | 最多64TB                                                     | 无限制 可到64PB                                       |
| InnoDB额外功能     | 支持事务和外键                                               |                                                       |
| **AUTO_INCREMENT** | 有自动提交属性字段必须有索引                                 | 不强制有自动提交属性的字段有索引                      |
| **锁差异**         | 支持行级锁                                                   | 只支持表级锁                                          |
| **全文索引**       | 使用sphinx插件 MySQL5.6后支持                                | 支持                                                  |
| **表行数**         | 不保存表行数                                                 | 保存表行数(SELECT COUNT(*) FROM TABLE)                |
| **CRUD**           | 执行INSERT/UPDATE性能好<br />DELETE时不会重新建立表而是逐行删除 | 执行SELECT性能好                                      |
| **B-Tree索引实现** | B+树                                                         | B树                                                   |

<img src="./asset/innodb_myisam_compare.png" alt="innodb_myisam_compare" style="zoom: 67%;" />



#### 其他内建存储引擎

##### Memory

**可快速访问数据 所有数据保存在内存中 不需要磁盘IO 表重启后数据丢失保留结构**

- 支持**哈希索引** 查找速度极快
- 采用**表锁** 并发写入性能低
- **不支持BLOB/TEXT类型**列 每行长度固定

适合以下场景

- 查找/映射表：**MySQL查询过程需要临时表(即Memory表)保存中间结果** 
- 缓存周期性聚合数据
- **保存中间数据**

Memory表和临时表并不相同：**临时表可使用任何存储引擎 连接断开时临时表会被消除**

##### Archive

**非事务型**引擎 **只支持插入和查询操作 缓存所有写且压缩插入行 比MyISAM磁盘IO更少** 每次查询执行全表扫描 

支持行级锁和专用缓冲区->**高并发插入**

适合**日志/数据采集类应用**

##### Blackhole

没有存储机制 丢弃所有插入数据 由**服务器记录表日志**

适合**特殊复制架构/日志审核**

##### CSV

将CSV文件作为MySQL表处理 **不支持索引**

适合**数据交换**

##### Federated

远程访问其他MySQL服务器的代理

##### Merge

MyISAM变种

分区前适用于日志/数据仓库类应用

##### NDB集群

MySQL集群：MySQL服务器+NDB集群存储引擎+NDB数据库组合(分布式、share-nothing、容灾、高可用)

#### 存储引擎择优

**除非要用InnoDB不具备且没法替代的特性** 否则都**优先使用InnoDB引擎**

##### 考虑因素

- **事务**
  - **需要事务：InnoDB**是最稳定且经过验证的选择
  - **不需要事务且主要是插入/查询** (日志类)：**MyISAM**
- **备份**
  - **忽略备份**因素：**可定期关闭服务器执行备份**
  - **不能忽略备份：InnoDB在线热备份**
- **崩溃恢复**
  - **MyISAM崩溃损坏概率高 恢复速度慢**
  - 即使不需事务也**大多选择InnoDB**
- **特性**
  - **聚簇索引优化**
  - **MyISAM支持地理空间搜索**

##### 日志型应用

- 对插入速度有高要求：**MyISAM/Archive 开销低 插入速度快**
- 对记录日志做分析报表
  - 利用MySQL内置的复制方案复制数据到备库 **备库执行查询 主库执行插入**
  - **日志记录表名中包含年月信息** 区分历史表(查询)/当前表(插入)

##### 只读表

读多写少：**InnoDB 保证崩溃恢复 缓解锁争用**

##### 订单处理

必须**支持事务+外键：InnoDB**

##### 主体讨论区

**支持高读写的存储引擎**

##### CD-ROM应用

使用MySQL数据文件：**MyISAM(压缩)表** 表之间隔离且可相互拷贝

##### 大数据量应用

**建立数据仓库**：Infobright/TokuDB

#### 转换表的引擎

##### ALTER TABLE

**适用于任何存储引擎 但执行时间很长** 数据复制期间**消耗系统所有IO能力 原表加读锁**

**转换过程失去原引擎特性**(InnoDB->MyISAM->InnoDB会导致外键丢失)

```mysql
ALTER TABLE mytable ENGINE = InnoDB;
```

##### 导出+导入

使用**mysqldump工具将数据导出**到文件 然后**更改CREATE TABLE中的引擎选项** 同时**修改表名**

```mysql
CREATE TABLE mytable_new ( 
	...
) ENGINE=InnoDB;
```

##### 创建+查询

**创建新的存储引擎表->INSERT...SELECT语法**

```mysql
CREATE TABLE innodb_table LIKE myisam_table; 
ALTER TABLE innodb_table ENGINE=InnoDB; 
INSERT INTO innodb_table SELECT * FROM myisam_table;
```

**数据量大时分批处理** 

```mysql
# 新表为原表全量复制 原表还在
START TRANSACTION; 
INSERT INTO innodb_table SELECT * FROM myisam_table WHERE id BETWEEN x AND y; 
COMMIT;
```

### 数据完整性&表约束

#### 数据完整性

数据完整性(Data Integrity)是指**数据的精确(Accuracy)和可靠性(Reliability)** 是**对数据的准确性和一致性的一种保证**

1) **实体完整性**：**表的每一行在表中是唯一的实体**

2) **域完整性**：**表中的列必须满足某种特定的数据类型约束(取值范围、 精度等)**

3) **参照完整性：两个表的主关键字和外关键字的数据应一致，保证了表之间的数据的一致性，防止数据丢失或无意义数据扩散**

4) **用户定义的完整性：针对某个特定关系数据库的约束条件 反映某一具体应用必须满足的语义要求**


#### 表约束

**约束确保数据的准确性和一致性**

- **主键约束(PK)primary key constraint：唯一不为空** 
- **唯一约束(UQ)unique constraint：唯一可为空（只能出现一次NULL）**
- **默认约束(DF)default constraint：默认值**
- **检查约束(CK)check constraint：范围以及格式限制**
- **外键约束(FK)foreign key constraint：维护表关系**

### 硬件优化

- CPU：一条SQL语句只能在一个CPU执行
  - 最佳：核心数多+主频高
  - 非密集型查询：主频高
  - 密集型、高并发查询：核心数多
- 内存
  - 大小：和总数据量相近
  - 频率：CPU支持最高频率
- 磁盘
  - 性能比较：PCIe > SSD（随机IO快） > Raid10（提升硬盘性能） > 硬盘 > 网络存储（数据备份）
- 操作系统
  - 合理配置参数
  - 合适文件系统

## Schema与数据类型优化

### 数据类型择优

#### 尽可能小

**尽量使用可正确存储数据的最小数据类型** **可减少磁盘、内存、CPU缓存的占用 CPU处理周期变少** 但要确保**没有低估需要值的范围**

#### 简单

简单数据类型需要**更少CPU周期** 

- **操作代价低：整型>字符**
- **存日期和时间：内建时间类型>字符串**
- **存IP地址：整型>字符串**

#### 避免NULL

**列的默认属性是可为NULL** 

**可为NULL**对MySQL来说**更难优化（索引、索引统计、值比较更复杂）且占用更多存储空间** 且**需要被特殊处理**

可为NULL列**被索引时每个索引记录需要一个额外字节**

InnoDB使用位存储NULL值 **稀疏数据(只有少数行有非NULL值)**有很好的空间效率

#### 整数类型（Whole Numbers）

存储整数类型：TINYINT	SMALLINT	MEDIUMINT	INT		BIGINT（整数计算一般使用BIGINT）

存储空间(位)：		8				16				24				32			64

可存储值范围：−2^(N–1)~2^(N–1)–1	N为位数

**UNSIGNED**属性：可选该属性使**正数上限提高一倍**

MySQL可**为整数类型指定宽度**(例如**INT(11)**) 但不会限制值的合法范围 只是规定显示个数 **存储和计算中INT(1)和INT(20)相同**

#### 实数类型（Real Numbers）

实数是**带有小数部分**的数字 也可以用于**存储比BIGINT还大的整数**

**FLOAT和DOUBLE类型(浮点类型)**支持使用**标准浮点运算**进行近似计算

**DECIMAL类型用于存储精确小数** MySQL5.0后支持精确计算 **可指定精度**：DECIMAL(18,9)小数点两边各存储9个字节数字(X.X X->4 .->1) 计算时DECIMAL会转换为DOUBLE

**浮点类型存储相同范围值比DECIMAL使用更少空间**(FLOAT--4bytes DOUBLE--8bytes)

**数据量大时考虑用BIGINT**代替DECIMAL 将需要单位根据小数位数乘相应倍数 **避免浮点存储计算不准确和DECIMAL精确计算代价高**

#### 字符串类型（String Types）

##### VARCHAR类型

VARCHAR类型**存储可变长字符串** 比定长类型**省空间(仅使用必要空间)**

需要**额外字节记录字符串长度**

- **列最大长度<=255字节--使用1个额外字节**
- 列最大长度>255字节--使用2个额外字节

行变长->UPDATE时可能需要做额外工作->行占用空间变长->**页内没有更多空间存储**

- **MyISAM将行拆成不同字段存储**
- **InnoDB需要分裂页来存储行**

以下情况**适合使用VARCHAR**

- 字符串**列的最大长度比平均长度大很多**
- **列更新少** 不怕碎片化
- 使用**复杂字符集**(如UTF-8)

存储VARCHAR值**不会删除末尾空格**

```mysql
SELECT CONCAT("'", varchar_col, "'") FROM varchar_test; 
+-------------------------------+ 
| CONCAT("'", varchar_col, "'") | 
+-------------------------------+ 
| 'string1' 										| 
| ' string2' 										| 
| 'string3 ' 										| 
+-------------------------------+
```

##### CHAR类型

CHAR类型可**存储定长字符串** 根据定义字符串长度分配足够空间

以下情况**适合使用CHAR**

- **存储很短的列**
- **列所有值接近同一长度**

存储CHAR值会**删除所有末尾空格** **填充/截取空格是由MySQL服务器进行处理的**

```mysql
SELECT CONCAT("'", char_col, "'") FROM char_test; 
+----------------------------+ 
| CONCAT("'", char_col, "'") | 
+----------------------------+ 
| 'string1' 								 | 
| ' string2' 								 | 
| 'string3' 								 | 
+----------------------------+
```

##### BINARY类型

BINARY类型存储的是**二进制字符串(存储的是字节码** 不是字符) **填充BINARY采用零字节**而不是空格 检索时也不会去掉填充

以下情况**适合使用BINARY**

- 需要**存储二进制数据**
- 使用**字节码（二进制）比较**
- **大小写敏感**

##### VARBINARY类型

**存储可变长二进制字符串** 其余特性和CHAR/VARCHAR对比关系的BINARY一致

##### BLOB类型

BLOB类型存储**很大的二进制字符串数据** **没有排序规则/字符集**

**InnoDB将过长VARCHAR存为BLOB**

##### TEXT类型

TEXT类型存储**很大的字符方式字符串数据** **有排序规则/字符集**

##### BLOB & TEXT

MySQL把BLOB和TEXT值都当作**独立对象**处理  **只对每列的最前max_sort_length字节做排序** 

```mysql
# 只需排序前面一部分字符
减小max_sort_length配置
ORDER BY SUBSTRING(col, length)
```

**值太大时 InnoDB在行内存储指针指向在外部存储区域的实际值**

**尽量避免使用BLOB/TEXT**：查询不能使用Memory引擎(不支持BLOB/TEXT)的隐式临时表 需转为MyISAM磁盘临时表造成严重性能开销

**实在不能避免BLOB/TEXT**：所有用到的地方都使用**SUBSTRING(col, length)转为字符串**使用内存临时表(**Extra：Using temporary**)

##### 枚举类型（ENUM）

可以使用枚举列代替常用的字符串类型 把**不重复的字符串存储为预定义集合** 添加/删除枚举元素时必须使用ALTER TABLE

MySQL存储枚举时将每个值在**列表中的位置保存为整数**

```mysql
CREATE TABLE enum_test(
  e ENUM('fish', 'apple', 'dog') NOT NULL
);
INSERT INTO enum_test(e) VALUES('fish'), ('dog'), ('apple');
```

```mysql
SELECT e + 0 FROM enum_test; 
+-------+ 
| e + 0 | 
+-------+ 
| 		1 | 
| 		3 | 
| 		2 | 
+-------+
```

枚举字段**按照内部存储的整数排序**而**不是定义的字符串**排序

```mysql
SELECT e FROM enum_test ORDER BY e; 
+-------+ 
| e 		| 
+-------+ 
| fish 	| 
| apple | 
| dog 	| 
+-------+
```

绕过该限制

- **按照需要的顺序定义枚举列**

- 查询时使用**FIELD()函数显式指定**排序顺序 但**无法利用索引消除排序**

  ```mysql
  SELECT e FROM enum_test ORDER BY FIELD(e, 'apple', 'dog', 'fish');
  ```

##### VARCHAR & ENUM

```mysql
CREATE TABLE webservicecalls ( 
  day date NOT NULL, 
  account smallint NOT NULL, 
  service varchar(10) NOT NULL, 
  method varchar(50) NOT NULL, 
  ... omitted ... 
	PRIMARY KEY (day, account, service, method) 
) ENGINE=InnoDB;
```

把service和method**换成ENUM 可以让表缩小**

```mysql
CREATE TABLE webservicecalls_enum (
  ... omitted ...
  service ENUM(...values omitted...) NOT NULL, 
  method ENUM(...values omitted...) NOT NULL,
 	... omitted ... 
) ENGINE=InnoDB;
```

```mysql
# VARCHAR和VARCHAR关联起来
SELECT SQL_NO_CACHE COUNT(*) 
FROM webservicecalls
JOIN webservicecalls 
USING(day, account, service, method);
# VARCHAR和ENUM关联起来
SELECT SQL_NO_CACHE COUNT(*) 
FROM webservicecalls
JOIN webservicecalls_enum
USING(day, account, service, method);
```

| Test                      | Queries per second |
| ------------------------- | ------------------ |
| VARCHAR joined to VARCHAR | 2.6                |
| VARCHAR joined to ENUM    | 1.7                |
| ENUM joined to VARCHAR    | 1.8                |
| **ENUM joined to ENUM**   | **3.5**            |

#### 日期和时间类型（Date&Time Types）

| DATETIME类型                           | TIMESTAMP类型                          |
| -------------------------------------- | -------------------------------------- |
| 保存**1001年～9999年**精度为秒的范围值 | 保存**1970年～2038年**精度为秒的范围值 |
| 存储空间**8字节**                      | 存储空间**4字节** **空间效率高**       |
| **与时区无关**                         | **与时区有关**                         |
|                                        | 列默认为**NOT NULL**                   |

除特殊行为外**尽量使用TIMESTAMP**

如需**存储比秒更小粒度的时间：BIGINT存储微秒级别时间戳、DOUBLE存储秒之后小数部分**、使用MariaDB代替MySQL

#### 位数据类型（Bit-Packed Data Types）

位数据技术上说都是**字符串类型**

##### BIT类型

**存储一个/多个true/false值** **MyISAM打包存储所有BIT列** **InnoDB/Memory为每个BIT列使用足够的最小整数类型**存放(不能省空间)

MySQL将BIT当作字符串类型 BIT(x)结果是包含二进制值的字符串

**1bit存储1个true/false：创建为空的CHAR(0)列(**保存NULL/空字符串)

##### SET类型

**存储很多true/false值时合并列到SET类型(打包的位集合)** 有效利用存储空间但**无法使用索引且改变列定义的代价高(需要ALTER TABLE)**

##### ACL例子

ACL：保存权限的访问控制列表

**使用SET**

```mysql
CREATE TABLE acl (
  perms SET('CAN_READ', 'CAN_WRITE', 'CAN_DELETE') NOT NULL
);

INSERT INTO acl(perms) VALUES ('CAN_READ,CAN_DELETE'); 
SELECT perms FROM acl WHERE FIND_IN_SET('AN_READ', perms); 
+---------------------+ 
| perms 							| 
+---------------------+ 
| CAN_READ,CAN_DELETE | 
+---------------------+
```

**使用整数变量**

```mysql
SET @CAN_READ := 1 << 0, @CAN_WRITE := 1 << 1,@CAN_DELETE := 1 << 2; 
CREATE TABLE acl (
  perms TINYINT UNSIGNED NOT NULL DEFAULT 0
);
INSERT INTO acl(perms) VALUES(@CAN_READ + @CAN_DELETE);
SELECT perms FROM acl WHERE perms & @CAN_READ; 
+-------+ 
| perms | 
+-------+ 
| 		5 | 
+-------+
```

#### 选择标识符（Choosing Identifiers）

| 整数类型             | ENUM和SET类型      | 字符串类型                     |
| -------------------- | ------------------ | ------------------------------ |
| 查询快               |                    | 查询慢(MyISAM压缩索引)         |
| **标识列最好的选择** | 尽量避免作为标识列 | 尽量避免作为标识列             |
|                      | 存储固定状态       | 消耗空间                       |
|                      |                    | 随机字符串任意分布在很大空间中 |
|                      |                    |                                |

##### 随机新值导致INSERT/SELECT变慢

**插入值随机写到索引不同位置->INSERT变慢->页分裂、磁盘随机访问、聚簇索引碎片**

**逻辑相邻行分布在磁盘不同地方->SELECT变慢**

**访问局部性原理失效->缓存效果差**

#### 特殊类型数据

**IP地址**：不要拿字符串类型存 **用无符号整数存（INET_ATON()&INET_NTOA()）** 

### schema设计陷阱

#### 过多的列

**存储引擎API工作流程**

1. **服务器层<--行缓冲拷贝数据-->存储引擎层**
2. **服务器层将缓冲内容解码：编码列>>行数据结构**

解码代价：**MyISAM变长行、InnoDB行结构需要转换 列越多转换代价越高**

#### 过多的关联

**单个查询最好在12个表以内做关联** 查询执行快且并发性好

#### 全能枚举

**避免过度使用枚举**

```mysql
CREATE TABLE ... country enum('','0','1','2',...,'31')
```

#### 变相枚举

**ENUM允许列中存储一组定义值的单个值 SET允许存储一组定义值的单/多个值**

避免使用SET做真假判断列(列值不能同时出现)

```mysql
CREATE TABLE ... is_default set('Y','N') NOT NULL default 'N'
```

#### NULL

尽可能避免使用NULL(**MySQL索引中存储NULL值**)：**存储事实空值**时可以用**0/特殊值/空字符串**代替

**表示未知值时不要害怕使用NULL**：DATETIME伪造全0值会导致导致问题

### 范式/反范式

- **码（候选码） ：为某表中的一个属性或属性组 除码之外的所有属性都完全函数依赖于码** 
  - **码确定的情况下该表除码之外的所有属性的值也随之确定**
  -  **一张表中可以有超过一个码**
- **部分函数依赖：依赖于码的一部分**
- **非主属性：包含在任何一个码中的属性成为主属性**

1. **第一范式 1NF**：**每个属性都不可再分**，1NF 是所有关系型数据库的基本
2. **第二范式 2NF**：在 1NF 的基础之上，**消除了非主属性对于码的部分函数依赖**
3. **第三范式 3NF**：在 2NF 的基础之上，**消除了非主属性对于码的传递函数依赖**
4. **BCNF 范式**：在 3NF 的基础上，**消除主属性对于码的部分与传递函数依赖**
5. **反范式：增加冗余数据或数据分组**来提高数据库读性能

#### (反)范式优缺点

| 范式                                         | 反范式                                                   |
| -------------------------------------------- | -------------------------------------------------------- |
| **更新操作快**                               | 更新操作慢                                               |
| **只有很少/没有重复数据(修改数据少)**        | 信息冗余                                                 |
| **表通常更小** 占用内存空间小 **执行操作快** | 列多表大 占用内存空间大 执行操作无索引时慢               |
| **查询时更少需要DISTINCT/GROUP BY**          | **查询时最差情况全表扫描(数据大时比关联快--避免随机IO)** |
| 常需关联多个表 关联代价高                    | **避免关联** 数据都在一张表里                            |
| 可能使**索引无效(同索引列分布在不同表)**     | 可用更**有效的索引策略**                                 |

#### 混用范式化和反范式化

**反范式化数据用于复制/缓存 不同表中存储相同的特定列(提升性能)** 

### 缓存表/汇总表

#### 缓存表

**存储较易且慢的从schema获取数据的表** **对优化搜索/检索查询语句有效**

##### 例：不同索引加速查询 

**创建只包含部分列的缓存表且对缓存表使用不同存储引擎**

- 主表使用InnoDB
- **缓存表使用MyISAM 压缩索引且可全文搜素**

#### 汇总表/累积表

**保存GROUP BY聚合数据的表**

##### 例：获得过去24小时消息发送数量

方案一（**不严格计数**）：每小时生成一张汇总表 但计数器并不精确

方案二（**小范围查询填满间隙的严格计数**）：以每小时汇总表为基础 把**之前23小时统计表加起来再加开始/结束阶段不完整小时内的计数** 

```mysql
CREATE TABLE msg_per_hr ( 
  hr DATETIME NOT NULL, 
  cnt INT UNSIGNED NOT NULL, 
  PRIMARY KEY(hr) 
);

# LEFT(NOW(), 14)获得当前日期和时间最接近的小时
SELECT SUM(cnt) FROM msg_per_hr 
WHERE hr BETWEEN
CONCAT(LEFT(NOW(), 14), '00:00') - INTERVAL 23 HOUR
AND CONCAT(LEFT(NOW(), 14), '00:00') - INTERVAL 1 HOUR; 

SELECT COUNT(*) FROM message
WHERE posted >= NOW() - INTERVAL 24 HOUR
AND posted < CONCAT(LEFT(NOW(), 14), '00:00') - INTERVAL 23 HOUR; 

SELECT COUNT(*) FROM message
WHERE posted >= CONCAT(LEFT(NOW(), 14), '00:00');
```

**建立汇总表主要原因**：两个方案都**比计算所有行效率高**

#### 影子表

**影子表保证了重建汇总表/缓存表时数据在操作时依然可用** 

```mysql
DROP TABLE IF EXISTS my_summary_new, my_summary_old; 
CREATE TABLE my_summary_new LIKE my_summary; 
-- populate my_summary_new as desired 
# 可通过原子重命名操作切换影子表和原表 下一次重建之前保留旧版本数据(my_summary_old)
RENAME TABLE my_summary TO my_summary_old, my_summary_new TO my_summary;
```

#### 物化视图

**预先计算且存在磁盘上的表**

#### 计数器表

**使用独立计数器表(小、快)可避免查询缓存失效**

##### 简单计数器表

记录上有互斥锁**事务只能串行执行**

```mysql
CREATE TABLE hit_counter (
  cnt int unsigned not null
) ENGINE=InnoDB;

UPDATE hit_counter SET cnt = cnt + 1;
```

##### 并发更新计数器表

将**计数器保存在多行中 每次随机选择一行更新** 提高事务并发更新性能

```mysql
CREATE TABLE hit_counter (
  slot tinyint unsigned not null primary key,
  cnt int unsigned not null
) ENGINE=InnoDB;

# 预先生成行数据 选择随机槽更新
UPDATE hit_counter SET cnt = cnt + 1 WHERE slot = RAND() * 100;

# 统计结果
SELECT SUM(cnt) FROM hit_counter;
```

##### 时间间隔计数器表

每隔一段时间开始一个新计数器

```mysql
CREATE TABLE daily_hit_counter (
  day date not null,
  slot tinyint unsigned not null,
  cnt int unsigned not null,
  primary key(day, slot)
) ENGINE=InnoDB;

# 不用预先生成行数据 用ON DUPLICATE KEY UPDATE代替
INSERT INTO daily_hit_counter(day, slot, cnt)
VALUES(CURRENT_DATE, RAND() * 100, 1)
ON DUPLICATE KEY UPDATE cnt = cnt + 1;

# 周期执行任务 合并所有结果到0号槽 删除其他槽 减少表行数
UPDATE daily_hit_counter as c
INNER JOIN (
  SELECT day, SUM(cnt) AS cnt, MIN(slot) AS mslot
  FROM daily_hit_counter
  GROUP BY day
) AS x USING(day)
SET c.cnt = IF(c.slot = x.mslot, x.cnt, 0), c.slot = IF(c.slot = x.mslot, 0, c.slot);
DELETE FROM daily_hit_counter WHERE slot <> 0 AND cnt = 0;
```

#### 快读慢写

高性能数据库常用技巧：**额外索引、冗余列、缓存表、汇总表**-->**提升读查询**--> **写操作变慢 增加读写操作开发难度**

### ALTER TABLE优化

由于**大部分ALTER TABLE**都会导**致MySQL服务中断** 需要对其进行优化

ALTER TABLE有**可能不需要重建表**：**移除(不是增加)一个列的AUTO_INCREMENT属性、增删改ENUM/SET常量、删改列默认值**

```mysql
# MODIFY COLUMN操作会导致表重建
ALTER TABLE sakila.film MODIFY COLUMN rental_duration TINYINT(3) NOT NULL DEFAULT 5;

# ALTER COLUMN操作不会会导致表重建 (直接修改.frm文件不涉及表数据)
ALTER TABLE sakila.film ALTER COLUMN rental_duration SET DEFAULT 5;
```

#### 常见场景

1. 先在**不提供服务的机器上执行ALTER TABLE** 然后**和提供服务的主库进行切换**
2. **影子拷贝**：按原表**创建表结构一样的新表** 然后**重命名+删表**来交换两张表

#### 特殊场景

##### 只修改.frm文件

为**想要的表结构创建新.frm文件替换掉旧.frm文件**

1. **创建一张相同结构空表 进行所需的修改**
2. 执行**FLUSH TABLES WITH READ LOCK**(关闭所有正使用的表且禁止任何表被打开)
3. **交换.frm文件**
4. 执行**UNLOCK TABLES释放步骤2的读锁**

```mysql
SHOW COLUMNS FROM sakila.film LIKE 'rating'; 
+--------+------------------------------------+------+-----+---------+-------+ 
| Field  | Type 															| Null | Key | Default | Extra | 
+--------+------------------------------------+------+-----+---------+-------+ 
| rating | enum('G','PG','PG-13','R','NC-17') | YES  |		 | G 			 | 			 | 
+--------+------------------------------------+------+-----+---------+-------+

CREATE TABLE sakila.film_new LIKE sakila.film; 
ALTER TABLE sakila.film_new
MODIFY COLUMN rating ENUM('G','PG','PG-13','R','NC-17', 'PG-14') DEFAULT 'G';
FLUSH TABLES WITH READ LOCK;

/var/lib/mysql/sakila# mv film.frm film_tmp.frm 
/var/lib/mysql/sakila# mv film_new.frm film.frm 
/var/lib/mysql/sakila# mv film_tmp.frm film_new.frm

UNLOCK TABLES; mysql> SHOW COLUMNS FROM sakila.film LIKE 'rating'\G 
*************************** 1. row *************************** 
Field: rating 
	Type: enum('G','PG','PG-13','R','NC-17','PG-14')

DROP TABLE sakila.film_new;
```

##### 快速创建MyISAM索引

###### MyISAM表高效载入数据

**禁用索引->载入数据->重新启用索引**

**构建索引工作被延迟到数据完全载入后 可以用排序构建索引 索引树碎片少且紧凑**

**缺点：对唯一索引无效**

```mysql
ALTER TABLE test.load_data DISABLE KEYS; 
-- load the data 
ALTER TABLE test.load_data ENABLE KEYS;
```

###### 唯一索引解决方法

- **依赖于InnoDB快速在线创建索引功能：删除所有非唯一索引->加新列->重新创建删除掉的索引**
- hack
  - **用需要的表结构创建空表 不带索引**
  - **载入数据到表中(创建.MYD文件)**
  - **用需要的表结构创建另一张空表 带索引(创建.frm和.MYI文件)**
  - **获取读锁 刷新表**
  - **重命名第二张表.frm和.MYI文件到第一张表**
  - **释放读锁**
  - **使用REPAIR TABLE重建表索引(排序构建所有索引)**

## 索引优化

#### 索引基础

**索引(键)是存储引擎用于快速找到记录的一种数据结构** 存储引擎先在排好序的索引中找到对应值 根据匹配索引记录找到对应的数据行

##### 局部性原理

数据/程序都有**聚集成群的倾向**

**空间局部性：被查询过的数据很可能会再次被查询**

**时间局部性：被查询过的数据短期内可能会再次被查询**

##### 磁盘预读

**内存和磁盘在进行交互时要保证每次读取都是一个逻辑单位（页 ）而不是单个数据行** 

存储引擎进行读取时一般取4kb整数倍(InnoDB取16kb)

##### 数据结构选择

**InnoDB和MyISAM**两大存储引擎都使用**B-Tree索引作为主要索引结构** 而**不是哈希结构/二叉树/AVL树/红黑树**

<img src="./asset/algo.png" alt="algo" style="zoom:100%;" />

###### 哈希表

哈希算法根据键和哈希函数算出哈希值 **哈希结构效率高 但哈希表只能精确查找值 无法执行范围查询** 数据**存在磁盘时需精确查找 IO量大**

且**可能有哈希碰撞** 

<img src="./asset/hashtable.png" alt="hashtable" style="zoom: 50%;" />

###### 二叉搜索树

二叉搜索树作为索引结构时**磁盘IO次数和树高度相关** **极端情况树可能会成链状**

<img src="./asset/binary_tree.png" alt="binary_tree" style="zoom: 33%;" />

###### AVL树

解决上述二叉搜索树极端链状情况 可以使用AVL树 **AVL树高度平衡**(左子树和右子树高度相差<=1) **查找效率较高**

但AVL树**增删效率低** 多个节点时可能需要**旋转多次**

<img src="./asset/avl_tree.png" alt="avl_tree" style="zoom: 25%;" />

###### 红黑树

**2-3树**：内部节点可以**有2个子节点和1个数据元素(2节点)**、或有**3个子节点和2个数据元素(3节点)**，**叶子节点有1至2个数据元素**

**红黑树等价于2-3树** **2节点等价于普通AVL节点** **3节点本质上是非平衡性的缓存 ** **增删效率高 用空间换时间**

**需要再平衡/增删改时 2节点和3节点间的转化会吸收不平衡性 减少旋转次数 数据随机且增删相当时3节点非平衡性缓存效果越明显**

<img src="./asset/red_black_tree.png" alt="red_black_tree" style="zoom: 50%;" />

红黑树比AVL综合深度低 但**数据量大时深度依然会很高 导致IO变多**

**红黑树特性**

- **节点颜色非黑即红**
- **根节点和所有叶子节点(NIL空节点)是黑色**
- **红色节点的两个子节点都是黑色**
- **任意节点到每个叶子节点的最短路径都包含相同数目的黑色节点**
- **增删改操作时间复杂度O(logn)** 
  - **非严格平衡 插入效率更高 最多旋转三次** 
  - **删除节点情况**
    - **红色节点：直接删除**
    - **只有一个子节点的节点：子节点改为黑色**
    - **黑色节点：复杂调整**
- **左右子树高度最多差一倍**

###### B-Tree

B树**保证了更少的磁盘访问（IO少）高度低** B树的**节点可以看作磁盘块**(包括**指向下面磁盘块的指针**)

**阶：节点的子节点数的最大值**

一个 **M 阶的 B 树（M>2） 且k 的取值范围为 [ceil(M/2), M]** 有以下的特性

1. **根节点包含子节点个数的范围[2,M]**

2. **中间节点包含 k-1 个关键字和 k 个子节点** 

3. **叶子节点包括 k-1 个关键字(无子节点)**

4. 假设中间节点节点的关键字为：Key[1], Key[2], …, Key[k-1]，且**关键字按照升序排序**，即 **Key[i] <Key[i+1]**

5. **所有叶子节点位于同一层**

**B树相比二叉树结构高度更低、磁盘IO更少、查询性能更好**

<img src="./asset/b_tree.png" alt="b_tree" style="zoom: 43%;" />

###### B+Tree

**B+树为B树的变种 仅在叶子节点存储信息（所有关键字）各层节点的关键字均是下一层对应节点中的最大(小)关键字的复写**

**B+Tree vs B-Tree**

1. **B+树有 k 个孩子的节点就有 k 个关键字。也就是子节点数量 = 关键字数**，而 **B 树中，子节点数量 = 关键字数 +1**
2. **非叶子节点的关键字也会同时存在在子节点中，并且是在子节点中所有关键字的最大（或最小）**
3. **非叶子节点仅用于索引，不保存数据记录，跟记录有关的信息都放在叶子节点中**。而 **B 树中， 非叶子节点既保存索引，也保存数据记录** 

4. 所有**关键字都在叶子节点出现**，**叶子节点间构成一个有序链表**，而且**叶子节点本身按照关键字的大 小从小到大顺序链接**

![B+_Tree](./asset/B+_Tree.webp)

##### 索引类型

**索引在存储引擎层实现而非服务器层** 

没有统一的索引标准 不同存储引擎索引工作方式不同 支持索引类型也不同 同一索引不同引擎底层实现可能不同

|                | InnoDB                   | MyISAM   | Memory     | NDB    |
| -------------- | ------------------------ | -------- | ---------- | ------ |
| **B-Tree索引** | **支持**                 | **支持** | **支持**   | 不支持 |
| **哈希索引**   | **支持(自适应哈希索引)** | 不支持   | **支持**   | 支持   |
| **全文索引**   | **支持**                 | **支持** | **不支持** | 不支持 |
| **空间索引**   | 不支持                   | **支持** | 不支持     | 不支持 |

###### B-Tree索引

Note：**实际上大多存储引擎使用的是B+Tree 每个叶子节点都包含指向下一叶子节点的指针方便叶子节点范围遍历**

**存储引擎以不同方式使用B-Tree索引** **B-Tree的所有值都是按序存储且每个叶子页到根的距离相同**

- **MyISAM使用前缀压缩索引 通过数据物理位置引用被索引的行**
- **InnoDB按原数据格式存储 通过主键引用被索引的行**

B-Tree索引**避免了全表扫描加快访问数据速度** 且对**索引列是顺序组织存储的(对定义索引列顺序排序)很适合查找范围数据**

- **存储引擎从索引根节点(槽中存放指向子节点指针)开始搜索**
- **根据指针向下层查找 比较节点页的值和查找的值**
- **找到合适指针进入下层子节点 子节点存储值的上下限** 
- **叶子节点中的指针指向被索引的数据 不是其他节点页**
- **最终找到对应值或记录不存在**

使用B-Tree索引的**查询类型**

- **全值匹配**：匹配所有索引列
- **匹配最左前缀**：从索引第一列开始匹配
- **匹配列前缀**：可以只匹配列前缀(满足最左匹配)
- **匹配范围值**：可以范围查询(满足最左匹配)
- **精确匹配顺序列并范围匹配另一列**：最左列精确匹配 其右面的列可以范围匹配
- **只访问索引的查询**：覆盖索引
- **按序查找**：索引扫描做排序 ORDER BY

**B-Tree索引失效**

- **不符合最左匹配原则**
- **按序跳过索引列**
- **范围查询列右面的所有列都索引失效**
- **计算、函数、类型转换(自动或手动)** （详见高性能索引策略--独立的列）
- 表达式中**使用不等于符号（!=或<>）或通配符%（LIKE前面不能用：左模糊/全模糊）**
- 表达式中使用**IS NOT NULL** （IS NULL可以使用索引）
- **OR前后存在非索引列**
- **字符集转换**

###### 哈希索引

哈希索引**基于哈希表实现** 只能在**精准匹配索引所有列的查询**生效

**存储引擎对索引列的每一行数据计算一个哈希码 哈希索引将所有哈希码存储在索引中并在哈希表中保存每一个数据行的指针**

只有**Memory引擎显式支持哈希索引且将其作为默认索引类型** 同时也支持B-Tree索引、**非唯一哈希索引(哈希碰撞时采用链地址法)**

```mysql
# 哈希索引示例
CREATE TABLE testhash (
  fname VARCHAR(50) NOT NULL, 
  lname VARCHAR(50) NOT NULL,
  KEY USING HASH(fname) 
) ENGINE=MEMORY;

SELECT * FROM testhash; 
+--------+-----------+ 
| fname  | lname 		 | 
+--------+-----------+ 
| Arjen | Lentz 		 | 
| Baron | Schwartz	 | 
| Peter | Zaitsev		 | 
| Vadim | Tkachenko	 | 
+--------+-----------+

# 哈希函数f()
f('Arjen')= 2323 
f('Baron')= 7437 
f('Peter')= 8784 
f('Vadim')= 2458
```

| Slot | Value            |
| ---- | ---------------- |
| 2323 | Pointer to row 1 |
| 2458 | Pointer to row 4 |
| 7437 | Pointer to row 2 |
| 8784 | Pointer to row 3 |

```mysql
# 先计算'Peter'的哈希值->使用该值找到对应记录指针
f('Peter')= 8784->索引查找8784->找到第三行指针->比较第三行值是否为'Peter'
SELECT lname FROM testhash WHERE fname='Peter';
```

- 哈希索引**优点**
  - 索引本身**只存对应哈希值** 结构紧凑
  - **查找速度非常快**
- 哈希索引**缺点**
  - **不存字段值** 不能使用索引中的值避免读取行 但**访问内存的行速度很快** 性能影响不明显
  - **无法用于排序 不按照索引值顺序存储**
  - **不支持部分索引列匹配查找**(不能使用最左前缀)
  - **只支持等值比较(=、IN()、<=>)查询 不支持任何范围查询**
  - **哈希冲突必须遍历链表所有行指针逐行比较**
  - **哈希冲突很多时索引维护代价高**

**自定义哈希索引**

**InnoDB特有功能 用户无法控制/设置 当某些索引值被使用的非常频繁时会在内存中基于B-Tree索引之上再创建一个哈希索引**

**创建自定义哈希索引**

**在查询的WHERE手动指定使用哈希函数**：B-Tree基础上创建一个伪哈希索引->还是使B-Tree查找 但使用哈希值不是键本身进行索引查找

示例：**存储大量URL并根据URL搜索查找**

```mysql
# 新增url_crc索引列(选择性高 体积小) 用CRC32做哈希
# 性能高：优化器使用url_crc索引列完成查找
# 缺陷：需要手动维护/触发器维护哈希值
SELECT id FROM url 
WHERE url="http://www.mysql.com" 
AND url_crc=CRC32("http://www.mysql.com");

CREATE TABLE pseudohash ( 
  id int unsigned NOT NULL auto_increment, 
  url varchar(255) NOT NULL, 
  url_crc int unsigned NOT NULL DEFAULT 0, 
  PRIMARY KEY(id) 
);

# 插入/更新时触发器维护哈希值
DELIMITER //

CREATE TRIGGER pseudohash_crc_ins BEFORE INSERT ON pseudohash FOR EACH ROW BEGIN 
SET NEW.url_crc=crc32(NEW.url); 
END; //

CREATE TRIGGER pseudohash_crc_upd BEFORE UPDATE ON pseudohash FOR EACH ROW BEGIN 
SET NEW.url_crc=crc32(NEW.url); 
END; //

DELIMITER ;

# 验证触发器
INSERT INTO pseudohash (url) VALUES ('http://www.mysql.com'); 
SELECT * FROM pseudohash; 
+----+----------------------+------------+ 
| id | url | url_crc | 
+----+----------------------+------------+ 
| 1 | http://www.mysql.com | 1560514994 | 
+----+----------------------+------------+ 

UPDATE pseudohash SET url='http://www.mysql.com/' WHERE id=1; 
SELECT * FROM pseudohash; 
+----+---------------------- +------------+ 
| id | url | url_crc | 
+----+---------------------- +------------+ 
| 1 | http://www.mysql.com/ | 1558250469 | 
+----+---------------------- +------------+

# 不要用SHA()/MD5()强加密函数(哈希值非常长)作为哈希函数

# 数据表非常大时CRC32()出现大量哈希冲突：可以实现简单的64位哈希函数(返回整数)
# 简便方法：MD5函数返回值的一部分作为自定义哈希函数
SELECT CONV(RIGHT(MD5('http://www.mysql.com/'), 16), 16, 10) AS HASH64; 
+---------------------+ 
| HASH64 							| 
+---------------------+ 
| 9761173720318281581 | 
+---------------------+

# 哈希冲突
SELECT word, crc FROM words WHERE crc = CRC32('gnu'); 
+---------+------------+ 
| word 		| crc				 | 
+---------+------------+ 
| codding | 1774765869 | 
| gnu 		| 1774765869 | 
+---------+------------+

# 处理哈希冲突：在WHERE条件中带入哈希值和对应列值
SELECT word, crc FROM words WHERE crc = CRC32('gnu') AND word = 'gnu'; 
+------+------------+ 
| word | crc 				| 
+------+------------+ 
| gnu  | 1774765869 | 
+------+------------+
```

###### R-Tree(空间)索引

MyISAM表支持空间索引 可以用作**地理数据存储** 

**空间索引从所有维度索引数据 无需前缀查询 可以从任意维度组合查询数据(必须使用GIS函数维护数据 但MySQL的GIS并不完善)**

###### 全文索引

**特殊类型索引 查找文本中的关键词而不是直接比较中的值** 类似搜索引擎 不是简单WHERE条件匹配 适用于MATCH AGAINST操作

###### MySQL索引分类

- **功能逻辑：普通索引、唯一索引、主键索引、全文索引、R-Tree索引**
- **存储方式：聚簇索引、非聚簇索引**
- **字段数量：单列索引、联合索引**
- **数据结构：B-Tree索引、哈希索引、R-Tree索引、全文索引**

##### 索引优点

1. **极大的减少了服务器需要扫描的数据量**
2. **帮助服务器避免排序和建临时表**
3. **随机IO变为顺序IO**

###### 三星系统

1. 一星：**索引将相关记录放到一起**
2. 二星：**索引中数据顺序和查找中的排列顺序一致**
3. 三星：**索引中的列包含查询所需的所有列**

###### 查询记录择优

只有当索引**帮助存储引擎快速找到记录带来的好处大于其额外工作**时索引才是最有效的

1. **非常小的表：全表扫描**
2. **中到大型表：索引**
3. **特大型的表：分区技术**(创建/使用索引代价随数据量增长)
4. **表的数量极多：元数据信息表**

#### 高性能索引策略

##### 独立的列

**索引列不能是表达式的一部分/函数的参数/出现类型类型转换**

1. 不能在计算表达式中使用索引列

   ```mysql
   # 可用索引
   EXPLAIN SELECT SQL_NO_CACHE id, stuno, NAME FROM student WHERE stuno = 900000 + 1;
   # 索引失效
   EXPLAIN SELECT SQL_NO_CACHE id, stuno, NAME FROM student WHERE stuno + 1 = 900001;
   ```

2. 不能将索引列作为函数的参数

   ```mysql
   # 可用索引
   EXPLAIN SELECT SQL_NO_CACHE * FROM student WHERE student.name LIKE 'abc%';
   # 索引失效
   EXPLAIN SELECT SQL_NO_CACHE * FROM student WHERE LEFT(student.name,3) = 'abc';
   ```

3. 不能将索引列类型转换

   ```mysql
   # 可用索引
   EXPLAIN SELECT SQL_NO_CACHE * FROM student WHERE name = '123';
   # 索引失效
   EXPLAIN SELECT SQL_NO_CACHE * FROM student WHERE name = 123;
   ```

##### 前缀索引

###### **基数**

**列中唯一值的数量**

```mysql
# 查看基数
SELECT COUNT(DISTINCT column_name) FROM table_name;
```

###### **选择性**

**列基数/表总行数** 

- **选择性越高的索引可过滤掉更多的行 查询效率越高**
- 唯一索引选择性为1(最好的索引选择性) 性能最好
- **复合索引中一般把选择性最高的列排最前面** 尽可能缩小匹配行初始候选值
- 帮助创建**前缀索引**

```mysql
SELECT (COUNT(DISTINCT column_name) / COUNT(*)) from table_name;
```

###### **前缀索引**

前缀索引是一种**能使索引更小更快**的索引 但**MySQL无法对前缀索引做分组(GROUP BY)或排序(ORDER BY)或覆盖扫描**

**选择前缀：足够长的前缀保证高选择性 （列前缀的基数要接近于完整列的基数） 同时不能过长以节约空间**

**选择前缀示例**

1. 根据前缀长度测试选择性

```mysql
# 找最常见城市
SELECT COUNT(*) AS cnt, city 
FROM sakila.city_demo 
GROUP BY city 
ORDER BY cnt DESC 
LIMIT 10; 
+-----+----------------+ 
| cnt | city | 
+-----+----------------+ 
| 65 | London | 
| 49 | Hiroshima | 
| 48 | Teboksary | 
| 48 | Pak Kret |
...
+-----+----------------+

# 找最频繁出现的城市前缀 从3个开始
SELECT COUNT(*) AS cnt, LEFT(city, 3) AS pref 
FROM sakila.city_demo 
GROUP BY pref 
ORDER BY cnt DESC 
LIMIT 10; 
+-----+------+ 
| cnt | pref | 
+-----+------+ 
| 483 | San | 
| 195 | Cha | 
| 177 | Tan | 
| 167 | Sou | 
...
+-----+------+

# 增加前缀长度 直到前缀选择性接近完整列的选择性 示例中前缀长度为7
SELECT COUNT(*) AS cnt, LEFT(city, 7) AS pref
FROM sakila.city_demo 
GROUP BY pref 
ORDER BY cnt DESC 
LIMIT 10; 
+-----+---------+ 
| cnt | pref | 
+-----+---------+ 
| 70 | Santiag | 
| 68 | San Fel | 
| 65 | London | 
| 61 | Valle d | 
...
+-----+---------+
```

2. **让前缀选择性接近计算出的完整列的选择性**

```mysql
# 计算完整列选择性
SELECT COUNT(DISTINCT city)/COUNT(*) FROM sakila.city_demo;
+-------------------------------+ 
| COUNT(DISTINCT city)/COUNT(*) | 
+-------------------------------+ 
| 											 0.0312 | 
+-------------------------------+

# 计算各前缀长度选择性
SELECT 
COUNT(DISTINCT LEFT(city, 3))/COUNT(*) AS sel3,
COUNT(DISTINCT LEFT(city, 4))/COUNT(*) AS sel4,
COUNT(DISTINCT LEFT(city, 5))/COUNT(*) AS sel5,
COUNT(DISTINCT LEFT(city, 6))/COUNT(*) AS sel6,
COUNT(DISTINCT LEFT(city, 7))/COUNT(*) AS sel7
FROM sakila.city_demo;

+--------+--------+--------+--------+--------+ 
| sel3 	 | sel4	  | sel5	 | sel6	  | sel7 	 | 
+--------+--------+--------+--------+--------+ 
| 0.0239 | 0.0293 | 0.0305 | 0.0309 | 0.0310 | 
+--------+--------+--------+--------+--------+
```

**创建前缀索引**

```mysql
ALTER TABLE sakila.city_demo ADD KEY (city(7));
```

##### 后缀索引

**MySQL原生不支持反向索引 但可把列值反转存储并建立前缀索引 通过触发器维护这种后缀索引**

##### 索引合并

**每个列上建立独立索引会导致索引效率降低**

MySQL5.0后引入了**索引合并**策略使查询可以**同时使用多个单列索引扫描行并将结果合并**

索引合并算法有三个变种  **EXPLAIN结果中的type=index_merge时Extra列会显示对应算法**

1. 并集union

   ```mysql
   EXPLAIN SELECT film_id, actor_id FROM sakila.film_actor
   WHERE actor_id = 1 OR film_id = 1
   ...
   type: index_merge
   key: PRIMARY,idx_fk_film_id
   ...
   Extra: Using union(PRIMARY,idx_fk_film_id); Using where
   ```

2. 交集intersection

   ```mysql
   EXPLAIN SELECT film_id, actor_id FROM sakila.film_actor
   WHERE actor_id = 1 AND film_id = 1
   ...
   type: index_merge
   key: PRIMARY,idx_fk_film_id
   ...
   Extra: Using intersect(PRIMARY,idx_fk_film_id); Using where
   ```

3. 有序并集sort_union  **在WHERE子句中指定OR组合的多个范围条件 且必须在返回行数据前先获取行ID并对行ID进行排序**

   ```mysql
   EXPLAIN SELECT film_id, actor_id FROM sakila.film_actor
   WHERE actor_id > 1 OR film_id <= 1
   ...
   type: index_merge
   key: PRIMARY,idx_fk_film_id
   ...
   Extra: Using sort_union(PRIMARY,idx_fk_film_id); Using where
   ```

**索引合并本身是优化结果** 但实际是**说明索引建立的不够好** 

- 对**多个索引做相交操作**时意味着需要**一个包含所有相关列的组合索引而不是多个独立的单列索引**
- 对**多个索引做联合操作**时意味着服务器需要**消耗大量CPU/内存资源在算法缓存、排序、合并**操作上
- **优化器不计入索引合并出现的查询成本** 容易使得成本被低估 影响查询并发性

出现索引合并时需要**检查表/查询结构**或通过参数**optimizer_switch关闭索引合并**或使用**优化器IGNORE INDEX提示**

##### 组合索引

**组合索引只创建一棵索引树** **树的叶子节点(InnoDB)存放所有列的值**

<img src="./asset/composite_index.png" alt="composite_index" style="zoom:20%;" />

###### **不考虑排序和分组**

**只优化WHERE条件查找：需要考虑全局基数和选择性**

1. **性能依赖于所有索引列的选择性：一般将选择性最高的列放在最前面**
2. **性能依赖于查询条件具体值分布**

```mysql
# 示例查询 考虑建(staff_id, customer_id)还是(customer_id, staff_id)组合索引
SELECT * FROM payment WHERE staff_id = 2 AND customer_id = 584;

# 执行sarg 可搜索的参数(searchable argument) 查看各个WHERE条件分支对应的条件值数
SELECT SUM(staff_id = 2), SUM(customer_id = 584) FROM payment;
*************************** 1. row *************************** 
SUM(staff_id = 2): 7992 
SUM(customer_id = 584): 30 # 条件值数量更小 应放前面

# 具体查询：对于customer_id = 584这个条件值 查看对应的staff_id = 2选择性 
SELECT SUM(staff_id = 2) FROM payment WHERE customer_id = 584;
*************************** 1. row *************************** 
SUM(staff_id = 2): 17 # 查询结果非常依赖具体值 可能造成整体性能下降

# 查看两个列的选择性
SELECT COUNT(DISTINCT staff_id)/COUNT(*) AS staff_id_selectivity,
COUNT(DISTINCT customer_id)/COUNT(*) AS customer_id_selectivity,
COUNT(*)
FROM payment
*************************** 1. row *************************** 
staff_id_selectivity: 0.0001 
customer_id_selectivity: 0.0373 # 选择性更高
COUNT(*): 16049

# 组合索引把customer_id放前面
ALTER TABLE payment ADD KEY(customer_id, staff_id);
```

###### 特殊用户

特殊条件值：**基数比正常值高** 如未登录用户、大量好友用户 查询时易出现**服务器性能问题**

```mysql
SELECT COUNT(DISTINCT threadId) AS COUNT_VALUE
FROM Message
WHERE (groupId = 10137) AND (userId = 1288826) AND (anonymous = 0)
ORDER BY priority DESC, modifiedDate DESC;

# EXPLAIN 结果
id: 1 
select_type: SIMPLE 
table: Message 
type: ref 
key: ix_groupId_userId # 优化器使用了(group_Id, userId)组合索引
key_len: 18 
ref: const,const 
rows: 1251162 
Extra: Using where

# sarg 查看各条件匹配行
SELECT COUNT(*), SUM(groupId = 10137), SUM(userId = 1288826), SUM(anonymous = 0) FROM Message

*************************** 1. row *************************** 
# 组合索引基本没用
count(*): 4142217 
sum(groupId = 10137): 4092654 # 几乎满足表中所有行
sum(userId = 1288826): 1288496 # 也很多
sum(anonymous = 0): 4141934

# 解决方案：修改应用程序 区分特殊用户 禁止针对特殊用户执行查询
```

##### 聚簇索引

(非)聚簇索引**不是单独的索引类型** 而是**数据存储方式**  聚簇即**数据行和相邻键值紧凑的存储在一起**

- 表中有聚簇索引时索引的**叶子页中实际存放了表数据行的全部数据** 节点页只包含索引列

- **一个表中只能有一个聚簇索引**(**实际数据页只能按照一棵B+树进行排序**)

- 使用**聚簇主键可能导致性能提升也可能导致性能下降**

- **聚簇索引优点**

  - **可以把相关数据存储在一起**
  - **减少磁盘IO** 最大限度**提高IO密集型应用性能**
  - **数据访问更快**
  - **可使用覆盖索引扫描查询直接获取主键**

- **聚簇索引缺点**

  - 如果**数据都放在内存中访问顺序不重要**时**没有优势**

  - **插入速度严重依赖于插入顺序**

    - 按主键顺序插入最快
    - 不按照主键顺序插入 建议用**OPTIMIZE TABLE**命令重新组织表

  - **更新聚簇索引列代价很高**

  - **不按主键顺序插入新行/主键更新导致移动行**可能出现**页分裂**

    - **行的主键值要求必须将这行插入到已满的页中时**
    - **存储引擎会将该页分裂成两个页面来容纳该行**
    - 页分裂本身易**导致表占用更多磁盘空间**

  - 聚簇索引存储的**行比较稀疏/页分裂导致数据存储不连续**时可能导致**全表扫描变慢** 

  - 使用聚簇索引会导致**非聚簇(二级)索引的叶子节点包含引用行的主键列**

  - **二级索引访问时需要两次索引查找**

    - **二级索引叶子节点保存的不是指向行的物理位置指针而是行的主键值**

    - **回表：存储引擎先找到二级索引的叶子节点获得对应主键值->根据主键值去聚簇索引找到对应行**

    - 二级索引中如果出现**行更新导致的行碎片化/移动行在原位置保存prev指针时会出现查询性能更差**的情况

      ```mysql
      # 表结构
      create table T(
        id int primary key, # 主键(聚簇索引)
        k int not null, 
        name varchar(16),
        index (k) # 非主键 二级索引
      ) engine=InnoDB;
      
      # 插入数据(id, k): R1(100,1)、R2(200,2)、R3(300,3)、R4(500,5)、R5(600,6)
      ```

      <img src="./asset/innodb_index.png" alt="innodb_index" style="zoom: 67%;" />

      如图：**左为聚簇索引(叶子节点存整个数据行) 右为二级索引(叶子节点存主键值)**

      ```mysql
      select * from T where k between 3 and 5;
      
      有以下的执行流程
      1、在 k 索引树上找到 k = 3,取得ID = 300
      2、再到 ID 索引树查到 ID=300对应的R3
      3、在 k 索引树取下一个值 k=5，取得ID=500
      4、再回到 ID 索引树查到 ID=500对应的 R4
      5、在 k 索引树取下一个值 k=6，不满足条件，查询结束
      
      从二级索引回到主键索引树查找的过程，称为回表
      上面流程回表了两次，分别是步骤2和步骤4
      ```

##### InnDB索引 vs MyISAM索引

###### **InnoDB索引**

**InnoDB的聚簇索引在同一个结构中保存了B-Tree索引和数据行** **通过主键聚集数据**

###### **InnoDB主键选择**

1. **主键被定义：定义主键作聚簇索引**
2. **主键没有被定义：用唯一的非空索引代替**
3. **没有主键/唯一非空索引：隐式定义主键(6字节自增列)作聚簇索引**

###### **InnoDB表中按主键顺序插入行**

**代理键：一种主键形式 数据与应用无关** 一般使用AUTO_INCREMENT**自增列（保证数据行按顺序写入）**

**尽量避免随机(不连续且值分布范围大)的主键（聚簇索引）特别是IO密集型应用**

**代理键和随机主键对比**

- **随机主键**
  - **写入目标页可能已经刷到磁盘并从缓存删除/没被加载到缓存中** **插入前必须从磁盘查取目标页到内存** 导致大量**随机IO**
  - **写入乱序 频繁页分裂为新行分配空间、移动大量数据、一次插入最少修改三个页**
  - 频繁页分裂导致**页空间稀疏且被不规则填充**最终**数据碎片化**
- **代理键**
  - 主键值是**顺序且单调递增**的 到达页最大填充因子时下一条记录写到新页 **主键页被顺序记录填满**
  - **高并发不适用**
    - **并发插入导致间隙锁竞争**
    - AUTO_INCREMENT锁机制

###### **MyISAM索引**

**MyISAM采用非聚簇索引** 不通过主键聚集数据 **主键索引(名为PRIMARY的唯一非空索引)和其他索引结构无差别**

###### **InnoDB & MyISAM 数据分布对比**

- **MyISAM数据分布**

  - **按照数据插入的顺序存储在磁盘**中 由操作系统缓存
  - **行号从0开始递增且行定长**：MyISAM可以**跳过所需字节找到需要的行**
  - **易创建索引** 且**索引存在内存中方便查询**

- **InnoDB数据分布**

  - InnoDB的**聚簇索引和非聚簇索引存储数据方式不同**
  - **聚簇索引**
    - **每个叶子节点都包括主键列、事务ID、回滚指针(事务/MVCC)、剩余的其他列**
    - <img src="./asset/innodb_index2.png" alt="innodb_index2" style="zoom: 77%;" />
  - **非聚簇(二级)索引**
    - **每个叶子节点都存储的是索引列和主键列**
    - **主键列当行指针**会**占用更多空间但也保证了InnoDB在移动行的物理位置时不用更新该指针**
    - <img src="./asset/innodb_index3.png" alt="innodb_index3" style="zoom:78%;" />

- InnoDB&MyISAM**数据分布对比**

  <img src="./asset/innodb_myisam_index.png" alt="innodb_myisam_index" style="zoom:20%;" />

##### 覆盖索引

覆盖索引：**索引覆盖了所有需要查询字段 MySQL可以直接通过索引获取列数据而无需读取数据行**

- 覆盖索引不需回表 极大提高性能

- 索引条目远小于数据行大小 只读取索引可减少数据访问量

- 索引在单页内按照列值顺序存储 范围查询可使用完全顺序的索引访问 性能提高

- 覆盖索引如果**在InnoDB表中二级主键可以覆盖查询就能避免回表**

- **索引覆盖查询：EXPLAIN中有Extra：Using index**

  - 索引可以覆盖

    ```mysql
    EXPLAIN SELECT store_id, film_id FROM sakila.inventory
    *************************** 1. row *************************** 
    id: 1
    select_type: SIMPLE 
    table: inventory 
    type: index 
    possible_keys: NULL 
    key: idx_store_id_film_id 
    key_len: 3 
    ref: NULL 
    rows: 4673 
    Extra: Using index
    ```

  - **索引无法覆盖**

    - **没有任何索引可以覆盖所有列(SELECT *)** 
    - **LIKE%(通配符在后)可以做最左前缀匹配**
    - **%LIKE(通配符在前)不可以做最左前缀匹配 只能提取行值而不是索引值**

    ```mysql
    EXPLAIN SELECT * FROM products WHERE actor='SEAN CARREY' AND title like '%APOLLO%'
    *************************** 1. row *************************** 
    id: 1 
    select_type: SIMPLE 
    table: products 
    type: ref 
    possible_keys: ACTOR,IX_PROD_ACTOR 
    key: ACTOR 
    key_len: 52 
    ref: const 
    rows: 10 
    Extra: Using where
    ```

  - 解决以上覆盖索引问题

    - 扩展索引(artist, title, prod_id)
    - **延迟关联(查询第一阶段可使用覆盖索引->在FROM子句的子查询找到匹配prod_id->根据id值在外层查询匹配获取所需列值)**

    ```mysql
    EXPLAIN SELECT *
    FROM products
    	JOIN (
        SELECT prod_id
        FROM products
        WHERE actor='SEAN CARREY' AND title LIKE '%APOLLO%'
      ) AS t1 ON (t1.prod_id=products.prod_id)
    
    *************************** 1. row *************************** 
    id: 1
    select_type: PRIMARY
    table: <derived2> 
    ...omitted...
    *************************** 2. row *************************** 
    id: 1
    select_type: PRIMARY
    table: products ...omitted...
    *************************** 3. row *************************** 
    id: 2
    select_type: DERIVED
    table: products
    type: ref 
    possible_keys: ACTOR,ACTOR_2,IX_PROD_ACTOR 
    key: ACTOR_2
    key_len: 52 
    ref:
    rows: 11
    Extra: Using where; Using index
    ```

##### 索引扫描做排序

MySQL用两种方式**生成有序结果**

- **排序操作**：**Extra列中有Using filesort**
- **索引顺序扫描**：**type列为index**
  - 索引不能覆盖查询全部列时会导致随机IO
  - 当**索引列顺序和ORDER BY顺序完全一致且所有列排序方向一致**才能使用索引排序
  - **查询关联多张表**时只有**ORDER BY子句引用字段全部在一个表**时才能使用索引排序

###### 特殊情况

**ORDER BY在前导列为常量时可以不满足索引最左前缀 （WHERE/JOIN对列指定常量）**

```mysql
# 组合索引 (rental_date, inventory_id, customer_id):

CREATE TABLE rental (
  ...
  PRIMARY KEY (rental_id),
  UNIQUE KEY rental_date (rental_date,inventory_id,customer_id), 
  KEY idx_fk_inventory_id (inventory_id), 
  KEY idx_fk_customer_id (customer_id), 
  KEY idx_fk_staff_id (staff_id), 
  ...
);

# ORDER BY可以用于查询排序且不需要满足最左前缀匹配 
# 索引的第一列(rental_date)被指定为常数
EXPLAIN SELECT rental_id, staff_id FROM sakila.rental
WHERE rental_date = '2005-05-25' 
ORDER BY inventory_id, customer_id;

*************************** 1. row *************************** 
type: ref 
possible_keys: rental_date 
key: rental_date
rows: 1
Extra: Using where 	# 没有 Using filesort


# 可以索引排序：索引第一列为常量且ORDER BY第二列(组合形成最左前缀)
... WHERE rental_date = '2005-05-25' ORDER BY inventory_id DESC;

# 可以索引排序：索引第一列为范围查询但ORDER BY第一、二列(最左前缀)
... WHERE rental_date > '2005-05-25' ORDER BY rental_date, inventory_id;

# 不可以索引排序：索引列排序方向不一致
... WHERE rental_date = '2005-05-25' ORDER BY inventory_id DESC, customer_id ASC;

# 不可以索引排序：ORDER BY不在索引的列
... WHERE rental_date = '2005-05-25' ORDER BY inventory_id, staff_id;

# 不可以索引排序：索引第一列常量+ORDER BY第三列无法形成最左前缀
... WHERE rental_date = '2005-05-25' ORDER BY customer_id;

# 不可以索引排序：索引第一列为范围查询 无法使用索引其余列
... WHERE rental_date > '2005-05-25' ORDER BY inventory_id, customer_id;

# 不可以索引排序：索引第二列使用IN(多个等于条件) 对于排序来说也是范围查询
... WHERE rental_date = '2005-05-25' AND inventory_id IN(1,2) ORDER BY customer_ id;

# 理论上可以使用索引排序 但优化器将film_actor当成关联的第二个表 不能使用索引
EXPLAIN SELECT actor_id, title FROM sakila.film_actor 
INNER JOIN sakila.film USING(film_id) ORDER BY actor_id
+------------+----------------------------------------------+ 
| table 		 | Extra 																				| 
+------------+----------------------------------------------+ 
| film 			 | Using index; Using temporary; Using filesort | 
| film_actor | Using index 																	| 
+------------+----------------------------------------------+
```

##### 压缩索引

**MyISAM使用前缀压缩来减少索引大小** 可提高性能 默认只压缩字符串 可在CREATE TABLE中指定PACK_KEYS控制索引压缩

- **压缩索引块**流程：**完全保存索引块第一个值**->**其他值和第一个值进行比较**得到并保存**相同前缀的字节数和剩余的不同后缀** 
- **压缩块空间占用少 但操作慢(压缩计算依赖第一个值)** 
- **MyISAM查找时无法使用二分 只能从头扫描索引块 倒序扫描时性能差 平均查找某行需要扫描半个索引块**
- **CPU密集（随机查找扫描）：索引查找慢** 倒序扫描更慢
- **IO密集：查询性能高**

##### 冗余&重复索引

###### 重复索引

**相同列上按照相同顺序创建的相同类型索引**

```mysql
CREATE TABLE test ( 
  ID INT NOT NULL PRIMARY KEY, # 主键索引
  A INT NOT NULL, 
  B INT NOT NULL, 
  UNIQUE(ID), 								 # 唯一索引
  INDEX(ID) 									 # 普通索引
) ENGINE=InnoDB;
```

###### 冗余索引

**创建索引(A,B)再创建索引(A)（索引(A)算是索引(A,B)的前缀索引）**

**冗余索引(A, 主键)：主键列已经包含在A列的二级索引中了**

###### 优化冗余&重复索引

- **优化冗余索引**
  - **删除冗余索引**
  - **尽量扩展已有索引而不是创建新索引**
  - **如果特殊情况考虑性能可以使用冗余索引(扩展已有索引会致其过大)**
- **优化重复索引**
  - **删除重复索引**
- **找出冗余&重复索引**
  - 查询INFORMATION_SCHEMA(数据量大时可能导致性能下降)
  - common_schema视图定位
  - Percona Toolkit's pt-duplicate-key-checker 

##### 未使用索引

**考虑删除服务器永远不同的索引（除了用于唯一约束避免产生重复数据的索引）**

##### 索引和锁

- MySQL5.1后：InnoDB可以在**服务端过滤掉行后释放锁**

- MySQL5.1前：**事务提交才能释放锁** 

  ```mysql
  SET AUTOCOMMIT=0;
  BEGIN; 
  SELECT actor_id FROM sakila.actor WHERE actor_id < 5 AND actor_id <> 1 FOR UPDATE;
  # 查询仅返回2-4行
  +----------+ 
  | actor_id |
  +----------+ 
  | 			 2 | 
  | 			 3 | 
  | 			 4 | 
  +----------+
  
  # 实际获取了1-4行排他锁 
  # type:range(索引范围扫描) 
  # Extra:Using where代表服务器将存储引擎返回行后再应用WHERE过滤条件
  EXPLAIN SELECT actor_id FROM sakila.actor WHERE actor_id < 5 AND actor_id <> 1 FOR UPDATE; 
  +----+-------------+-------+-------+---------+--------------------------+ 
  | id | select_type | table | type  | key 		 | Extra 										|
  +----+-------------+-------+-------+---------+--------------------------+ 
  | 1 | SIMPLE 			 | actor | range | PRIMARY | Using where; Using index | 
  +----+-------------+-------+-------+---------+--------------------------+
  ```

- **索引可以让查询锁定更少的行 可以减少行级锁的数量**
- **InnDB在二级索引上使用共享锁 访问主键索引需要排他锁** 
  - 如果**索引无法过滤所有无效列 会导致服务器Using where无法使用覆盖索引(不是所有情况)**
  - **SELECT FOR UPDATE会比LOCK IN SHARE MODE/非锁定查询要慢很多**

###### Innodb三种行锁

1. **行锁**（Record Lock）：**锁直接加在索引记录上面而不是行数据，锁住的是key**
2. **间隙锁**（Gap Lock）：**锁定索引记录间隙，确保索引记录的间隙不变** 针对事务隔离级别为**可重复读或以上**
3. **Next-Key Lock**： **行锁和间隙锁组合**

###### Innodb默认使用Next-Key Lock

默认情况（可重复读隔离级别）**加锁时会以Next-Key Lock的方式对数据进行加锁 防止幻读**

###### Innodb Next-Key Lock加锁规则

**扫描索引记录->先对索引记录加上行锁-> 对索引记录两边的间隙加上间隙锁->其他事务不能在这个间隙修改或者插入记录**

优化1： 索引上的等值查询，给**唯一索引加锁**时next-key lock**退化为行锁**

优化2：索引上的等值查询，**向右遍历时且最后一个值不满足等值条件时**next-key lock**退化为间隙锁**

##### 索引下推

**索引下推 Index Condition Pushdown(ICP)是MySQL 5.6中的新特性** 将**数据筛选放在磁盘中操作而不是在内存中 减少IO**

索引下推**帮助存储引擎层使用索引过滤数据** **减少存储引擎访问基表的次数以及MySQL服务器访问存储引擎的次数**

###### 没有索引下推

<img src="./asset/ICP1.png" alt="ICP1" style="zoom:30%;" />

<img src="./asset/ICP2.png" alt="ICP2" style="zoom:30%;" />

###### 有索引下推

<img src="./asset/ICP3.png" alt="ICP3" style="zoom:30%;" />

<img src="./asset/ICP4.png" alt="ICP4" style="zoom:30%;" />

###### 使用限制

- **只能用于二级索引**
- **WHERE的条件字段不在索引列时不做索引下推**
- **覆盖索引不能索引下推**

###### 示例

```mysql
SELECT * FROM tuser WHERE NAME LIKE '张%' AND age = 10 AND ismale = 1;
```

没有索引下推 逐行查找 返回时再筛选

<img src="./asset/ICP5.png" alt="ICP5" style="zoom:33%;" />

有索引下推 直接利用索引条件查找

<img src="./asset/ICP6.png" alt="ICP6" style="zoom:33%;" />

##### MRR(Multi-Range Read Optimization)

优化器将**随机IO转化为顺序IO**以降低查询过程中IO开销 **二级索引回表时在查询主键时会把主键排序 按序访问数据行（顺序IO）**

##### FIC(Fast Index Creation)

**FIC可使创建和删除二级索引时不需要拷贝整个表的数据(表加共享锁)**

#### 维护索引和表

##### 找到并修复损坏的表

**MyISAM中的表损坏**通常是**系统崩溃**导致的 

**其他引擎**也会因硬**件问题、MySQL本身缺陷、操作系统问题**出现**索引损坏（可能导致查询返回错误结果/主键冲突/数据库崩溃）**

- 运行**CHECK TABLE**可以**找出大多数表/索引错误**
- 执行**REPAIR TABLE修复损坏表**
- 不支持REPAIR TABLE可通过**no-op的ALTER操作重建表 ** 例如将**当前InnoDB表用ALTER TABLE中ENGINE属性定义为InnoDB**
- **使用myisamchk工具**
- **数据导出再导入**
- 如果损坏的是**系统区域/表的行数据区域**而不是索引
  - **备份中恢复表**
  - 从损坏的数据文件中**尽可能恢复数据**

**InnoDB由于设计并不会轻易出现表损坏** 如果出现损坏可能由于以下原因

- 数据库硬件问题
- 管理员错误操作
- InnoDB本身缺陷(不太可能)

InnoDB**数据损坏**时可定义**innodb_force_recovery参数进入InnoDB强制恢复模式修复数据**或Percona InnoDB Data Recovery Toolkit修复

##### 维护准确的索引统计信息

**查询优化器**通过两个API知道**存储引擎索引值的分布信息**

- **records_in_range()**：向存储引擎传入两个边界值**获取范围记录 MyISAM返回精确值 InnoDB返回估算值**
- **Info()：返回各种类型数据包括索引基数**

**表没有统计信息/统计信息不准确**时可执行**ANALYSE TABLE重新生成统计信息**

- **Memory引擎不存储索引统计信息**
- **MyISAM将索引统计信息存在磁盘** ANALYSE TABLE需要**全索引扫描 锁表**
- **InnoDB**在MySQL5.5后通过**随机索引访问评估将其存储在内存**

###### **查看索引基数**

1. **SHOW INDEX FROM**

```mysql
SHOW INDEX FROM sakila.actor
*************************** 1. row *************************** 
Table: actor 
Non_unique: 0 
Key_name: PRIMARY 
Seq_in_index: 1 
Column_name: actor_id 
Collation: A 
Cardinality: 200 
Sub_part: NULL 
Packed: NULL 
Null:
Index_type: BTREE 
Comment:

*************************** 2. row *************************** 
Table: actor 
Non_unique: 1 
Key_name: idx_actor_last_name 
Seq_in_index: 1 
Column_name: last_name 
Collation: A 
Cardinality: 200 S
ub_part: NULL 
Packed: NULL 
Null:
Index_type: BTREE 
Comment:
```

2. MySQL5.0后可从**INFORMATION_SCHEMA.STATISTICS**表查到

###### InnoDB统计信息

**抽样计算**：随机读取少量索引页面 以此为样本计算索引的统计信息

**计算索引统计信息时机**

- **表首次打开**
- **执行ANALYZE TABLE**
- **表大小发生非常大的变化**(大小变化超过1/16或新插入20亿行)

**更新索引统计信息时机**

- **打开INFORMATION_SCHEMA.STATISTICS**
- **SHOW TABLE STATUS**
- **SHOW INDEX**
- MySQL**客户端开启自动补全**功能

##### 减少索引和数据的碎片

**B-Tree需要随机磁盘访问才能定义到叶子页 随机访问不可避免** 

###### 数据碎片

**数据碎片会导致随机访问次数变多** 查询性能变差

- **行碎片：一个数据行被存储到多个地方的多个片段中**
- **行间碎片：逻辑上顺序的页、行在磁盘上不是顺序存储的 严重影响全表扫描/聚簇索引扫描**
- **剩余空间碎片：数据页中有大量空余空间**导致服务器读取大量不需要的数据

###### 优化数据碎片

- **OPTIMIZE TABLE**

- 不支持OPTIMIZE TABLE时通过**no-op的ALTER TABLE重建表**

  ```mysql
  ALTER TABLE innodb_table ENGINE=INNODB;
  ```

- **数据导出再导入**

- **MyISAM**可通过**排序算法重建索引**消除碎片

- **InnoDB**最新版本可**在线增删索引**

#### 索引案例

- **适合创建索引**

  - **选择性高的列**
  - **频繁作为WHERE查询条件字段**
  - **经常排序(ORDER BY)/分组(GROUP BY)的(多)列**
  - **需要去重的字段(DISTINCT)**
  - **长字符串前缀(前缀索引/哈希索引)**
  - **唯一特性字段（包括组合字段）**

- **不适合创建索引**

  - **复杂且数据量大的字段**
  - **WHERE查询用不到的字段**
  - **表的数据量小**
  - **选择性低的列**
  - **经常更新的表**
  - **无序值(如UUID)**

- **支持多种过滤条件：根据选择性/查询频繁率建普通/组合索引**

  - **选择性低但查询频繁的列**：查询不限制种类时使用**IN(XXX, XXX, XXX)**匹配最左前缀

- **避免多个范围条件**：**范围值/列表值**都会使用EXPLAIN中**type: range**

  - **范围条件查询：<、>、<=、>= 后面不可以用其他索引**
  - **多个等值条件查询：IN() 后面可以用索引**

- **尽量避免OR连接条件**

  - **单列索引OR可使用索引**
  - **组合索引**
    - **表中全部列组成组合索引：OR使用全部列组合索引**
    - **表中部分列组成组合索引：全表扫描**

  - 建议**使用UNION ALL**

- **优化排序**

  - **选择性低的列加特殊索引(选择性低的列+排序列的组合索引)**

  - **翻页优化：加延迟关联**

    ```mysql
    # 普通翻页查询 偏移量大时查询较慢
    SELECT <cols> FROM profiles WHERE sex='M' ORDER BY rating LIMIT 100000, 10;
    
    # 延迟关联优化
    SELECT <cols> FROM profiles 
    INNER JOIN (
      SELECT <primary key cols> FROM profiles
      WHERE x.sex='M' ORDER BY rating LIMIT 100000, 10
    ) AS x 
    USING(<primary key cols>);
    ```

## 查询优化

### 查询基础

<img src="./asset/query1.jpg" alt="query1" style="zoom:27%;" />

<img src="./asset/query2.jpg" alt="query2" style="zoom:27%;" />

### 查询优化器提示

#### HIGH_PRIORITY & LOW_PRIORITY

**多个语句访问某个表时提示某查询语句优先级** **仅对使用表锁的存储引擎有效** 这两种提示会**导致并发插入被禁用 可能导致性能严重下降**

**HIGH_PRIORITY：SELECT重新调度在等待表锁的其他查询语句之前** 用于INSERT时抵消全局LOW_PRIORITY对该插入语句的影响

**LOW_PRIORITY：让SELECT语句一直处于等待 易出现饥饿**

#### DELAYED

**对INSERT/REPLACE有效 使用该提示语句立即返回客户端并将插入行放入缓冲区**以供表空闲时批量写入数据

#### STRAIGHT_JOIN

- **让查询中所有对表按照语句出现的顺序进行关联**
- **固定前后两个表关联顺序**
- **放在SELECT关键字后/任意两个关联表名字间**

#### SQL_SMALL_RESULT & SQL_BIG_RESULT

**对SELECT语句有效**

**SQL_SMALL_RESULT**：提示优化器结果会很小 **将结果集放在内存中的索引临时表 避免排序操作**

**SQL_BIG_RESULT**：提示优化器结果会很大 **建议使用磁盘临时表做排序**

#### SQL_BUFFER_RESULT

提示优化器将**查询结果放入临时表 尽快释放表锁**

#### SQL_CACHE & SQL_NO_CACHE

**提示MySQL结果集是否应该缓存在查询缓存中**

#### SQL_CALC_FOUND_ROWS

让结果集包含更多信息

#### FOR UPDATE & LOCK IN SHARE MODE

**控制SELECT语句锁机制(一致性锁定读) 只对行级锁引擎InnoDB有效** 会让**索引覆盖扫描失效 易导致服务器锁争用**

#### USE INDEX、IGNORE INDEX、FORCE INDEX

查询记录时**控制索引使用**

### SQL语句优化

确认程序**是否存在查询不需要的记录/mysql是否在扫描额外记录** 

1、**查询不需要的记录：使用LIMIT**

2、**尽量避免使用SELECT ***（对 I/O、内存消耗较大）

3、子查询的性能比外连接性能慢：尽量**用外连接来替换子查询**

```mysql
SELECT * FROM A WHERE EXISTS (SELECT * FROM B WHERE id >= 3000 AND A.uuid = B.uuid); 

# 简单优化：用 innerjoin 代替子查询
SELECT * FROM A INNER JOIN B USING(uuid) WHERE b.uuid >= 3000;
```

4、**尽量少排序**	(排序操作会**消耗较多的 CPU 资源**（可以使用索引） )

5、**连续的数值能用 BETWEEN 就不用 IN**

6、**切分查询**：将大查询切分成小查询，每个查询功能一样，只完成一小部分（避免大查询锁粒度大、阻塞其他查询）

### 特定类型查询优化

#### 优化COUNT()查询

##### COUNT()作用

COUNT()是一个特殊函数

主要有两大用途：

- **统计某个列值的数量**
  - **不统计NULL值**(列值需非空)
  - **括号中指定了列/列表达式** 则统计这个列/列表达式**有值的结果数**
- **统计行数**
  - MySQL**确认括号内的表达式值不可能为空时** 统计行数
  - **COUNT(*) 通配符**不会扩展为所有列 而是**忽略所有列直接统计所有(结果集)行数**

Note：

- MySQL知道**某列col不可能有NULL值时**就会内部**将COUNT(col)优化为COUNT(*)**
- MySQL对COUNT(*)和COUNT(1)采取一样的处理和优化 两者性能上无差别 
  - 建议用**COUNT(*)：标准统计行数的语法**

##### MyISAM & COUNT()

使用**MyISAM**作为存储引擎的查询**只有在没有任何WHERE条件时COUNT(*)会非常快** (**直接利用MyISAM特性获得该值**)

##### COUNT(*)特定条件优化

例1：查找**所有ID>5的城市**

```mysql
SELECT COUNT(*) FROM world.City WHERE ID > 5;
```

将**条件反转** 先查找ID<=5的城市数再用总城市数减就能得到相应结果

```mysql
SELECT 
( SELECT COUNT(*) FROM world.City ) # 查询优化时会将子查询直接当常数处理
- COUNT(*) FROM world.City WHERE ID <= 5;
```

```mysql
+----+-------------+-------+...+------+------------------------------+ 
| id | select_type | table |...| rows | Extra 											 | 
+----+-------------+-------+...+------+------------------------------+ 
|  1 | PRIMARY 		 | City  |...| 		6 | Using where; Using index 		 | 
|  2 | SUBQUERY 	 | NULL  |...| NULL | Select tables optimized away | 
+----+-------------+-------+...+------+------------------------------+
```

在**同一查询中统计同一列不同值当数量**来**减少查询语句量**

例2：通过一个查询返回各种不同颜色商品数量

```mysql
# 用SUM
SELECT SUM(color = 'blue') AS blue, SUM(color = 'red') AS red FROM items;
SELECT SUM(IF(color = 'blue', 1, 0)) AS blue,SUM(IF(color = 'red', 1, 0)) AS red FROM items;

# 用COUNT (color为NULL时不计数)
SELECT COUNT(color = 'blue' OR NULL) AS blue, COUNT(color = 'red' OR NULL) AS red FROM items;
```

##### 使用近似值

业务场景有时不要求完全精确的COUNT值 可用近似值代替

**EXPLAIN的优化器估算行数可作为近似值（不需要执行查询 成本低）**

##### 复杂优化

不可能三角**(快速、精确、实现简单) 三者只能满足其二**

- **MySQL层可以使用索引覆盖扫描**
- **修改应用架构 增加汇总表**
- **增加类Memcached的外部缓存系统**

#### 优化关联查询

- **ON/USING子句中的列必须有索引** 
  - 创建该索引时考虑关联顺序
    - 表A和表B用列c关联时 如果优化器关联顺序B->A则不需要在B的对应列建索引
    - 一般**只需要在关联顺序中的第二个表的相应列上创索引**
- **GROUP/ORDER BY中只涉及到一个表中的列(可以用索引)**
- 升级MySQL时注意**关联语法/运算符优先级**(普通关联->笛卡尔积、不同类型关联->不同结果)

#### 优化子查询

MySQL5.6之前：**尽可能使用关联查询代替**

MySQL5.6之后/MariaDB：无需优化子查询

#### 优化GROUP BY & DISTINCT

**优化器会在内部相互转化这两类查询** 

**无法使用索引时 GROUP BY**由两种策略完成

- 使用**临时表**
- **文件排序**做分组

##### 关联查询分组

对**关联查询做GROUP BY(分组)且按照查找表某个列进行分组** 用查找表的**标识列**分组效率更高

```mysql
SELECT actor.first_name, actor.last_name, COUNT(*)
FROM sakila.film_actor
INNER JOIN sakila.actor USING(actor_id) 
# 用first_name, last_name做分组
GROUP BY actor.first_name, actor.last_name;

SELECT actor.first_name, actor.last_name, COUNT(*)
FROM sakila.film_actor
INNER JOIN sakila.actor USING(actor_id)
# 用actor_id(标识列)做分组效率更高
GROUP BY film_actor.actor_id;
# 用actor.actor_id比film_actor.actor_id效率更高
GROUP BY actor.actor_id;
```

以上查询利用first_name、last_name与actor_id直接相关的特性改写后不影响结果 其他情况**SELECT直接使用非分组列可能导致结果不定**

**SELECT后面出现的非分组列一定直接依赖分组列且每组值唯一**（可以**设置SQL_MODE包含ONLY_FULL_GROUP_BY检查**）

##### 分组排序

**没有通过ORDER BY显示指定排序列时自动按照GROUP BY的字段排序**

不关心结果集顺序但默认排序导致文件排序时**可以用ORDER BY NULL消除文件排序**

##### 优化GROUP BY WITH ROLLUP

**WITH ROLLUP**：分组查询变种 **对返回分组结果再做超级聚合**

优化方法

- 用EXPLAIN观察执行计划中分组是否因加/减WITH ROLLUP导致文件排序/加临时表
- 最好的优化是**把聚合功能在应用程序处理**

#### 优化LIMIT分页

**分页操作：LIMIT+偏移量+合适的ORDER BY+合适索引**

##### 延迟关联

**偏移量大**时 如LIMIT10000,20 需要查询10020条记录只返回最后20条 前10000条记录都丢掉 代价非常高

优化偏移量大分页

- **页面中限制分页数量**

- **优化大偏移量性能**

  - 尽可能使用索引覆盖扫描 不查询所有列

  - 做一次关联操作再返回所需列

  - **延迟关联可提升查询效率 扫描尽可能少的页面 获取所需记录再根据关联列回原表查询所需列**

    ```mysql
    # 普通分页
    SELECT film_id, description FROM sakila.film ORDER BY title LIMIT 50, 5;
    
    # 表非常大时延迟关联分页
    SELECT film.film_id, film.description
    FROM sakila.film
    INNER JOIN (
      SELECT film_id FROM sakila.film
      ORDER BY title LIMIT 50, 5
    ) AS lim 
    USING(film_id);
    ```


##### 转换为已知位置查询

将LIMIT查询转换为**已知位置查询** 通过**范围扫描**获得结果

```mysql
# 预先计算边界值且列上有索引
SELECT film_id, description FROM sakila.film 
WHERE position BETWEEN 50 AND 54 ORDER BY position;
```

##### OFFSET偏移量问题

分页中**OFFSET会导致**MySQL**扫描大量不需要的行然后再抛弃掉**

优化：使用**书签记录上次读取数据位置** **下次直接从记录开始位置扫描** 避免OFFSET 无论翻页到多后面性能都比较好

```mysql
# 假设查询返回16049-16030
SELECT * FROM sakila.rental ORDER BY rental_id DESC LIMIT 20;

# 下一次查询可以从16030开始
SELECT * FROM sakila.rental
WHERE rental_id < 16030
ORDER BY rental_id DESC LIMIT 20;
```

##### 使用预先计算汇总表

##### 关联冗余表

冗余表**只包含主键列和需要做排序的数据列**

#### 优化UNION查询

MySQL通过**创建并填充临时表**的方式**执行UNION查询**

除非确实需要服务器消除重复行 否则必须使用**UNION ALL**(**没有ALL会给临时表加DISTINCT做唯一性检查**)

MySQL总会把**结果放入临时表读出再返回给客户端** UNION ALL也得用临时表

#### 静态查询分析

使用Percona Toolkit中pt-query-advisor工具可解析查询日志、分析查询模式、对可能存在问题的查询给出建议

#### 用户自定义变量

用户自定义变量是用来**存储内容的临时容器**

```mysql
# 用SET和SELECT语句定义
SET @one 			 := 1;
SET @min_actor := (SELECT MIN(actor_id) FROM sakila.actor); 
SET @last_week := CURRENT_DATE-INTERVAL 1 WEEK;

# 在任意表达式使用变量
SELECT ... WHERE col <= @last_week;
```

##### 用户自定义变量限制

1. 使用自定义变量的查询**无法使用查询缓存**
2. **不能在使用常量/标识符的地方(表名、列名、LIMIT子句)使用**自定义变量
3. **不能用自定义变量做连接间通信** 其生命周期仅在一个连接内有效
4. 使用连接池/持久化连接时用自定义变量可能会让**无关系代码交互**
5. MySQL5.0之前**大小写敏感**
6. **不能显式声明自定义变量类型(动态类型)** 
7. **优化器可能会消除变量**
8. **赋值顺序/赋值时间点不固定** 取决于优化器
9. 赋值符号:=优先级非常低 **赋值表达式需用明确括号**
10. **使用未定义变量不会产生语法错误**

##### 优化排名语句

变量重要特性：**可以在一个变量赋值的同时使用该变量**（左值）

```mysql
# 行号功能
SET @rownum := 0;
SELECT actor_id, @rownum := @rownum + 1 AS rownum FROM sakila.actor LIMIT 3; 
+----------+--------+ 
| actor_id | rownum | 
+----------+--------+ 
| 			 1 | 			1 | 
| 			 2 | 			2 | 
| 			 3 | 			3 | 
+----------+--------+
```

查询获取电影参演数量最多的演员并根据电影数量排名

```mysql
SELECT actor_id, COUNT(*) as cnt 
FROM sakila.film_actor
GROUP BY actor_id
ORDER BY cnt DESC
LIMIT 10; 
+----------+-----+ 
| actor_id | cnt | 
+----------+-----+ 
| 		 107 | 	42 | 
| 		 102 |  41 | 
|			 198 | 	40 | 
...
```

使用三个变量但无法更新排名(**EXPLAIN**中显示由于**变量赋值时间不同**导致无法更新)

```mysql
SET @curr_cnt := 0, @prev_cnt := 0, @rank := 0;
SELECT 
	actor_id, 
	@curr_cnt := COUNT(*) AS cnt,
	@rank := IF(@prev_cnt <> @curr_cnt, @rank + 1, @rank) AS rank,
	@prev_cnt := @curr_cnt AS dummy
FROM sakila.film_actor
GROUP BY actor_id
ORDER BY cnt DESC
LIMIT 10;
```

**解决变量赋值时间不定：在FROM子句使用子查询生成中间临时表**

```mysql
SET @curr_cnt := 0, @prev_cnt := 0, @rank := 0;
SELECT 
	actor_id, 
	@curr_cnt := cnt AS cnt,
	@rank := IF(@prev_cnt <> @curr_cnt, @rank + 1, @rank) AS rank,
	@prev_cnt := @curr_cnt AS dummy
FROM (
  SELECT actor_id, COUNT(*) AS cnt
  FROM sakila.film_actor
  GROUP BY actor_id
  ORDER BY cnt DESC
  LIMIT 10;
) as der;
+----------+-----+------+-------+ 
| actor_id | cnt | rank | dummy | 
+----------+-----+------+-------+ 
| 		 107 | 	42 | 		1 | 	 42 | 
| 		 102 | 	41 |	  2 | 	 41 | 
| 		 198 | 	40 | 		3 | 	 40 | 
..
```

##### 避免重复查询刚更新的数据

可以用变量**更新行的时候同时返回该行信息**

```mysql
# 更新记录时间戳并查询当前记录存放的时间戳
UPDATE t1 SET lastUpdated = NOW() WHERE id = 1; 
SELECT lastUpdated FROM t1 WHERE id = 1;

# 用变量仍需要两个查询 但第二次查询无需访问任何数据表
UPDATE t1 SET lastUpdated = NOW() WHERE id = 1 AND @now := NOW(); 
SELECT @now;
```

##### 统计更新和插入数量

使用**INSERT ON DEPLICATE KEY UPDATE**时使用变量可知**插入行数据的数量和因数据冲突改写成更新操作的数据数量**

```mysql
# 冲突更新时对变量@x自增 对这个表达式乘0让其不要影响更新内容 MySQL会返回被更改总行数
INSERT INTO t1(c1, c2) VALUES(4, 4), (2, 1), (3, 1) ON DUPLICATE KEY UPDATE
c1 = VALUES(c1) + ( 0 * ( @x := @x +1 ) );
```

##### 确定取值顺序

**SELECT和WHERE执行阶段不同** **加入ORDER BY引入文件排序(WHERE在文件排序前取值)** 

```mysql
SET @rownum := 0;
# 返回表全部记录
SELECT actor_id, @rownum := @rownum + 1 AS cnt
FROM sakila.actor
WHERE @rownum <= 1
ORDER BY first_name;


# 解决办法：变量赋值/取值发生在执行查询同一阶段
SET @rownum := 0;
SELECT actor_id, @rownum AS rownum
FROM sakila.actor
WHERE (@rownum := @rownum + 1) <= 1; 
+----------+--------+ 
| actor_id | rownum | 
+----------+--------+ 
| 			 1 | 1 			| 
+----------+--------+
```

将**赋值语句放到LEAST()函数中**可以在**完全不改变排序顺序的同时完成赋值操作** (**不希望对子句执行结果有影响但又要完成赋值**时使用)

类似函数：GREATEST()、LENGTH()、ISNULL()、NULLIF()、IF()、COALESCE()

```mysql
SET @rownum := 0;
SELECT actor_id, first_name, @rownum AS rownum
FROM sakila.actor
WHERE @rownum <= 1
ORDER BY first_name, LEAST(0, @rownum := @rownum + 1);
```

##### 懒UNION

**懒UNION查询：第一个子查询先执行 找到匹配行跳过第二个子查询** 

**提高缓存命中率**：**先**在频繁访问表**找热数据** **找不到**再去较少访问表**找冷数据**

```mysql
# 常规UNION ALL：必须访问两个表
SELECT id FROM users WHERE id = 123 
UNION ALL 
SELECT id FROM users_archived WHERE id = 123;

# 使用@found变量完成懒UNION
SELECT GREATEST(@found := −1, id) AS id, 'users' AS which_tbl 
FROM users WHERE id = 1 
UNION ALL
	SELECT id, 'users_archived'
	FROM users_archived WHERE id = 1 AND @found IS NULL 
UNION ALL
	SELECT 1, 'reset' FROM DUAL WHERE ( @found := NULL ) IS NOT NULL;
```

##### 用户自变量其他用处

- 查询运行时计算总数/平均值
- 模拟GROUP语句中的函数FIRST()/LAST()
- 大量数据计算
- 计算大表MD5散列值
- 样本处理函数
- 模拟读写游标











