# **CREATE ACCOUNT**

## **语法说明**

为其中一个集群成员创建一个新的租户。

## **语法结构**

```
> CREATE ACCOUNT  [IF NOT EXISTS]
account auth_option
[COMMENT 'comment_string']

auth_option: {
    ADMIN_NAME [=] 'admin_name'
    IDENTIFIED BY 'auth_string'
}
```

### 语法说明

#### auth_option

指定租户默认的帐号名和授权方式，`auth_string` 表示显式返回指定密码。

## **示例**

```sql
> create account tenant_test admin_name = 'root' identified by '111' comment 'tenant_test';
Query OK, 0 rows affected (0.08 sec)
```
