# **SHOW PUBLICATIONS**

## **语法说明**

返回所有 PUBLICATION 名称列表与对应数据库名。

如需查看更多信息，需要拥有租户管理员权限，查看系统表 mo_pubs 查看更多参数。

## **语法结构**

```
SHOW PUBLICATIONS;
```

## **示例**

```sql
create account acc0 admin_name 'root' identified by '111';
create account acc1 admin_name 'root' identified by '111';
create account acc2 admin_name 'root' identified by '111';
create database t;
create publication pub3 database t account acc0,acc1;

mysql> show publications;
+------+----------+
| Name | Database |
+------+----------+
| pub3 | t        |
+------+----------+
1 row in set (0.00 sec)
```
