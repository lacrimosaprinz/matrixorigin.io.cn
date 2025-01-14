# 发布订阅

数据库的发布订阅（Publish-Subscribe，简称 Pub/Sub）是一种消息传递模式，其中**发布者**将消息发送给一个或多个**订阅者**，而**订阅者**则接收并处理该消息。在这种模式下，发布者和订阅者之间是松耦合的，它们之间不需要直接通信，因此可以提高应用程序的可扩展性和灵活性。

在数据库中，发布订阅功能通常被用于实时数据更新、缓存同步、业务事件通知等场景。例如，当数据库中某个表的数据发生变化时，可以通过发布订阅功能实时通知订阅者，从而实现实时数据同步和处理。另外，也可以通过发布订阅功能来实现业务事件的通知，例如某个订单被取消、某个库存数量不足等等。

通常，数据库的发布订阅功能由两部分组成：**发布者**和**订阅者**。**发布者**负责发布消息，而**订阅者**则订阅相应消息以达到数据同步的目的。发布者和订阅者之间可以存在多对多的关系，即一个发布者可以向多个订阅者发布消息，而一个订阅者也可以订阅多个消息/数据。

## 应用场景

发布订阅功能具有多种典型的应用场景：

- **数据同步**：当一个数据库需要与另一个数据库保持同步时，发布订阅功能可以用来将数据更改发送到订阅者数据库。例如，当一个网站需要将数据从一个地理位置传输到另一个地理位置时，发布订阅功能可以用来确保两个数据库之间的数据同步。

- **业务数据分发**：发布订阅功能可以用来将业务数据分发到不同的系统或业务流程中。例如，当一个银行需要将客户账户信息分发到多个业务系统中时，发布订阅功能可以用来将数据分发到相应的系统中，确保各个业务流程之间的数据一致性。

- **数据备份**：发布订阅功能可以用来备份数据。例如，当一个数据库需要备份到另一个数据库时，发布订阅功能可以用来将数据备份到订阅者数据库中，以便在主数据库出现故障时恢复数据。

- **实时数据处理**：发布订阅功能可以用来实现实时数据处理。例如，当一个网站需要对来自不同用户的数据进行处理时，发布订阅功能可以用来将数据传输到处理程序中进行处理，以便实现实时数据分析和决策。

<!--- 通知和提醒：发布订阅功能可以用来向不同的用户或系统发送通知和提醒。例如，当一个电子商务网站需要通知顾客有新产品上线时，发布订阅功能可以用来将新产品信息发布到顾客的订阅列表中，以便顾客及时了解到新产品信息。-->

## 名词解释

- **发布**：在数据库中，发布通常指的是将一个数据库对象设置为可供其他租户访问的状态。这是数据共享和复制的一个重要步骤，发布的对象可以被其他租户订阅并获取数据。

- **订阅**：订阅是指一个数据库选择接收和复制发布的数据库对象的数据。

- **发布端（Pub）**：发布端是执行发布操作的数据库。发布端负责创建和管理发布的对象，以及管理订阅该发布对象的数据库的访问权限。

- **订阅端（Sub）**：订阅端是订阅发布对象的租户。

- **发布对象**：发布对象是在发布端创建并设置为可发布的数据库对象，即数据库。这些对象的数据可以被订阅端访问和复制。

- **订阅对象**：订阅对象是在订阅端复制和存储的发布对象。订阅对象的数据会根据发布端的数据进行更新。

## 发布订阅范围说明

### 发布/订阅应用范围

**发布端（Pub）**和**订阅端（Sub）**均为 MatrixOne 的租户。

### 可发布/可订阅权限范围

- **发布端（Pub）**只有 ACCOUNTADMIN 或 MOADMIN 角色可以创建发布与订阅。
- **订阅端（Sub）**由 ACCOUNTADMIN 或 MOADMIN 角色操作访问订阅数据权限。

### 发布/订阅数据范围

- 一个**发布**只能与单一数据库关联。
- 发布和订阅只在数据库级别实现，目前还不支持直接进行表级别的发布和订阅。
- **订阅端**对**订阅库**只具备读取权限。

- 若**发布端（Pub）**调整了发布的分享范围，那些不在新范围内的**订阅端（Sub）**如果已经创建了订阅库，那么对这个**订阅库**的访问将无效。
- 若**发布端（Pub）**尝试删除已经发布的数据库，那么此次删除将不会成功。
- 若**发布端（Pub）**删除了**发布**，但订阅库中的对应对象仍存在，此时**订阅端（Sub）**访问这个对象会触发错误，需要由**订阅端（Sub）**删除对应的**订阅**。
- 若**发布端（Pub）**删除了**发布对象**，但在订阅库中的对应对象仍然存在，此时**订阅端（Sub）**访问这个对象会触发错误，需要由**订阅端（Sub）**删除对应的**订阅对象**。

