# DolphinDB ParquetWin Plugin

ParquetWin 是专门为 Windows 环境优化的 Apache Parquet 文件交互插件。它支持将 Parquet 文件高效导入 DolphinDB 内存表或分布式数据库（DFS），并支持将 DolphinDB 表导出为标准 Parquet 格式。

主要特性包括：
*   **高性能 I/O**：利用 Parquet 的列式存储特性，支持多线程并发读取，极大地提升了海量数据的加载速度。
*   **灵活的 Schema 映射**：支持在加载时通过索引映射重命名列名，并进行数据类型强制转换（如 TIMESTAMP 转 DATE）。
*   **分布式集成**：提供 loadParquetEx 接口，支持将数据直接流式写入 DolphinDB 分区表，并能在加载过程中执行自定义转换函数（Transform）。
*   **时间精度自动对齐**：智能识别 Parquet 文件的多种时间单位（Millis, Micros, Nanos），自动与 DolphinDB 对应的存储精度对齐，确保跨系统数据交换时的时间戳准确无误。
*   **MapReduce 兼容**：通过 parquetDS 函数生成数据源，支持与 DolphinDB 的分布式计算框架无缝对接。


---

## 在插件市场安装插件
### 版本要求
- DolphinDB Server 3.00.4 及更高版本。
- 支持 Windows x64。
### 安装步骤
1. 在 DolphinDB 客户端中使用 `listRemotePlugins` 命令查看插件仓库中的插件信息。
```
login("admin", "123456")
listRemotePlugins()
```

2. 使用 `installPlugin` 完成安装。
```
installPlugin("ParquetWin")
```
3. 使用 `loadPlugin` 加载插件。
```
loadPlugin("ParquetWin")
```
## 接口说明
所有函数均定义在 `ParquetWin` 命名空间下。

## 用户接口

### extractParquetSchema

**语法**

```
ParquetWin::extractParquetSchema(fileName)
```

**参数**

**fileName** Parquet 文件名，类型为字符串标量。

**详情**

获取 Parquet 文件的结构，返回一个包含两列的表：name (列名) 和 type (数据类型) 。

**例子**

```
scm = ParquetWin::extractParquetSchema("D:/data/sample.parquet")
```


### loadParquet

**语法**

```
ParquetWin::loadParquet(fileName,[schema],[columnsToLoad],[startRowGroup],[rowGroupNum])
```

**参数**

**fileName** Parquet 文件名，类型为字符串标量。

**schema** 可选参数，必须是包含列名和列数据类型的表。通过设置该参数，可改变系统自动生成的列数据类型。

**columnsToLoad** 可选参数，是一个整数向量（索引从 0 开始），表示要读取的列索引。若不指定，读取所有列。

**startRowGroup** 可选参数，是一个非负整数，从哪一个 row group 开始读取 Parquet 文件。若不指定，默认从文件起始位置读取。

**rowGroupNum** 可选参数，是一个是正整数，要读取 row group 的数量。若不指定，默认读到文件的结尾。

**详情**

将 Parquet 文件数据加载为 DolphinDB 数据库的内存表。关于 Parquet 数据类型及在 DolphinDB 中的转化规则，参见下文支持的数据类型章节。

**例子**

```
// 加载 parquet 文件中的所有数据
ParquetWin::loadParquet("userdata1.parquet")

// 加载特定列并将第 6 列转换为 DOUBLE
scm = ParquetWin::extractParquetSchema("spark.parquet")
update scm set type="DOUBLE" where name=`volume
re = ParquetWin::loadParquet("spark.parquet", scm, [0, 1, 6])

