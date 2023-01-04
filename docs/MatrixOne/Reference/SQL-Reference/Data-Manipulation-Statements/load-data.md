# **LOAD DATA**

## **概述**

`LOAD DATA` 语句可以极快地将文本文件中的行读入表中。你可以从服务器主机或 [S3 兼容对象存储](../../../Develop/import-data/bulk-load/load-s3.md) 读取该文件。`LOAD DATA` 是 [`SELECT ... INTO OUTFILE`](../../../Develop/export-data/select-into-outfile.md) 相反的操作。

- 将文件读回表中，使用 `LOAD DATA`。
- 将表中的数据写入文件，使用 `SELECT ... INTO OUTFILE`。
- `FIELDS` 和 `LINES` 子句的语法对于 `LOAD DATA` 和 `SELECT ... INTO OUTFILE` 这两个语句的使用方式一致，使用 Fields 和 Lines 参数来指定如何处理数据格式。

## **语法结构**

```
> LOAD DATA
    INFILE 'file_name'
    INTO TABLE tbl_name
    [{FIELDS | COLUMNS}
        [TERMINATED BY 'string']
        [[OPTIONALLY] ENCLOSED BY 'char']
        [ESCAPED BY 'char']
    ]
    [LINES
        [STARTING BY 'string']
        [TERMINATED BY 'string']
    ]
    [IGNORE number {LINES | ROWS}]
```

**参数解释**

上述语法结构中的参数解释如下：

### INFILE

MatrixOne 目前只支持加载服务器主机上的数据，`file_name` 必须是 MatrixOne 所在的服务器主机上存放文件的绝对路径名称。

### IGNORE LINES

`IGNORE number LINES` 子句可用于忽略文件开头的行。例如，你可以使用 `IGNORE 1 LINES` 跳过包含列名的初始标题行：

```
LOAD DATA INFILE '/tmp/test.txt' INTO TABLE table1 IGNORE 1 LINES;
```

### FIELDS 和 LINES 参数说明

使用 `FIELDS` 和 `LINES` 参数来指定如何处理数据格式。

对于 `LOAD DATA` 和 `SELECT ... INTO OUTFILE` 语句，`FIELDS` 和 `LINES` 子句的语法是相同的。这两个子句都是可选的，但如果两者都指定，则 `FIELDS` 必须在 `LINES` 之前。

如果指定 `FIELDS` 子句，那么 `FIELDS` 的每个子句（`TERMINATED BY`、`[OPTIONALLY] ENCLOSED BY` 和 `ESCAPED BY`）也是可选的，除非你必须至少指定其中一个。

`LOAD DATA` 也支持使用十六进制 `ASCII` 字符表达式或二进制 `ASCII` 字符表达式作为 `FIELDS ENCLOSED BY` 和 `FIELDS TERMINATED BY` 的参数。

如果不指定处理数据的参数，则使用默认值如下：

```
FIELDS TERMINATED BY ',' ENCLOSED BY '"' LINES TERMINATED BY '\n'
```

意义如下：

- `FIELDS TERMINATED BY ','`：以 , 作为分隔符
- `ENCLOSED BY '"'`：使用双引号把各个字段括起来
- `LINES TERMINATED BY '\n'`：以'\n'为行间分隔符

**FIELDS TERMINATED BY**

`FIELDS TERMINATED BY` 表示字段与字段之间的分隔符，使用 `FIELDS TERMINATED BY` 就可以指定每个数据的分隔符号。

`FIELDS TERMINATED BY` 指定的值可以超过一个字符。

- **正确示例**：

例如，读取使用*逗号*分隔的文件，语法是：

```
LOAD DATA INFILE 'data.txt' INTO TABLE table1
  FIELDS TERMINATED BY ',';
```

- **错误示例**：

如果你使用如下所示的语句读取文件，将会产生报错，因为它表示的是 `LOAD DATA` 查找字段之间的制表符：

```
LOAD DATA INFILE 'data.txt' INTO TABLE table1
  FIELDS TERMINATED BY '\t';
```

这样可能会导致结果被解释为每个输入行都是一个字段，你可能会遇到 `ERROR 20101 (HY000): internal error: the table column is larger than input data column` 错误。

**FIELDS ENCLOSED BY**

