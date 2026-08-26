# MySQL数据库常见问题手�?

## 1. 连接与性能类问�?

### 1.1 "Too many connections" (连接数过�?

**现象**：应用无法连接数据库，报错`ERROR 1040 (HY000): Too many connections`

**根本原因**�?
- 应用连接池配置过大或未正确释放连�?
- 存在大量Sleep空闲连接
- `max_connections`设置过小

**诊断步骤**�?
```sql
-- 查看当前连接�?
SHOW STATUS LIKE 'Threads_connected';
-- 查看最大使用过的连接数
SHOW STATUS LIKE 'Max_used_connections';
-- 查看连接数上�?
SHOW VARIABLES LIKE 'max_connections';
-- 查看连接状态分�?
SELECT state, COUNT(*) FROM information_schema.PROCESSLIST GROUP BY state;
```

**解决方案**�?
```sql
-- 临时增加连接数（重启失效�?
SET GLOBAL max_connections = 1000;

-- 批量杀掉空闲超�?小时的连�?
SELECT CONCAT('KILL ', id, ';') FROM information_schema.PROCESSLIST 
WHERE Command = 'Sleep' AND Time > 3600;
```

**长期优化**�?
```ini
[mysqld]
max_connections = 800          # 根据内存调整：max_connections * (sort_buffer_size + join_buffer_size...) �?总内�?
extra_port = 13306             # 预留管理端口
wait_timeout = 600             # 空闲连接超时时间(�?
interactive_timeout = 600
```

### 1.2 "Lock wait timeout exceeded" (锁等待超�?

**现象**：`ERROR 1205: Lock wait timeout exceeded; try restarting transaction`

**根本原因**：事务A持有行锁未提交，事务B等待超时

**诊断步骤**�?
```sql
-- 查看当前锁等待关�?
SELECT * FROM sys.innodb_lock_waits;

-- 查看正在运行的事务（8.0�?
SELECT * FROM performance_schema.innodb_trx\G

-- 找出未提交的事务
SELECT trx_id, trx_state, trx_mysql_thread_id, trx_query, trx_started
FROM information_schema.INNODB_TRX 
WHERE trx_state = 'RUNNING';
```

**解决方案**�?
```sql
-- 杀掉阻塞源（使用上一步查出的trx_mysql_thread_id�?
KILL 12345;
```

**预防措施**�?
- 确保事务短小精悍，及时COMMIT
- 避免在事务中进行远程调用或复杂业务逻辑
- 检查应用是否开启自动提交：`SHOW VARIABLES LIKE 'autocommit';`

### 1.3 死锁 (Deadlock)

**现象**：`ERROR 1213: Deadlock found when trying to get lock; try restarting transaction`

**诊断**：查看最近一次死锁信�?
```sql
SHOW ENGINE INNODB STATUS\G
-- 查看输出中的 "LATEST DETECTED DEADLOCK" 部分
```

**常见死锁场景及解�?*�?

| 场景               | 原因                   | 解决方案                                       |
| :----------------- | :--------------------- | :--------------------------------------------- |
| 多表更新顺序不一�?| A更新t1→t2，B更新t2→t1 | 统一更新顺序                                   |
| 批量删除+插入并发  | 间隙�?Gap Lock)冲突   | 降低隔离级别为RC(Read Committed)               |
| 主键冲突           | 并发INSERT相同唯一�?  | 使用`INSERT IGNORE`或`ON DUPLICATE KEY UPDATE` |

**临时处理**：应用层捕获死锁异常并重试（通常重试1-2次即可成功）�?

### 1.4 慢查询导致数据库响应缓慢

**现象**：整体QPS下降，CPU飙升，应用超�?

**定位慢SQL**�?
```sql
-- 查看当前正在执行的慢查询
SELECT * FROM information_schema.PROCESSLIST WHERE Time > 10;

-- 开启慢查询日志（临时）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = ON;
```

**分析慢SQL**�?
```sql
-- 使用EXPLAIN分析执行计划
EXPLAIN SELECT * FROM orders WHERE user_id = 12345;
```

**关键执行计划解读**�?

| type�?    | 含义         | 处理方式       |
| :--------- | :----------- | :------------- |
| ALL        | 全表扫描     | 紧急：添加索引 |
| index      | 全索引扫�?  | 仍需优化       |
| range      | 索引范围扫描 | 可接�?        |
| ref/eq_ref | 索引精确查找 | 理想           |

**紧急处�?*：发现全表扫描大表时，可临时添加索引�?
```sql
ALTER TABLE orders ADD INDEX idx_user_id (user_id);
```

---

## 2. 数据一致性问�?

### 2.1 主从复制延迟

