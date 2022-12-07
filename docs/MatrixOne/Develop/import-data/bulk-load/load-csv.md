# 导入 *.csv* 格式数据

本篇文档将指导你在 MySQL 客户端启动 MatrixOne 时如何完成 *.csv* 格式数据导入。

## 开始前准备

已完成[单机部署 MatrixOne](../../../Get-Started/install-standalone-matrixone.md)。

!!! note
    如果你是通过 `docker` 安装的 MatrixOne，那么导出目录默认位于 docker 镜像中。如果你要需要挂载本地目录，参见下面的代码示例：本地文件系统路径 *${local_data_path}/mo-data* 挂载到 MatrixOne Docker 镜像中，并映射到 */mo-data* 路径下。更多信息，参见 [Docker Mount Volume tutorial](https://www.freecodecamp.org/news/docker-mount-volume-guide-how-to-mount-a-local-directory/)。

```
sudo docker run --name <name> --privileged -d -p 6001:6001 -v ${local_data_path}/mo-data:/mo-data:rw matrixorigin/matrixone:0.6.0
```

## MySQL Client 中使用 `Load data` 命令导入数据

在 MySQL Client 中使用 `Load data` 命令将外部大数据文件导入，但目前只支持 *.csv* 格式的文件导入。

1. 在 MatrixOne 中执行 `Load Data` 之前，需要提前在 MatrixOne 中创建完成数据表。目前，数据文件需要与 MatrixOne 服务器在同一台机器上，如果它们在不同的机器上，则需要进行文件传输。

    !!! note
        如果你使用 Docker 启动 MatrixOne，需要先将数据文件挂载到容器的目录中。

2. 在 MatrixOne 本地服务器中启动 MySQL 客户端以访问本地文件系统。

    ```
    mysql -h 127.0.0.1 -P 6001 -udump -p111
    ```

3. 在 MySQL 客户端对对应的文件路径执行 `LOAD DATA`：

    ```
    mysql> LOAD DATA INFILE '/tmp/xxx.csv'
    INTO TABLE xxxxxx
    FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' LINES TERMINATED BY "\r\n";
    ```

    !!! note
        如果你使用 Docker 启动 MatrixOne，需要先将这个数据文件路径挂载到容器的目录中。

### 示例：使用 *docker* 启动 MatrixOne 执行 `Load data`

本示例将介绍如何使用 Docker 启动 MatrixOne 0.6.0 加载数据。

1. 下载数据集，并且将数据集存储到本地 *~/tmp/docker_loaddata_demo/* 路径下：

    ```
    cd ~/tmp/docker_loaddata_demo/
    wget https://community-shared-data-1308875761.cos.ap-beijing.myqcloud.com/lineorder_flat.tar.bz2
    ```

2. 解压数据集：

    ```
    tar -jxvf lineorder_flat.tar.bz2
    ```

3. 使用 Docker 启动 MatrixOne，启动时将存放了数据文件的目录 *~/tmp/docker_loaddata_demo/* 挂载到容器的某个目录下，这里容器目录以 */ssb-dbgen-path* 为例：

    ```
    sudo docker run --name <name> --privileged -d -p 6001:6001 -v ~/tmp/docker_loaddata_demo/:/ssb-dbgen-path:rw matrixorigin/matrixone:0.6.0
    ```

4. 连接 MatrixOne 服务：

    ```
    mysql -h 127.0.0.1 -P 6001 -udump -p111
    ```

5. 在 MatrixOne 中新建表 *lineorder_flat*，并且将数据集导入至 MatriOne：

    ```
    mysql> create database if not exists ssb;
    mysql> use ssb;
    mysql> drop table if exists lineorder_flat;
    mysql> CREATE TABLE lineorder_flat(
      LO_ORDERKEY bigint key,
      LO_LINENUMBER int,
      LO_CUSTKEY int,
      LO_PARTKEY int,
      LO_SUPPKEY int,
      LO_ORDERDATE date,
      LO_ORDERPRIORITY char(15),
      LO_SHIPPRIORITY tinyint,
      LO_QUANTITY double,
      LO_EXTENDEDPRICE double,
      LO_ORDTOTALPRICE double,
      LO_DISCOUNT double,
      LO_REVENUE int unsigned,
      LO_SUPPLYCOST int unsigned,
      LO_TAX double,
      LO_COMMITDATE date,
      LO_SHIPMODE char(10),
      C_NAME varchar(25),
      C_ADDRESS varchar(25),
      C_CITY char(10),
      C_NATION char(15),
      C_REGION char(12),
      C_PHONE char(15),
      C_MKTSEGMENT char(10),
      S_NAME char(25),
      S_ADDRESS varchar(25),
      S_CITY char(10),
      S_NATION char(15),
      S_REGION char(12),
      S_PHONE char(15),
      P_NAME varchar(22),
      P_MFGR char(6),
      P_CATEGORY char(7),
      P_BRAND char(9),
      P_COLOR varchar(11),
      P_TYPE varchar(25),
      P_SIZE int,
      P_CONTAINER char(10)
    );
    mysql> load data infile '/ssb-dbgen-path/lineorder_flat.tbl' into table lineorder_flat FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' LINES TERMINATED BY '\n';
    ```

6. 导入成功后，可以使用 SQL 语句查看导入数据的行数：

    ```
    select count(*) from lineorder_flat;
    /*
        expected results:
     */
    +----------+
    | count(*) |
    +----------+
    | 10272594 |
    +----------+
    ```