`FIELDS TERMINATED BY` 指定的值包含输入值的字符。 `ENCLOSED BY` 指定的值必须是单个字符；如果输入值不一定包含在引号中，需要在 `ENCLOSED BY` 选项之前使用 `OPTIONALLY`。

如下面的例子所示，即表示一部分输入值用可以用引号括起来，另一些可以不用引号括起来：

```
LOAD DATA INFILE 'data.txt' INTO TABLE table1
  FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"';
```

如果 `ENCLOSED BY` 前不加 `OPTIONALLY`，比如说，`ENCLOSED BY '"'` 就表示使用双引号把各个字段都括起来。

**FIELDS ESCAPED BY**

`FIELDS ESCAPED BY` 用于控制如何写入或读取特殊字符。`FIELDS ESCAPED BY` 值必须是单个字符。如果 `FIELDS ESCAPED BY` 字符不是空字符，则该字符将被去除，同时保留其后一个字符。

一些例外的双字符序列，其中第一个字符作为转义字符。

这些序列如下表所示（使用 \ 作为转义字符）。

|  字符   | 转义序列  |
|  ----  | ----  |
| \0 | 	反转义成 0（0x00） |
| \b | 	回车字符 |
| \n | 	换行(换行)字符  |
| \r | 	回车字符 |
| \t | 	Tab 字符 |
| \Z | 	ASCII 26 (Control+Z) |
| \N | 	NULL |

例如，如果某些输入值是特殊字符 “\”，则可以使用 `ESCAPE BY`：

```
LOAD DATA INFILE 'data.txt' INTO TABLE table1
  FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"' ESCAPED BY '\\';
```

如果 `FIELDS ESCAPED BY` 字符为空，则不会进行转义序列。

**LINES TERMINATED BY**

`LINES TERMINATED BY` 用于指定一行的分隔符。 `LINES TERMINATED BY` 值可以超过一个字符。

例如，如果 *csv* 文件中的行以回车符/换行符对结束，你在加载它时，可以使用 `LINES TERMINATED BY '\r\n'`：

```
LOAD DATA INFILE 'data.txt' INTO TABLE table1
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\r\n';
```

**LINE STARTING BY**

如果所有输入行都有一个你想忽略的公共前缀，你可以使用 `LINES STARTING BY` 'prefix_string' 来忽略前缀和前缀之前的任何内容。

如果一行不包含前缀，则跳过整行。如下语句所示：

```
LOAD DATA INFILE '/tmp/test.txt' INTO TABLE table1
  FIELDS TERMINATED BY ','  LINES STARTING BY 'xxx';
```

如果数据文件是如下样式：

```
xxx"abc",1
something xxx"def",2
"ghi",3
```

则输出的结果行是("abc"，1)和("def"，2)。文件中的第三行由于没有前缀，则被忽略。

## 支持的文件格式

在 MatrixOne 当前版本中，`LOAD DATA` 支持 *CSV* 格式和 *JSONLines* 格式文件。

有关导入这两种格式的文档，参见 [导入 *.csv* 格式数据](../../../Develop/import-data/bulk-load/load-csv.md) 和 [导入 JSONLines 数据](../../../Develop/import-data/bulk-load/load-jsonline.md)。

## **示例**

你可以在 SSB 测试中了解 `LOAD DATA` 语句的用法，参见[完成 SSB 测试](../../../Test/performance-testing/SSB-test-with-matrixone.md)。

语法示例如下：

```
> LOAD DATA INFILE '/ssb-dbgen-path/lineorder_flat.tbl ' INTO TABLE lineorder_flat;
```

