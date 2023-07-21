## InnoDB 1.0.x 版本之前
- 具有最高的线程优先级别
- 内部由多个循环（loop）组成，Master Thread 根据数据库运行状态切换
	- 主循环 loop
	- 后台循环 background loop
	- 刷新循环 flush loop
	- 暂停循环 suspend loop

### 主循环 - loop
- 通过 thread sleep 实现（所以不精确），负载很大的情况下可能会有延迟
- 每秒一次的操作
	- 日志缓冲刷新到磁盘（即使这个事务还没有提交）
	- 合并插入缓冲
		- 判断当前一秒内发生的 IO 次数，如果小于 5 次，则认为当前 IO 压力很小，可以执行合并插入缓冲操作
	- 至多刷新 100 个 InnoDB 的缓冲池脏页到磁盘
		- 判断当前缓冲池中脏页的比例（buf_get_modified_ratio_pct）是否超过配置文件中的阈值（innodb_max_dirty_pages_pct）
	- 如果当前没有用户活动，切换到background loop
- 每 10 秒的操作
	- 刷新 100 个脏页到磁盘
		- 判断过去 10 秒内 IO 操作是否小于 200 次才执行
	- 合并至多 5 个插入缓冲 ^36149e
	- 将日志缓冲刷新到磁盘
	- 删除无用的 Undo 页
	- 刷新 100 个或者 10 个脏页到磁盘

### 后台循环 background loop
没有用户活动（数据库空间时）或者数据库关闭（shutdown），就会切换到这个循环。
- 删除无用的 Undo 页
- 合并 20 个插入缓冲
- 跳回到主循环
- 不断刷新 100 个页直到符合条件（可能，跳转到 flush loop 中完成）
- 若 flush loop 也没事可做了，就切换到 suspend loop，将 Master Thread 挂起

## InnoDB 1.2.x 版本之前的 Master Thread
- 1.0.x 版本对于 IO 有限制，缓冲池向磁盘刷新时做了一定的硬编码，当固态硬盘出现时，这种规定在很大程度上限制了 InnoDB 对磁盘 IO 的性能，尤其是写入性能
- 因此InnoDB Plugin（从 InnoDB 1.0.x 版本开始）提供了参数 innodb_io_capacity，用来表示磁盘 IO 的吞吐量
- innodb_max_dirty_pages_pct 默认值为 90 “太大”了，因为在每秒刷新和 flush loop 时会判断该值，超出了才刷新 100 个脏页，如果有很大的内存或者数据库服务器压力很大，这时刷新脏页的速度反而会降低
- 1.0.x版本引入参数 innodb_purge_batch_size ，可以控制每次 full purge 回收的 Undo 页的数量，默认值为 20 （之前最多回收 20 个 Undo 页）。可以动态修改

## InnoDB 1.2.x 版本的 Master Thread
刷新脏页的操作从 Master Thread 分离到单独的 Page Cleaner Thread