**现象**：从库数据落后主库，`Seconds_Behind_Master > 0`且持续增�?

**诊断**�?
```sql
-- 从库执行
SHOW SLAVE STATUS\G
-- 关键字段：Seconds_Behind_Master, Relay_Log_Space, Slave_IO_Running, Slave_SQL_Running
```

**常见原因及处�?*�?

| 原因         | 特征                  | 解决方案                       |
| :----------- | :-------------------- | :----------------------------- |
| 从库硬件�?  | IO延迟�?             | 升级从库配置，或启用并行复制   |
| 大事�?      | 单个事务执行时间>延迟 | 拆分大事务，避免批量操作       |
| 无主键表     | 复制每行需全表扫描    | 为所有表添加主键               |
| 从库有读负载 | 从库CPU/IO�?         | 读写分离迁移部分查询到其他从�?|

**并行复制配置�?.0推荐�?*�?
```ini
[mysqld]
slave_parallel_workers = 8
slave_parallel_type = LOGICAL_CLOCK
```

**紧急追�?*：延迟过大时，考虑重新搭建从库或临时暂停写入�?

### 2.2 主从复制中断

**现象**：`Slave_SQL_Running: No`，复制停�?

**查看错误**�?
```sql
SHOW SLAVE STATUS\G
-- 查看 Last_Errno �?Last_Error 字段
```

**常见错误码及处理**�?

| 错误�?| 含义                 | 解决方案                                         |
| :----- | :------------------- | :----------------------------------------------- |
| 1062   | 主键/唯一键冲�?     | `SET GLOBAL sql_slave_skip_counter = 1;` 跳过1�?|
| 1032   | 记录不存在（被删除） | 同上跳过，或从主库补数据                         |
| 1146   | 表不存在             | 从主库同步表结构                                 |
| 1452   | 外键约束失败         | 检查主从数据一致�?                              |

**安全跳过方法**�?
```sql
-- 方法1：跳�?个事�?
STOP SLAVE;
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE;

-- 方法2：基于GTID跳过（推荐）
STOP SLAVE;
SET GTID_NEXT = '对应的失败GTID';
BEGIN; COMMIT;  -- 空事务覆�?
SET GTID_NEXT = 'AUTOMATIC';
START SLAVE;
```

---

## 3. 存储与空间问�?

### 3.1 磁盘空间不足

**现象**：写入失败，错误日志报`No space left on device`

**紧急处理步�?*�?
```bash
# 1. 定位占用空间的文�?
du -sh /var/lib/mysql/* | sort -rh | head -20

# 2. 清理二进制日志（紧急情况）
mysql -e "PURGE BINARY LOGS BEFORE NOW();"

# 3. 清理慢查询日志、错误日�?
> /var/log/mysql/slow-query.log
> /var/log/mysql/error.log

# 4. 如果还不足，删除过期binlog
mysql -e "SHOW BINARY LOGS;"  # 查看binlog列表
mysql -e "PURGE BINARY LOGS TO 'mysql-bin.000150';"
```

**预防配置**�?
```ini
[mysqld]
expire_logs_days = 7        # binlog保留7天（8.0使用binlog_expire_logs_seconds�?
innodb_data_file_path = ibdata1:12M:autoextend:max:10G  # 限制系统表空间增�?
```

### 3.2 表空间碎片严�?

**现象**：表数据量不大但占用空间很大，查询性能下降

**诊断碎片**�?
```sql
-- 查看碎片�?
SELECT 
    table_name,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(data_free / 1024 / 1024, 2) AS free_mb,
    ROUND(data_free / data_length * 100, 2) AS fragment_rate
FROM information_schema.TABLES 
WHERE table_schema = 'your_database' 
  AND data_free > 0
ORDER BY fragment_rate DESC;
```

**整理碎片**�?
```sql
-- 方法1：OPTIMIZE（会锁表，InnoDB实际是重建表�?
OPTIMIZE TABLE large_table;

-- 方法2：重建表�?.0支持即时DDL�?
ALTER TABLE large_table ENGINE = InnoDB;

-- 方法3：pt-online-schema-change（生产环境推荐）
pt-online-schema-change --alter "ENGINE=InnoDB" D=database,t=table
```

### 3.3 "Table is full" (表已�?

**现象**：`ERROR 1114: The table 'xxx' is full`

**原因及解�?*�?
- **InnoDB**：通常是因为`innodb_data_file_path`设置了`autoextend`上限
  ```sql
  -- 修改为无限扩�?
  ALTER TABLESPACE innodb_system SET AUTOEXTEND_SIZE = 64M;
  ```