上面这行语句表示：将 */ssb-dbgen-path/* 这个目录路径下的 *lineorder_flat.tbl* 数据集加载到 MatrixOne 的数据表 *lineorder_flat* 中。

你也可以参考以下语法示例，来快速了解 `LOAD DATA`：

### 示例 1：LOAD CSV

#### 简单导入示例

本地命名为 *char_varchar.csv* 文件内数据如下：

```
a|b|c|d
"a"|"b"|"c"|"d"
'a'|'b'|'c'|'d'
"'a'"|"'b'"|"'c'"|"'d'"
"aa|aa"|"bb|bb"|"cc|cc"|"dd|dd"
"aa|"|"bb|"|"cc|"|"dd|"
"aa|||aa"|"bb|||bb"|"cc|||cc"|"dd|||dd"
"aa'|'||aa"|"bb'|'||bb"|"cc'|'||cc"|"dd'|'||dd"
aa"aa|bb"bb|cc"cc|dd"dd
"aa"aa"|"bb"bb"|"cc"cc"|"dd"dd"
"aa""aa"|"bb""bb"|"cc""cc"|"dd""dd"
"aa"""aa"|"bb"""bb"|"cc"""cc"|"dd"""dd"
"aa""""aa"|"bb""""bb"|"cc""""cc"|"dd""""dd"
"aa""|aa"|"bb""|bb"|"cc""|cc"|"dd""|dd"
"aa""""|aa"|"bb""""|bb"|"cc""""|cc"|"dd""""|dd"
|||
||||
""|""|""|
""""|""""|""""|""""
""""""|""""""|""""""|""""""
```

在 MatrixOne 中建表：

```sql
mysql> drop table if exists t1;
Query OK, 0 rows affected (0.01 sec)

mysql> create table t1(
    -> col1 char(225),
    -> col2 varchar(225),
    -> col3 text,
    -> col4 varchar(225)
    -> );
Query OK, 0 rows affected (0.02 sec)
```

将数据文件导入到 MatrixOne 中的表 t1：

```sql
load data infile '<your-local-file-path>/char_varchar.csv' into table t1 fields terminated by'|';
```

查询结果如下：

```
mysql> select * from t1;
+-----------+-----------+-----------+-----------+
| col1      | col2      | col3      | col4      |
+-----------+-----------+-----------+-----------+
| a         | b         | c         | d         |
| a         | b         | c         | d         |
| 'a'       | 'b'       | 'c'       | 'd'       |
| 'a'       | 'b'       | 'c'       | 'd'       |
| aa|aa     | bb|bb     | cc|cc     | dd|dd     |
| aa|       | bb|       | cc|       | dd|       |
| aa|||aa   | bb|||bb   | cc|||cc   | dd|||dd   |
| aa'|'||aa | bb'|'||bb | cc'|'||cc | dd'|'||dd |
| aa"aa     | bb"bb     | cc"cc     | dd"dd     |
| aa"aa     | bb"bb     | cc"cc     | dd"dd     |
| aa"aa     | bb"bb     | cc"cc     | dd"dd     |
| aa""aa    | bb""bb    | cc""cc    | dd""dd    |
| aa""aa    | bb""bb    | cc""cc    | dd""dd    |
| aa"|aa    | bb"|bb    | cc"|cc    | dd"|dd    |
| aa""|aa   | bb""|bb   | cc""|cc   | dd""|dd   |
|           |           |           |           |
|           |           |           |           |
|           |           |           |           |
| "         | "         | "         | "         |
| ""        | ""        | ""        | ""        |
+-----------+-----------+-----------+-----------+
20 rows in set (0.00 sec)
```

### 增加条件导入示例

沿用上面的简单示例，你可以修改一下 LOAD DATA 语句，在末尾增加条件 `LINES STARTING BY 'aa' ignore 10 lines;`：

```sql
delete from t1;
load data infile '<your-local-file-path>/char_varchar.csv' into table t1 fields terminated by'|' LINES STARTING BY 'aa' ignore 10 lines;
```

查询结果如下：

```sql
mysql> select * from t1;
+---------+---------+---------+---------+
| col1    | col2    | col3    | col4    |
+---------+---------+---------+---------+
| aa"aa   | bb"bb   | cc"cc   | dd"dd   |
| aa""aa  | bb""bb  | cc""cc  | dd""dd  |
| aa""aa  | bb""bb  | cc""cc  | dd""dd  |
| aa"|aa  | bb"|bb  | cc"|cc  | dd"|dd  |
| aa""|aa | bb""|bb | cc""|cc | dd""|dd |
|         |         |         |         |
|         |         |         |         |
|         |         |         |         |
| "       | "       | "       | "       |
| ""      | ""      | ""      | ""      |
+---------+---------+---------+---------+
10 rows in set (0.00 sec)
```

可以看到，查询结果忽略了前 10 行，并且忽略了公共前缀 aa。

有关如何导入 *CSV* 格式文件的详细步骤，参见[导入 *.csv* 格式数据](../../../Develop/import-data/bulk-load/load-jsonline.md)。

### 示例 2：LOAD JSONLines

#### 简单导入示例

本地命名为 *jsonline_array.jl* 文件内数据如下：

```
[true,1,"var","2020-09-07","2020-09-07 00:00:00","2020-09-07 00:00:00","18",121.11,["1",2,null,false,true,{"q":1}],"1qaz",null,null]
["true","1","var","2020-09-07","2020-09-07 00:00:00","2020-09-07 00:00:00","18","121.11",{"c":1,"b":["a","b",{"q":4}]},"1aza",null,null]
```

在 MatrixOne 中建表：

```sql
mysql> drop table if exists t1;
Query OK, 0 rows affected (0.01 sec)

mysql> create table t1(col1 bool,col2 int,col3 varchar(100), col4 date,col5 datetime,col6 timestamp,col7 decimal,col8 float,col9 json,col10 text,col11 json,col12 bool);
Query OK, 0 rows affected (0.03 sec)
```

将数据文件导入到 MatrixOne 中的表 t1：

```
load data infile {'filepath'='<your-local-file-path>/jsonline_array.jl','format'='jsonline','jsondata'='array'} into table t1;
```

查询结果如下：

```sql
mysql> select * from t1;
+------+------+------+------------+---------------------+---------------------+------+--------+---------------------------------------+-------+-------+-------+
| col1 | col2 | col3 | col4       | col5                | col6                | col7 | col8   | col9                                  | col10 | col11 | col12 |
+------+------+------+------------+---------------------+---------------------+------+--------+---------------------------------------+-------+-------+-------+
| true |    1 | var  | 2020-09-07 | 2020-09-07 00:00:00 | 2020-09-07 00:00:00 |   18 | 121.11 | ["1", 2, null, false, true, {"q": 1}] | 1qaz  | NULL  | NULL  |
| true |    1 | var  | 2020-09-07 | 2020-09-07 00:00:00 | 2020-09-07 00:00:00 |   18 | 121.11 | {"b": ["a", "b", {"q": 4}], "c": 1}   | 1aza  | NULL  | NULL  |
+------+------+------+------------+---------------------+---------------------+------+--------+---------------------------------------+-------+-------+-------+
2 rows in set (0.00 sec)
```

#### 增加条件导入示例

沿用上面的简单示例，你可以修改一下 LOAD DATA 语句，增加 `ignore 1 lines` 在语句的末尾，体验一下区别：

```
delete from t1;
load data infile {'filepath'='<your-local-file-path>/jsonline_array.jl','format'='jsonline','jsondata'='array'} into table t1 ignore 1 lines;
```

查询结果如下：

```sql
mysql> select * from t1;
+------+------+------+------------+---------------------+---------------------+------+--------+-------------------------------------+-------+-------+-------+
| col1 | col2 | col3 | col4       | col5                | col6                | col7 | col8   | col9                                | col10 | col11 | col12 |
+------+------+------+------------+---------------------+---------------------+------+--------+-------------------------------------+-------+-------+-------+
| true |    1 | var  | 2020-09-07 | 2020-09-07 00:00:00 | 2020-09-07 00:00:00 |   18 | 121.11 | {"b": ["a", "b", {"q": 4}], "c": 1} | 1aza  | NULL  | NULL  |
+------+------+------+------------+---------------------+---------------------+------+--------+-------------------------------------+-------+-------+-------+
1 row in set (0.00 sec)
```

可以看到，查询结果忽略掉了第一行。

有关如何导入 *JSONLines* 格式文件的详细步骤，参见[导入 JSONLines 数据](../../../Develop/import-data/bulk-load/load-jsonline.md)。

## **限制**

1. `LOAD DATA` 暂不支持在客户端主机加载文件，仅支持在服务器主机加载文件。
2. `REPLACE` 和 `IGNORE` 修饰符解决唯一索引的冲突：`REPLACE` 表示若表中已经存在则用新的数据替换掉旧的数据，而 `IGNORE` 则表示保留旧的数据，忽略掉新数据。这两个修饰符在 MatrixOne 中尚不支持。
3. 当前只支持绝对文件路径。
4. `SET` 提供不是来源于输入文件的值，MatrixOne 当前部分支持 `SET`，仅支持 `SET columns_name=nullif(expr1,expr2)`。
