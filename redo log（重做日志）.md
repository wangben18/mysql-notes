## Checkpoint 技术
- WAL(Write Ahead Log) 先写重做日志，再修改页。当发生宕机时，可以通过重做日志来恢复数据到宕机发生时刻，但这需要两个目前无法很好满足的前提条件
	1. 缓冲池可以缓存数据库中的所有数据
	2. 重做日志可以无限增大
- Checkpoint 技术解决以下问题
	- 缩短数据库恢复时间
	- 缓冲池不够用时，将脏页刷新到磁盘
	- 重做日志不可用时，刷新脏页
- 当数据库发生宕机时不需要重做所有日志，只需对 Checkpoint 后的重做日志进行恢复，这样大大缩短恢复时间
- 当[[体系结构#^b1756e|缓冲池]]不够用时，LRU 算法会溢出最近最少使用的页，若为脏页，则强制执行 Checkpoint，将脏页也就是页的新版本刷回磁盘
- 通过 LSN(Log Sequence Number，8 字节数字)标记版本

- Sharp Checkpoint
	- 发生在数据库关闭时将所有脏页刷新回磁盘
	- 若在运行时使用，可用性将受到很大的影响
	- 
- Fuzzy Checkpoint
	- 可能发生的情况
		- Master Thread Checkpoint
			- 每秒或每十秒从缓冲池脏页列表中异步刷新一定比例的页回磁盘，用户查询线程不会阻塞
		- FLUSH_LRU_LIST Checkpoint
			- 要保证 LRU 列表中有 100 个空闲页，若没有，则需要将列表尾端的页移除，若其中有脏页，则需要进行 Checkpoint
			- 从 MySQL 5.6 版本，即 InnoDB 1.2.x 版本开始，这个检查被放在 Page Cleaner 线程中进行
		- Async/Sync Flush Checkpoint
			- 指的是重做日志文件不可用的情况
			- 从 MySQL 5.6 版本，即 InnoDB 1.2.x 版本开始，这个检查被放在 Page Cleaner 线程中进行
		- Dirty Page too much Checkpoint
			- 脏页数量太多，强制进行 Checkpoint