- **磁盘空间�?*：参�?.1节处�?
- **临时表空间满**：增加`innodb_temp_data_file_path`大小

---

## 4. 崩溃与恢复问�?

### 4.1 MySQL无法启动

**现象**：`systemctl start mysql`失败

**排查步骤**�?
```bash
# 1. 查看错误日志（最关键�?
tail -100 /var/log/mysql/error.log

# 2. 检查配置文件语�?
mysqld --validate-config

# 3. 检查端口占�?
netstat -tlnp | grep 3306

# 4. 检查数据目录权�?
chown -R mysql:mysql /var/lib/mysql
```

**常见启动失败错误**�?

| 错误信息                                     | 原因                     | 解决                                        |
| :------------------------------------------- | :----------------------- | :------------------------------------------ |
| `InnoDB: Unable to lock ./ibdata1 error: 11` | 文件锁冲突，已有进程运行 | `killall -9 mysql` 清理残留进程             |
| `InnoDB: corrupted page`                     | 数据页损�?              | 设置`innodb_force_recovery=1`后启动导出数�?|
| `Can't open the mysql.plugin table`          | 系统表损�?              | 使用mysql_upgrade或恢复备�?                |

### 4.2 InnoDB损坏修复

**恢复流程**（数据优先）�?
```ini
[mysqld]
innodb_force_recovery = 1   # �?开始，逐步增加�?
# 1: 忽略损坏页继续运行（最安全�?
# 2: 禁止后台线程运行
# 3: 不执行事务回�?
# 4: 不合并插入缓�?
# 5: 不查看undo log（可能丢失未提交数据�?
# 6: 不进行前滚恢复（最激进）
```

**恢复步骤**�?
```bash
# 1. 设置force_recovery=1启动MySQL
# 2. 用mysqldump导出所有可读数�?
mysqldump --all-databases --single-transaction > /backup/all.sql
# 3. 停止MySQL，删除损坏的数据目录
# 4. 重新初始化数据库
mysqld --initialize-insecure --user=mysql
# 5. 导入备份数据
mysql < /backup/all.sql
```

---

## 5. SQL执行异常

### 5.1 "Data too long for column"

**现象**：`ERROR 1406: Data too long for column 'name'`

**诊断**�?
```sql
-- 查看字段定义长度
SHOW COLUMNS FROM table_name LIKE 'name';
-- 检查插入数据的实际长度
SELECT LENGTH(name) FROM ...;
```

**解决**�?
```sql
-- 临时放宽字段长度（生产需评估影响�?
ALTER TABLE table_name MODIFY COLUMN name VARCHAR(500);
```

### 5.2 "Incorrect string value" (字符集问�?

**现象**：插入emoji或特殊字符时报错

**根本原因**：表或连接的字符集不是`utf8mb4`（MySQL的utf8是utf8mb3，不支持4字节字符�?

**诊断**�?
```sql
SHOW VARIABLES LIKE 'character_set_%';
SHOW CREATE TABLE table_name;
```

**解决**�?
```sql
-- 修改表字符集
ALTER TABLE table_name CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
-- 修改库默认字符集
ALTER DATABASE db_name CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
```

---

## 6. 快速参考表

### 6.1 常用诊断SQL汇�?

| 需�?                 | SQL                                                          |
| :-------------------- | :----------------------------------------------------------- |
| 查看所有连�?         | `SHOW PROCESSLIST;`                                          |
| 查看当前事务          | `SELECT * FROM information_schema.INNODB_TRX;`               |
| 查看锁等�?           | `SELECT * FROM sys.innodb_lock_waits;`                       |
| 查看表大�?           | `SELECT table_name, ROUND(data_length/1024/1024,2) AS size_mb FROM information_schema.TABLES WHERE table_schema='db';` |
| 查看Buffer Pool命中�?| `SHOW STATUS LIKE 'Innodb_buffer_pool_read%';`               |
| 查看QPS/TPS           | `SHOW GLOBAL STATUS WHERE Variable_name IN ('Questions','Uptime');` |

### 6.2 紧急操作速查

| 场景                      | 命令                                                         |
| :------------------------ | :----------------------------------------------------------- |
| 紧急杀查询                | `KILL query <thread_id>;`                                    |
| 杀全部慢查�?             | `SELECT CONCAT('KILL ',id,';') FROM information_schema.PROCESSLIST WHERE Time>60;` |
| 刷新查询缓存�?.0已废弃） | `RESET QUERY CACHE;`                                         |
| 清空表重�?               | `TRUNCATE TABLE table_name;`                                 |
| 强制刷新日志              | `FLUSH LOGS;`                                                |
| 重新加载权限�?           | `FLUSH PRIVILEGES;`                                          |

