# **PURGE_LOG()**

## **函数说明**

`PURGE_LOG()` 用于删除记录于 MatrixOne 数据库系统表中的日志。返回 0 表示删除成功；删除失败则返回报错信息。

!!! note
    目前，仅有 root 用户（即集群管理员，具有 `MOADMIN` 权限）拥有执行 `PURGE_LOG()` 函数以进行日志删除操作的权限。

## **函数语法**

```
> PURGE_LOG('sys_table_name', 'date')
```

## **参数释义**

|  参数  | 说明 |
|  ----  | ----  |
| 'sys_table_name' | 当前可进行删除的系统表仅三个：metric，rawlog，statement_info。<br> __Note:__ 'sys_table_name' 必须用引号包裹。|
| 'date'  | 选择日期，删除该日期之前产生的日志。<br> __Note:__ 'date' 必须用单引号包裹。|

!!! note
    MatrixOne 有且仅有 metric，rawlog，statement_info 三张系统日志表，有关这三张表的详细信息请参考 [MatrixOne 系统数据库和表](../../System-tables.md)。

## **示例**

- 示例 1：

```sql
-- 删除 2023-06-30 这一天之前的 statement_info 类型的日志
mysql> select purge_log('statement_info', '2023-06-30') a;
+------+
| a    |
+------+
|    0 |
+------+
1 row in set (0.01 sec)
```

- 示例 2：

```sql
-- 查询 metric 日志采集的时间和数量
mysql> select date(collecttime), count(1) from system_metrics.metric group by date(collecttime);
+-------------------+----------+
| date(collecttime) | count(1) |
+-------------------+----------+
| 2023-07-07        |    20067 |
| 2023-07-06        |    30246 |
| 2023-07-05        |    27759 |
+-------------------+----------+
3 rows in set (0.04 sec)

-- 删除 2023-07-06 这一天之前的 rawlog，statement_info，和 metric 三种类型的日志
mysql> select purge_log('rawlog, statement_info, metric', '2023-07-06');
+-------------------------------------------------------+
| purge_log(rawlog, statement_info, metric, 2023-07-06) |
+-------------------------------------------------------+
|                                                     0 |
+-------------------------------------------------------+
1 row in set (0.33 sec)

-- 再次查询 2023-07-05，2023-07-06 和 2023-07-07 这三天的 metric 日志数量
mysql> select date(collecttime), count(1) from system_metrics.metric group by date(collecttime);
+-------------------+----------+
| date(collecttime) | count(1) |
+-------------------+----------+
| 2023-07-06        |    30246 |
| 2023-07-07        |    20121 |
+-------------------+----------+
2 rows in set (0.01 sec)
```