### 发布订阅示例

![](https://community-shared-data-1308875761.cos.ap-beijing.myqcloud.com/artwork/docs/develop/pub-sub/example-zh.png)

本章节将给出一个示例，介绍当前在 MatrixOne 集群中，存在 3 个租户，sys、acc1 与 acc2，按照操作顺序对三个租户进行操作：

1. **发布者**：sys 租户创建数据库 sub1 与表 t1，并发布 pub1：

    ```sql
    create database sub1;
    create table sub1.t1(a int,b int);
    create publication pub1 database sub;
    ```

2. **订阅者**：acc1 和 acc2 都创建订阅库 syssub1，于是得到共享的数据表 t1：

    ```sql
    -- acc1 和 acc2 创建订阅库的 sql 语句一致，此处不做赘述
    create database syssub1 from sys publication pub1;
    use syssub1;
    show tables;
    mysql> show tables;
    +--------------------+
    | Tables_in_syssub1  |
    +--------------------+
    | t1                 |
    +--------------------+
    2 rows in set (0.02 sec)
    ```

3. **发布者**：sys 租户创建数据表 t2：

    ```sql
    create table sub1.t2(a text);
    ```

4. **订阅者**：acc1 和 acc2 得到共享的数据表 t1 和 t2：

    ```sql
    show tables;
    +--------------------+
    | Tables_in_syssub1  |
    +--------------------+
    | t1                 |
    +--------------------+
    | t2                 |
    +--------------------+
    2 rows in set (0.02 sec)
    ```

5. **发布者**：sys 租户创建数据库 sub2 与表 t2，并发布 pub2 给租户 acc1 和 acc3：

    ```sql
    create database sub2;
    create table sub2.t1(a float);
    create publication pub2 database sub2 account acc1,acc3;
    ```

6. **订阅者**：acc1 和 acc2 都创建订阅库 syssub2，acc1 得到共享的数据表 t1；acc2 创建订阅库 syssub2 失败：

    - acc1

    ```sql
    create database syssub2 from sys publication pub2;
    use syssub2;
    mysql> show tables;
    +--------------------+
    | Tables_in_syssub2  |
    +--------------------+
    | t1                 |
    +--------------------+
    2 rows in set (0.02 sec)
    ```

    - acc2

    ```sql
    create database syssub2 from sys publication pub2;
    > ERROR 20101 (HY000): internal error: the account acc3 is not allowed to subscribe the publication pub2
    ```

7. **发布者**：sys 租户修改发布 pub2 给全部租户：

    ```sql
    alter publication pub2 account all;
    ```

8. **订阅者**：acc2 创建订阅库 syssub2 成功，得到共享的数据表 t1：

    ```sql
    create database syssub2 from sys publication pub2;
    use syssub2;
    mysql> show tables;
    +--------------------+
    | Tables_in_syssub2  |
    +--------------------+
    | t1                 |
    +--------------------+
    2 rows in set (0.02 sec)
    ```

9. **发布者**：sys 租户删除发布 pub1：

    ```sql
    drop publication pub1;
    ```

10. **订阅者**：acc1 连接 syspub1 失败：

     ```sql
     use syssub1;
     ERROR 20101 (HY000): internal error: there is no publication pub1
     ```

11. **订阅者**：acc2 删除 syspub1：

     ```sql
     drop database syssub1;
     ```

12. **发布者**：sys 租户重新创建 pub1：

     ```sql
     create publication pub1 database sub;
     ```

13. **订阅者**：acc1 连接 syspub1 成功：

     ```sql
     create database syssub1 from sys publication pub1;
     use syssub1;
     mysql> show tables;
     +--------------------+
     | Tables_in_syssub1  |
     +--------------------+
     | t1                 |
     +--------------------+
     2 rows in set (0.02 sec)
     ```

## 参考文档

### 发布者参考文档

- [CREATE PUBLICATION](../../Reference/SQL-Reference/Data-Definition-Language/create-publication.md)
- [ALTER PUBLICATION](../../Reference/SQL-Reference/Data-Definition-Language/alter-publication.md)
- [DROP PUBLICATION](../../Reference/SQL-Reference/Data-Definition-Language/drop-publication.md)
- [SHOW PUBLICATIONS](../../Reference/SQL-Reference/Other/SHOW-Statements/show-publications.md)
- [SHOW CREATE PUBLICATION](../../Reference/SQL-Reference/Other/SHOW-Statements/show-create-publication.md)

### 订阅者参考文档

- [CREATE...FROM...PUBLICATION...](../../Reference/SQL-Reference/Data-Definition-Language/create-subscription.md)
- [SHOW SUBSCRIPTIONS](../../Reference/SQL-Reference/Other/SHOW-Statements/show-subscriptions.md)
