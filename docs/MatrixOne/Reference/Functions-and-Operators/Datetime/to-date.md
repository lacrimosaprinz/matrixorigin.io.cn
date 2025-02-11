# **TO_DATE()**

## **函数说明**

``TO_DATE()`` 函数按照指定日期或时间显示格式，将字符串转换为日期或日期时间类型。

格式字符串可以包含文字字符和以%开头的格式说明符。format 中的字面字符必须匹配 str 中的字面字符。format 中的格式说明符必须匹配 str 中的日期或时间部分。

## **函数语法**

```
> TO_DATE(str,format)
```

## **参数释义**

|  Arguments   | Description  |
|  ----  | ----  |
| str | Required.  <br>如果 ``str`` 为 ``NULL``，则函数返回 ``NULL``。 <br>如果从 ``str`` 中的 ``date`` 或 ``datetime`` 值不合法，则 ``STR_TO_DATE()`` 将返回 ``NULL`` 并产生警告。|
| format  | 可选参数。表示返回值格式的格式字符串。<br> 如果省略 format，则返回一个 ``DATETIME`` 值。 <br>如果 format 为空，则返回 ``NULL``。<br>如果 format 已存在指定格式，则返回值为 ``VARCHAR``。|

说明：格式字符串可以包含文字字符和以 *%* 开头的格式说明符。``format`` 中的字面字符必须匹配 ``str`` 中的字面字符。``format`` 中的格式说明符必须匹配 ``str`` 中的日期或时间部分。

## **示例**

```sql
mysql> SELECT TO_DATE('2022-01-06 10:20:30','%Y-%m-%d %H:%i:%s') as result;
+---------------------+
| result              |
+---------------------+
| 2022-01-06 10:20:30 |
+---------------------+
1 row in set (0.00 sec)                
```

## **限制**

目前 date 格式只支持 `yyyy-mm-dd` 和 `yyyymmdd` 的数据格式。  