```

### loadParquetEx

**语法**

```
ParquetWin::loadParquetEx(dbHandle,tableName,partitionColumns,fileName,[schema],[columnsToLoad],[startRowGroup],[rowGroupNum],[transform])
```

**参数**

**dbHandle** 数据库句柄。

**tableName** 一个字符串，表示表的名称。

**partitionColumns** 字符串标量或向量，表示分区列。在组合分区中，该参数是字符串向量。

**fileName** Parquet 文件名，类型为字符串标量。

**schema** 可选参数，必须是包含列名和列数据类型的表。通过设置该参数，可改变系统自动生成的列数据类型。

**columnsToLoad** 可选参数，整数向量，表示读取的列索引。若不指定，读取所有列。

**startRowGroup** 可选参数，是一个非负整数，从哪一个 row group 开始读取 Parquet 文件。若不指定，默认从文件起始位置读取。

**rowGroupNum** 可选参数，是一个是正整数，要读取 row group 的数量。若不指定，默认读到文件的结尾。

**transform** 可选参数，为一元函数，且该函数接受的参数必须是一个表。如果指定了 *transform* 参数，若分区表存在，程序会对数据文件中的数据执行 *transform* 参数指定的函数，再将得到的结果保存到分区表中。若分区表不存在，加载数据后先执行此函数（如聚合、清洗），然后根据函数返回的表结构创建分区表，再将数据存入分区表。

**详情**

将 Parquet 文件数据加载到 DolphinDB 数据库的分区表，返回该表的元数据。 

- 如果要将数据文件加载到分布式数据库或本地磁盘数据库中，必须指定 *dbHandle*，并且不能为空字符串。
- 如果要将数据文件加载到内存数据库中，那么 *dbHandle* 为空字符串或者不指定 *dbHandle*。

关于 Parquet 数据类型及在 DolphinDB 中的转化规则，参见下文支持的数据类型章节。

**例子**

- dfs 分区表

  ```
  db = database("dfs://rangedb", RANGE, 0 500 1000)
  ParquetWin::loadParquetEx(db,`tb,`id,"userdata1.parquet")
  ```

- 分区内存表

  ```
  db = database("", RANGE, 0 500 1000)
  ParquetWin::loadParquetEx(db,`tb,`id,"userdata1.parquet")
  ```

- 指定参数 *transform*，将数值类型表示的日期和时间（如：20200101）转化为指定类型（比如：日期类型）

  ```
  dbPath="dfs://DolphinDBdatabase"
  db=database(dbPath,VALUE,2020.01.01..2020.01.30)
  dataFilePath="level.parquet"
  schemaTB=ParquetWin::extractParquetSchema(dataFilePath)
  update schemaTB set type="DATE" where name="date"
  tb=table(1:0,schemaTB.name,schemaTB.type)
  tb1=db.createPartitionedTable(tb,`tb1,`date);
  def i2d(mutable t){
      return t.replaceColumn!(`date,datetimeParse(t.date),"yyyy.MM.dd"))
  }
  t = ParquetWin::loadParquetEx(db,`tb1,`date,dataFilePath,,,,i2d)
  ```

### parquetDS

**语法**

```
ParquetWin::parquetDS(fileName,[schema])
```

**参数**

**fileName** Parquet 文件名，类型为字符串标量。

**schema** 可选参数，必须是包含列名和列数据类型的表。通过设置该参数，可改变系统自动生成的列数据类型。

**详情**

根据输入的 Parquet 文件名创建数据源列表，生成的数据源数量等价于 row group 的数量。

**例子**

```
ds = ParquetWin::parquetDS("userdata1.parquet")
size ds;
//Output: 1
ds[0];
//Output:DataSource< loadParquet("userdata1.parquet",,,0,1) >
```

### saveParquet

**语法**

```
ParquetWin::saveParquet(table, fileName, [compressMethod])
```

**参数**

**table** 要保存的表。

**fileName** 保存的文件名，类型为字符串标量

**compressMethod** 压缩格式。类型为字符串标量。支持 snappy, gzip, zstd，默认为不压缩。

**详情**

将 DolphinDB 中的表以 Parquet 格式保存到文件中。

**例子**

```
ParquetWin::saveParquet(tb, "userdata1.parquet")

t = table(1..100 as id, rand(100.0, 100) as val)
ParquetWin::saveParquet(t, "D:/output.parquet", "snappy")
```


## 支持的数据类型

### 导入

DolphinDB 在导入 Parquet 数据时，优先按照源文件中定义的 LogicalType 转换相应的数据类型。如果没有定义 LogicalType 或 ConvertedType，则只根据原始数据类型（physical type）转换。目前版本还不支持将 Parquet 文件的 repeated 类型读取为 DolphinDB 的 array vector。

| Logical Type in Parquet                  | TimeUnit in Parquet | Type in DolphinDB                     |
| ---------------------------------------- | ------------------- | ------------------------------------- |
| INT(bit_width=8,is_signed=true)          | \\                   | CHAR                                  |
| INT(bit_width=8,is_signed=false or bit_width=16,is_signed=true) | \\                   | SHORT                                 |
| INT(bit_width=16,is_signed=false or bit_width=32,is_signed=true) | \\                   | INT                                   |
| INT(bit_width=32,is_signed=false or bit_width=64,is_signed=true) | \\                   | LONG                                  |
| INT(bit_width=64,is_signed=false)        | \\                   | LONG                                  |
| ENUM                                     | \\                   | SYMBOL                                |
| DECIMAL                                  | \\                   | DOUBLE                                |
| DATE                                     | \\                   | DATE                                  |
| TIME                                     | MILLIS\MICROS\NANOS | TIME\NANOTIME\NANOTIME                |
| TIMESTAMP                                | MILLIS\MICROS\NANOS | TIMESTAMP\NANOTIMESTAMP\NANOTIMESTAMP |
| INTEGER                                  | \\                   | INT\LONG                              |
| STRING                                   | \\                   | STRING                                |
| JSON                                     | \\                   | not support                           |
| BSON                                     | \\                   | not support                           |
| UUID                                     | \\                   | not support                           |
| MAP                                      | \\                   | not support                           |
| LIST                                     | \\                   | not support                           |
| NIL                                      | \\                   | not support                           |

| Converted Type in Parquet | Type in DolphinDB |
| ------------------------- | ----------------- |
| INT_8                     | CHAR              |
| UINT_8\INT_16             | SHORT             |
| UINT_16\INT_32            | INT               |
| TIMESTAMP_MICROS          | NANOTIMESTAMP     |
| TIMESTAMP_MILLIS          | TIMESTAMP         |
| DECIMAL                   | DOUBLE            |
| UINT_32\INT_64\UINT_64    | LONG              |
| TIME_MICROS               | NANOTIME          |
| TIME_MILLIS               | TIME              |
| DATE                      | DATE              |
| ENUM                      | SYMBOL            |
| UTF8                      | STRING            |
| MAP                       | not support       |
| LIST                      | not support       |
| JSON                      | not support       |
| BSON                      | not support       |
| MAP_KEY_VALUE             | not support       |

| Physical Type in Parquet | Type in DolphinDB |
| ------------------------ | ----------------- |
| BOOLEAN                  | BOOL              |
| INT32                    | INT               |
| INT64                    | LONG              |
| INT96                    | NANOTIMESTAMP     |
| FLOAT                    | FLOAT             |
| DOUBLE                   | DOUBLE            |
| BYTE_ARRAY               | STRING            |
| FIXED_LEN_BYTE_ARRAY     | STRING            |

**注意：**

- 在 Parquet 中标注了 DECIMAL 类型的字段中，仅支持转化原始数据类型（physical type）为 INT32, INT64 和 FIXED_LEN_BYTE_ARRAY 的数据。
- 由于 DolphinDB 不支持无符号类型，所以读取 Parquet 中的 UINT_64 时若发生溢出，则会取 DolphinDB 中的 NULL 值。
- STRING 转 数值/时间：支持通过指定 Schema 将 Parquet 的字符串/二进制列导入为 DolphinDB 的数值或时间类型。插件会尝试解析字符串内容（如 "10.0" 转为 10）。
- STRING 转 CHAR：若目标类型设为 CHAR，插件会取字符串的第一个有效字符。
- 数值转 BOOL：非 0 值均会被转换为 true。

### 导出

将 DolphinDB 数据导出为 Parquet 文件时，系统根据给出表的结构自动转换到 Parquet 文件支持的类型。目前版本还不支持将 DolphinDB 的 array vector 转换为 Parquet 的 repeated 类型。

| Type in DolphinDB | Physical Type in Parquet | Logical Type in Parquet |
| ----------------- | ------------------------ | ----------------------- |
| BOOL              | BOOLEAN                  | \\                       |
| CHAR              | FIXED_LEN_BYTE_ARRAY     | \\                       |
| SHORT             | INT32                    | INT(16)                 |
| INT               | INT32                    | INT(32)                 |
| LONG              | INT64                    | INT(64)                 |
| DATE              | INT32                    | DATE                    |
| MONTH             | INT32                    | DATE                    |
| TIME              | INT32                    | TIME_MILLIS             |
| MINUTE            | INT32                    | TIME_MILLIS             |
| SECOND            | INT32                    | TIME_MILLIS             |
| DATETIME          | INT64                    | TIMESTAMP_MILLIS        |
| TIMESTAMP         | INT64                    | TIMESTAMP_MILLIS        |
| NANOTIME          | INT64                    | TIME_NANOS              |
| NANOTIMESTAMP     | INT64                    | TIMESTAMP_NANOS         |
| FLOAT             | FLOAT                    | \\                       |
| DOUBLE            | DOUBLE                   | \\                       |
| STRING            | BYTE_ARRAY               | STRING                  |
| SYMBOL            | BYTE_ARRAY               | STRING                  |




