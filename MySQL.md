- ACID
	- Atomicity 原子性
	- Consistency 一致性
	- Isolation 隔离性
		- [[事务]]
	- Durability 持久性
		- [[redo log（重做日志）]]

---
- [[部署与连接（docker）]]
---
## 逻辑架构图
![[基础架构.jpeg]]
## MySQL 体系结构
![[MySQL体系结构.png]]


---
## 日志系统
- binlog（归档日志）
	- 由MySQL Server 层实现
	- 记录原始逻辑如“给ID=2这一行的c字段加1”
	- 追加写入，文件写到一定大小后会切换到下一个，不会覆盖以前的日志
- [[redo log（重做日志）]]
	- 通过 write_pos、check_point 循环写入
	- InnoDB 引擎特有
	- 记录物理日志，“在某个数据页上做了什么修改”
![[redo log 循环写入.jpeg]]
- [[两阶段提交]]
---
- [[索引]]
---
- InnoDB
	- [[体系结构]]
	- [[Master Thread 工作方式]]
---
## 全局锁
```SQL
Flush tables with read lock
```
释放锁（客户端断开时自动释放）
```SQL
unlock tables
```
让整个库处于只读状态，其他线程的以下语句会被阻塞：
- 数据更新（增删改）
- 数据定义（建表、修改表结构）
- 更新类事务的提交语句

问题
- 备份主库：不能执行更新，业务停摆
- 备份从库：不能执行主库同步的binlog，导致主从延迟

不锁会导致备份视图不一致

支持事务引擎的库使用 mysqldump 工具备份，通过参数 -single-transaction 启动事务备份。

## 表级锁
### 表锁
```SQL
lock tables ··· read/write
```
释放锁（客户端断开时自动释放）
```SQL
unlock tables
```
- 支持行锁的 InnoDB 一般不使用表锁
### 元数据锁（meta data lock, MDL）
- 不需要显式使用，访问表时会自动加上
- 在 MySQL 5.5版本中加入
- 引发锁问题![[MDL引发锁等待问题.png]]
	- 解决
		- 解决长事务
		- 在 alter table 语句里设定等待时间，拿不到MDL锁时不阻塞，先放弃

## MySQL “抖一下” 问题
- 当内存数据页与磁盘数据页内容不一致时，称该内存页为“脏页”。内存数据写入到磁盘后，称为“干净页”。“抖一下”时可能在 flush 脏页

### flush 脏页的四个场景
1. redo log 写满了。会停止所有操作，把checkpoint往前推进并将对应脏页flush到磁盘
	1. 需尽量避免，所有的更新会被阻塞
2. 系统内存不足。需要淘汰数据页。若淘汰的是脏页，则flush到磁盘
	1. 是常态
	2. InnDB 使用 buffer pool 管理内存，有三种状态
		1. 还没有使用
		2. 干净页
		3. 脏页
3. MySQL 认为“空闲”时
4. MySQL 正常关闭时

### InnoDB 刷脏页的控制策略
- 设置 innodb_io_capacity 参数为磁盘 IOPS 告诉 InnoDB 磁盘能力
- 磁盘的 IOPS 可用 fio 工具测试
```bash
fio filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest
```
- innodb_max_dirty_pages_pct 是脏页比例上限，默认值为 75%
- 平时关注脏页比例，不要让它经常接近 75%。脏页比例可用以下SQL查询
```SQL
select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME =
'Innodb_buffer_pool_pages_dirty';
select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME =
'Innodb_buffer_pool_pages_total';
select @a/@b;
```
- innodb_flush_neighbors
	- 在准备flush脏页时，会将“邻居”脏页一并刷掉的“连坐”机制，在使用机械硬盘时很有意义，但 SSD 这类高 IOPS 的设备“只刷自己”会减少SQL语句响应时间
	- MySQL 8.0 中该参数默认值已为 0

---
- 使用 [pt-query-digest](https://docs.percona.com/percona-toolkit/pt-query-digest.html) 测试所有 SQL 语句的返回结果
---

