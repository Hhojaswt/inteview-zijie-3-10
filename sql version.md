### **1. 基础语法与概念**
**问题：**  
解释 `INNER JOIN`、`LEFT JOIN`、`RIGHT JOIN` 和 `FULL OUTER JOIN` 的区别。  
**答案：**  
- **INNER JOIN**：仅返回两个表中匹配的行。  
- **LEFT JOIN**：返回左表所有行，右表不匹配的行用 `NULL` 填充。  
- **RIGHT JOIN**：返回右表所有行，左表不匹配的行用 `NULL` 填充。  
- **FULL OUTER JOIN**：返回左右表所有行，不匹配的部分用 `NULL` 填充（MySQL 不支持，需用 `UNION` 实现）。  
```sql
SELECT A.列1, A.列2, B.列3
FROM 表A
LEFT JOIN 表B
ON A.公共列 = B.公共列;

CROSS JOIN（交叉连接）
笛卡尔积（每一行匹配另一表的所有行）。
SELECT A.列1, B.列2
FROM 表A
CROSS JOIN 表B;  #生成所有 可能的匹配组合
```

## **哈希连接的原理** ##
Hash Join 主要用于 大规模数据集 之间的连接，尤其适用于 无索引的表连接，在 等值连接（A.col = B.col）情况下性能最佳。
- 构建哈希表（Build Phase）：
选择 较小的表（或较小的数据集） 作为 构建表（Build Table）。以 JOIN 连接键为 哈希键，为其创建 哈希表（Hash Table），存入内存。
- 探测匹配（Probe Phase）：
扫描 较大的表（Probe Table），使用相同的哈希函数计算 JOIN 连接键的哈希值，在哈希表中查找匹配项。
```sql
- 返回匹配数据：
如果哈希表中找到了匹配值，则返回结果，否则跳过该行。
SELECT A.ID, A.Name, B.Salary
FROM A
JOIN B ON A.ID = B.ID;
```
## **嵌套循环连接** ##
外层循环遍历 表A 的每一行，内层循环在 表B 中查找匹配行。

## **归并连接** ##
适用于 已排序的数据。顺序遍历两个表，从小到大逐步匹配，提高效率。


如果 两张表很小 ➝ 嵌套循环（Nested Loop）
如果 一张表小，一张表大 ➝ 哈希连接（Hash Join）
如果 两张表都很大且已排序 ➝ 归并连接（Merge Join）
如果 表有索引 ➝ 索引嵌套循环（Index Nested Loop）

---

### **2. 索引与性能优化**
**问题：**  
什么情况下索引会失效？如何避免？ 
```sql
CREATE INDEX idx_column_name ON table_name(column_name);
```
**答案：**  
索引失效的常见场景：  
- 对索引列进行运算或函数操作（如 `WHERE YEAR(date_col) = 2023`）.索引的核心作用是加速数据查找，它依赖于索引的有序性 来快速定位数据。然而，当对索引列进行运算或函数操作（如 YEAR(date_col) = 2023），数据库无法利用索引的有序性，导致索引失效，回退为全表扫描
- 使用 `LIKE` 以通配符开头（如 `LIKE '%abc'`）。 不能使用 B-Tree 索引，因为 % 号在前，无法利用索引的有序性。 
- 违反最左前缀原则（复合索引未按顺序使用）。B-Tree 索引中，查询条件必须从索引的最左列开始，否则索引可能不会被完全利用,索引会按照 age（第一列）排序，其次 city（第二列），最后 name（第三列），必须先使用 age，否则索引可能无法生效
- 数据类型隐式转换（如字符串列用数字查询）。WHERE phone = 13812345678,如果 phone 是 VARCHAR(11) 类型，而 13812345678 是 INT，MySQL 会将 phone 转换为 INT,改为phone = '13812345678'
**优化方法**：避免上述操作，使用覆盖索引，定期分析索引使用情况。

---

### **3. 复杂查询与窗口函数**
**问题：**  
有一张用户行为表 `user_actions`，字段为 `user_id, action_time, event`。找出每个用户最近一次登录的时间。  
**答案：**  
```sql
SELECT user_id, MAX(action_time) AS last_login
FROM user_actions
WHERE event = 'login'
GROUP BY user_id;
```
或使用窗口函数：  
```sql
SELECT user_id, action_time 
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY action_time DESC) AS rn
  FROM user_actions
  WHERE event = 'login'
) t
WHERE rn = 1; #只保留 rn = 1 的记录，即每个用户最新的 login 记录。
```

