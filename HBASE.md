# HBASE
**1、HBASE介绍**

1.1 存储与访问
- 近实时，s级别
- 布隆过滤器-->位数组形式
- 跳表结构，建立多级索引
- 索引存储在hdfs上，包括顶级索引；zookeeper存储索引入口
- HDFS-->底层存储计算、MAPREDUCE-->数据计算、zookeeper-->服务协调

1.2 NOSQL
- not sql
- not only sql，会有工具将NOSQL数据的原生查询语句封装为SQL，如hbase的phoenix工具

1.3 HBASE特点
- 介于NOSQL与RDBMS之间，`仅能通过rowkey和rowkey进行range查询`
- HBASE查询数据简单，`不支持join等复杂操作`，可通过hive支持多表join
- 不支持复杂事务，`只支持行级事务`
- 存储支持的数据类型，`byte[]`
- 只要存储结构化结构化、半结构数据化松散数据

1.4 表的特点
- 数据量大
- 面向列-->面向列簇的存储和权限控制，列簇独立检索
- 稀疏-->对于null的列不占用存储空间
- 无模式-->没行都有一个rowkey和任意多的列，列可以动态增加，同一表中的不同行可能有截然不同的列（hive-->读模式,mysql-->写模式）

1.6 查询流程
- 表-->rowkey-->列簇-->列-->时间戳


**2、HBASE中相关概念**

2.1 rowkey行键
- 一行数据的标志
- rowkey不宜过大,最大64kb，通常10-100byte，16byte最佳，涉及存储问题（每个列簇的物理文件中都会存储rowkey）
- hbase存储默认按照rowkey字典顺序升序排列

2.2 列簇
- 一个或者多个列，通常一个表中列簇个数不超过3个
- 列簇是hbase存储的物理切分单位（面向列簇），一个文件中列簇一样
- 相同IO特性（通常会一起访问的列）的列划分到一个列簇
- 列簇越多，存储的文件个数越多，会增加扫描成本；所以通常实际生产过程中列簇为1个
- 建表的时候指定列簇，数据插入的时候指定列名、列值

2.3 时间戳
- 数据在插入的时候，每插入一条数据就会自动生成一个时间戳，时间戳用于记录数据的版本
- hbase可以存储多个版本的数据，多版本之间靠时间戳记录，同意rowkey可以多次插入
- 时间戳默认为系统时间，ms级别，也可以手动指定

2.4 单元格
- 某一行数据的某一列的某一个值
- 定位一个单元格需要：rowkey+列簇+时间戳

2.5 hbase三种查询方式
- 全表扫描
- 通过rowkey范围查询
- 查询单个rowkey数据

2.6 hbase存储
- 寻址路径-->zookeeper
- 主从Hmaster,Hregionserver
- region由regionserver管理，hbase中行方向的逻辑切分概念
- 数据增加，region行方向分裂
- 由中间分裂，便于建立索引
