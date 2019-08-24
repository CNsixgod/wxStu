---
title: mysql优化
data:
categories:
- mysql
---



# mysql优化

## msyql优化方向
  + 存储层 存储引擎 列类型 范式
  + 设计层 索引 缓存 分表
  + sql层 效率高的sql语句
  + 架构层 分布式数据库架构 主从同步

###  存储引擎

1. **myisam**
- 物理文件
  + .frm    结构文件
  + .MYD    数据文件
  + .MYI    索引文件
- 存储顺序
  + 存储顺序为插入顺序  没有排序
- 复制备份，压缩机制
  + 移动三个物理文件即可备份恢复
  + 压缩： 节省空间，查询快速(表压缩之后无法进行写入操作,压缩表机制，适合读比较多的场景使用)
    - Mysql\bin  myisamchk.exe -- 重建索引及解压缩
    - myisampack.exe --压缩
      - cmd > D:\phpstudy\Mysql\bin\myisampack.exe D:\phpstudy\Mysql\data\student
    - myisamchk.exe --重建索引
      - cmd > D:\phpstudy\Mysql\bin\myisamchk.exe -rq D:\phpstudy\Mysql\data\student
    - myisamchk.exe --解压缩(压缩操作需要进行表的索引重建，但是解压不需要了，会自动重建。)
      - cmd > D:\phpstudy\Mysql\bin\myisamchk.exe --unpack D:\phpstudy\Mysql\data\student
- 并发性： 表级锁（并发写入）
  + 并发性稍弱，只有表锁 [相关链接](https://www.cnblogs.com/qq78292959/archive/2013/01/30/2883109.html)



2. **innodb**
- 物理文件
  + .frm    结构文件
  + 数据库的所有innodb数据和索引文件都放在data/ibdata1文件中
- 存储顺序
  + 存储顺序为数据主键顺序
- 复制备份
  + 需要导出sql文件，再导入进行备份和恢复操作
  + 所以需要进行innodb表数据结构的分开存储操作
    - mysql > show variables like 'innodb_file_per_table'; //查看innodb存储状态 value off 关闭
    - mysql > set global innodb_file_per_table=1;  //临时修改值为开启 value on 开启  建表后数据索引文件就分离 ibdata1了 重启后建表数据还是在一起，但是分开后的就不会再连一起了 xxx.ibd文件--数据和索引文件

- 并发性： 擅长并发
  + 更适合并发写入，并发写中，为了防止数据不一致 使用锁机制
  + 有更小的锁表粒度,行级别

3. **如何选择myisam innodb**
  + 根据引擎的特点进行选择
  + 功能性上，快速备份恢复myisam；事务和外键，选择innodb
  + 一般使用myisam，cms（内容管理系统）
  + innodb适合使用在抢购功能，订单功能，并发写入较多

### 字段选取
  - age 用tinyint  乌龟年龄 smallint 表数据不超过1600w+  使用mediumint
  - 内容长度
    + char(255) 固定字符 不足null补齐 varchar(65535) 如果是UTF则最大65535/3 - 1个字符，因为要预留空间存放该字段的字符数目)
    + 加密密码 手机号 char
    + 邮箱 varchar
    + char查询的效率比varchar高
  - 整型存储
    + 尽量使用整型存储其他数据的类型。节省空间提高查询速度
      + 时间戳
        + 使用int来存储时间。4字节
          +  mysql里的时间戳函数
          +  unix_timestamp()  当前时间戳信息
          +  from_unixtime()  读取一个时间戳信息
          +  php用date()转换
          +  int类型 时间的比较直接比大小就行了
        + 使用int存储IP 计算范围更加方便
          + mysql:  inet_aton(ip) &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  inet_ntoa(int)
          + php: ip2long(ip)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;long2ip(ip)


### 范式
1. **第一范式** 列类型具有原子性，每列(或者每个属性)都是不可再分的最小数据单元(也称为最小的原子单元)
2. **第二范式**  数据具有唯一性 设置主键
3. **第三范式** 数据字段和主键具有紧密联系/直接关系，不能够有冗（rong）余[重复]数据
---
1NF<2NF<3NF，范式是一层层满足的

#### 逆范式
&nbsp;&nbsp;一般情况下，设计的表结构符合三范式，被认为是良好的数据机构设计。
但是，在很多实际业务场景下，为了能够提高数据表的查询速度，要建立使用冗（rong）余字段。也就不符合第三范式了。为了整 第三范式。
之前的查询学生的所有信息（包含系别信息），需要进行连表操作。连表操作，遍历最大数为笛卡尔积（相乘）。可以修改表机构为如下，建立冗余字段。
- 建立冗余字段之后，查询变快。在修改和维护数据时，要同时维护冗余字段。保持数据的一致性

## 索引

### 索引优化