---

### **4. 数据去重与清理**
**问题：**  
如何删除表中重复数据（保留一条）？  
**答案：**  
```sql
DELETE FROM table
WHERE id NOT IN (
  SELECT MIN(id) 
  FROM table 
  GROUP BY col1, col2  -- 按重复字段分组
);
```
DISTINCT 只会去掉查询结果中的重复行，但并不会真正删除数据库中的重复数据。
---

### **5. 分页查询优化**
**问题：**  
大数据量下如何高效实现分页？  
**答案：**  
避免 `LIMIT N OFFSET M`（扫描前 M+N 行），改为基于索引的条件分页：  
```sql
SELECT * FROM table 
WHERE id > last_seen_id  -- 记录上一页末尾的 ID
ORDER BY id 
LIMIT 10;
```

---

### **6. 事务与锁机制**
**问题：**  
解释脏读、不可重复读和幻读的区别。  
**答案：**  
- **脏读**：读到其他事务未提交的数据。  
- **不可重复读**：同一事务内两次读取同一数据结果不同（数据被修改）。  
- **幻读**：同一事务内两次查询结果集不同（数据被新增或删除）。  
**隔离级别**：  
- `READ UNCOMMITTED` 允许脏读。 事务 A 修改了一条数据，但还没有提交（COMMIT）。事务 B 读取了事务 A 修改的数据。如果事务 A 回滚（ROLLBACK），那么事务 B 读取的数据就成了“脏数据”（不存在的数据）。
- `READ COMMITTED` 避免脏读。  
- `REPEATABLE READ`（MySQL 默认）避免脏读和不可重复读。  
- `SERIALIZABLE` 避免所有问题，但性能最低。

---

### **7. 数据库设计**
**问题：**  
设计一个短视频评论系统，如何设计表结构？  
**答案：**  
核心表设计：  
- **用户表**：`user_id, username, ...`  
- **视频表**：`video_id, user_id, title, ...`  
- **评论表**：`comment_id, video_id, user_id, content, create_time, parent_comment_id`（支持回复）。  
**索引优化**：对 `video_id` 和 `create_time` 加索引，按时间倒序分页查询。

---

### **8. 数据倾斜处理**
**问题：**  
SQL 查询中出现数据倾斜如何解决？  
**答案：**  
- **预处理**：将倾斜的 Key 单独处理（如加盐打散）。  
- **优化 JOIN**：大表 JOIN 小表时用 `MAPJOIN`（Hive）。  
- **调整 SQL 逻辑**：避免 `GROUP BY` 或 `DISTINCT` 倾斜 Key。

---

### **9. 主从同步与高可用**
**问题：**  
主库（Master） 执行事务，写入 binlog（二进制日志）。
从库（Slave） 读取 binlog 并写入 relay log（中继日志）。
从库（Slave） 执行 relay log 中的 SQL 语句。
MySQL 主从复制延迟过高，可能原因及解决方案？  
**答案：**  
**原因**：  
- 主库写入压力大。  
- 从库硬件性能差。  
- 长事务或大事务阻塞。  
**解决**：  
- 升级从库硬件。  
- 分库分表降低单库压力。  
- 使用半同步复制或并行复制。
- Step 2（IO 线程获取 binlog） → 网络延迟
Step 3（SQL 线程执行 relay log） → SQL 复杂度、IO 瓶颈、锁等待
析慢查询：
SHOW PROCESSLIST;
主从间使用私有网络，避免公网 IP 复制数据。检查网络：ping、traceroute 查看网络延迟。


---

### **10. 场景分析**
**问题：**  
某查询突然变慢，如何排查？  
**答案：**  
1. 检查是否索引失效或未命中。  
2. 使用 `EXPLAIN` 分析执行计划。  
3. 查看锁竞争（`SHOW PROCESSLIST`）。  
4. 检查数据量是否激增或统计信息过期（更新 `ANALYZE TABLE`）。  
5. 考虑服务器负载（CPU/IO 瓶颈）。
