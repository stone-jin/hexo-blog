---
title: Influx时序数据库
date: 2019-05-07 23:15:00
subtitle: InfluxDB 是用Go语言编写的一个开源分布式时序、事件和指标数据库，无需外部依赖。
cover: https://520stone-blog.oss-cn-beijing.aliyuncs.com/blog_fedfans/WX20190508-143057.png
author:
  nick: 金炳
  link: https://www.github.com/stone-jin
categories:
  - 数据库
  - Influx
tags:
  - 数据库
  - Influx
---

# InfluxDB--时序数据库

<a name="xaMqU"></a>
## 一、介绍
InfluxDB 是用Go语言编写的一个开源分布式时序、事件和指标数据库，无需外部依赖。

类似的数据库有Elasticsearch、Graphite等。<br />**其主要特色功能**

1. 基于时间序列，支持与时间有关的相关函数（如最大，最小，求和等）
1. 可度量性：你可以实时对大量数据进行计算
1. 基于事件：它支持任意的事件数据

**InfluxDB的主要特点**

1. 无结构（无模式）：可以是任意数量的列
1. 可拓展的
1. 支持min, max, sum, count, mean, median 等一系列函数，方便统计
1. 原生的HTTP支持，内置HTTP API
1. 强大的类SQL语法
<a name="gCDFr"></a>
### 1.1 概念
<a name="SySGA"></a>
#### 1.1.1 与传统数据库的名词做比较
![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557135823326-4ba86192-d5dc-42c1-aaad-2cf86c04e575.png#align=left&display=inline&height=187&name=image.png&originHeight=187&originWidth=654&size=18068&status=done&width=654)
<a name="GGjEM"></a>
#### 1.1.2 influx独有的概念
**Point**<br />Point由时间戳（time）、数据（field）、标签（tags）组成。<br />Point相当于传统数据库里的一行数据，如下表所示：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557135982313-63f36773-d447-4eda-88bf-f8a7366c0a36.png#align=left&display=inline&height=235&name=image.png&originHeight=235&originWidth=627&size=28123&status=done&width=627)<br />influxdb 的时间精度是纳秒。这个跟prometheus不一样，prometheus是毫秒。这点也注意一下。<br />此处是重点，就是我们插入一条数据，使用的是:

```
insert disk_free,xxx=1,yyy=2 value=3,age=18
```
其中xxx和yyy是tags，而value和age是fields。<br />对于我们要检索的条件来说，我们要符合以下结构：

```
select value,xxx from disk_free where xxx=1
```
select的第一个要是fields的名字，第二个才能是对应tags的名字，而且后续条件那块要以tags，虽然后面where的条件也能对fields进行检索。<br />所以下面这种情况是不对的:

```
select xxx from disk_free
```
但是往往我们是根据对应的tag进行group化：<br />所以我们会用下面这种情况

```
select value,yyy from disk_free group by xxx
```
这样我们如果使用到grafana就变成了两个折线图。<br />**series**<br />所有在数据库中的数据，都需要通过图表来展示，而这个series表示这个表里面的数据，可以在图表上画成几条线：通过tags排列组合算出来。<br />我们可以通过下面命令来查看某一个表的series

```
show series from xxx
```
或者查看整个database的series

```
show series
```

<a name="h0hER"></a>
## 二、安装
<a name="fk2Px"></a>
### 2.1 docker方式安装
[https://hub.docker.com/_/influxdb](https://hub.docker.com/_/influxdb)<br />安装：<br />docker run -itd -p 8086:8086 influxdb<br />然后我们进入容器内部，如下:<br />docker exec -it ${containerId} bash<br />然后执行influx这个客户端，可以连接InfluxDB的server端进行操作。
<a name="e5fBX"></a>
### 2.2 普通方式安装

```bash
OS X (via Homebrew)
brew update
brew install influxdb
MD5: 4f0aa76fee22cf4c18e2a0779ba4f462

Ubuntu & Debian (64-bit)

wget https://dl.influxdata.com/influxdb/releases/influxdb_0.13.0_amd64.deb
sudo dpkg -i influxdb_0.13.0_amd64.deb
MD5: bcca4c91bbd8e7f60e4a8325be67a08a

Ubuntu & Debian (ARM)

wget https://dl.influxdata.com/influxdb/releases/influxdb_0.13.0_armhf.deb
sudo dpkg -i influxdb_0.13.0_armhf.deb
MD5: b64ada82b6abf5d6382ed08dde1e8579

RedHat & CentOS (64-bit)
wget https://dl.influxdata.com/influxdb/releases/influxdb-0.13.0.x86_64.rpm
sudo yum localinstall influxdb-0.13.0.x86_64.rpm
MD5: 286b6c18aa4ef37225ea6605a729b61d

RedHat & CentOS (ARM)
wget https://dl.influxdata.com/influxdb/releases/influxdb-0.13.0.armhf.rpm
sudo yum localinstall influxdb-0.13.0.armhf.rpm
MD5: 4cf99debb5315fbbb26166506807d965

Standalone Binaries (64-bit)
wget https://dl.influxdata.com/influxdb/releases/influxdb-0.13.0_linux_amd64.tar.gz
tar xvfz influxdb-0.13.0_linux_amd64.tar.gz
MD5: 187854536393c67f7793ada1c096da8e

Standalone Binaries (ARM)
wget https://dl.influxdata.com/influxdb/releases/influxdb-0.13.0_linux_armhf.tar.gz
tar xvfz influxdb-0.13.0_linux_armhf.tar.gz

Docker Image
docker pull influxdb
```
<a name="yhRXE"></a>
### 2.3 路径说明

```
/etc/influxdb/influxdb.conf 默认的配置文件
/var/log/influxdb/influxd.log 日志文件
/var/lib/influxdb/data 数据文件
/usr/lib/influxdb/scripts 初始化脚本文件夹
/usr/bin/influx 启动数据库
```

<a name="wLgHq"></a>
## 三、基本操作
<a name="Mnz9b"></a>
### 3.1 使用客户端influx
以docker的为例，我们前面已经安装了对应的influx，然后我们登录容器内部

```
docker exec -it ${containerID} bash 
```
然后进来后，我们执行influx --version可以查看对应的版本。<br />使用influx可以用这个客户端连接InfluxDB。
<a name="mEe4t"></a>
### 3.2 数据库操作
<a name="i4LpV"></a>
#### 3.2.1 查看数据库
通过以下命令，我们可以查看一共有哪些databases。
```
show databases
```

<a name="Jrk4Z"></a>
#### 3.2.2 新增数据库
我们通过create database操作来新建一个数据库。
```
create database test
```

<a name="6m3Ar"></a>
#### 3.2.3 删除数据库
我们通过drop database操作来删除一个数据库。

```
drop database test
```

<a name="B2oAN"></a>
#### 3.2.4 使用数据库
我们可以通过use database来使用一个数据库

```
use database
```

<a name="7O8Jw"></a>
### 3.3 measurement操作
<a name="WjyUZ"></a>
#### 3.3.1 查看measurement
查看measurement其实相当于我们原来的查看有多少table一样。<br />执行的语句是：

```
show measurements
```

<a name="vzUzt"></a>
#### 3.3.2 新增measurement
没有显示的创建measurement的操作，我们只要插入了对应的数据就会创建对应的measurement

```
insert disk_free,hostname=server01 value=442221834240i
```
最后的时间，不需要我们进行插入，这个是会自己生成的。
<a name="pAvhv"></a>
#### 3.3.3 删除measurement

```
 drop measurement disk_free
```
<a name="MiCeF"></a>
#### 3.3.4 查询measurement的数据
如下我们来查询disk_free这个measurement的数据

```
select * from disk_free
```

<a name="u8GEM"></a>
### 3.4 新增数据
<a name="WIrhU"></a>
#### 3.4.1 新增数据
如下，进行增加数据，如果measurement没有存在，则会创建这个measurement。
```
insert disk_free,hostname=server01 value=442221834240i
```
时序数据库的特点，决定没有删除和修改操作，所以这块并没有这两块的说明。<br />增加数据采用insert的方式，要注意的是 InfluxDB的insert中，表名与数据之间用逗号（,）分隔，tag和field之间用 空格分隔，多个tag或者多个field之间用逗号（,）分隔。<br />在上面那条语句中，disk_free是表名,hostname=server01是tag，属于索引，value=xx是field，这个可以随意写，随意定义。
<a name="jcMPO"></a>
#### 3.4.2 数据查询
此处是重点，就是我们插入一条数据，使用的是:

```
insert disk_free,xxx=1,yyy=2 value=3,age=18
```
其中xxx和yyy是tags，而value和age是fields。<br />**情况一：**<br />对于我们要检索的条件来说，我们要符合以下结构：

```
select value,xxx from disk_free where xxx=1
```
select的第一个要是fields的名字，第二个才能是对应tags的名字，而且后续条件那块要以tags，虽然后面where的条件也能对fields进行检索。<br />所以下面这种情况是不对的:

```
select xxx from disk_free
```
**情况二：**<br />但是往往我们是根据对应的tag进行group化：<br />所以我们会用下面这种情况

```
select value,yyy from disk_free group by xxx
```
这样我们如果使用到grafana就变成了两个折线图。<br />**情况三：**<br />有时候group by xxx的时候，我们需要剔除掉某些不符合数据规范的数据，就可以用下面这个方法：

```
select value from disk_free where xxx!='' group by xxx
```
这样就会把那些xxx为空的数据给剔除掉了，这样我们的折线图就去掉了某些干扰项。<br />**情况四：**<br />有时候，我们要制作饼图，这时候其实是按tag进行比例的划分，所以这时候的语句应该是如下的：

```
select count(value) from disk_free group by xxx
```
那这个时候，其实也能剔除掉对应的一些垃圾数据，类似于情况三的操作方式。<br />**情况五**<br />有时候我们只要展示xxx的某几种情况，不仅仅是情况三那种干扰某一个，比如我们需要包含几项。这时候我们会用正则进行匹配。

```
select value from disk_free where xxx =~ /^(111|222)$/ group by xxx
```
这时候，111|222，我们可以通过grafana的变量来进行解决，所以上面的语句最终变成了：

```
select value from disk_free where xxx =~ /^$xxx$/ group by xxx
```
上面的$xxx会最终变成(111|222)，而/^$|则是一个正则。
<a name="Zj5fq"></a>
### 3.5 连续查询
InfluxDB的连续查询是在数据库中自动定时启动的一组语句，语句中必须包含SELECT关键词和GROUP BY time()关键词。<br />InfluxDB会将查询结果放到指定的数据表中。

**目的：**<br />使用连续查询是最优的降低采样率的方式，连续查询和存储策略搭配使用将会大大降低InfluxDB的系统占用量。<br />而且使用连续查询后，数据会存放到指定的数据库中，这样就为以后统计不同京都的数据提供了方便。
<a name="JD49G"></a>
#### 3.5.1 新增连续查询

```bash
CREATE CONTINUOUS QUERY <cq_name> ON <database_name> 
[RESAMPLE [EVERY <interval>] [FOR <interval>]] 
BEGIN SELECT <function>(<stuff>)[,<function>(<stuff>)] INTO <different_measurement> 
FROM <current_measurement> [WHERE <stuff>] GROUP BY time(<interval>)[,<stuff>] 
END
```
示例：

```bash
CREATE CONTINUOUS QUERY cq_30m ON telegraf BEGIN SELECT mean(used) INTO mem_used_30m FROM mem GROUP BY time(30m) END
```

<a name="21GVL"></a>
#### 3.5.2 删除连续查询

```
DROP CONTINUOUS QUERY <cq_name> ON <database_name>
```

<a name="SVuLR"></a>
#### 3.5.3 查询连续查询

```bash
SHOW CONTINUOUS QUERIES
```

<a name="nijqw"></a>
### 3.6 InfluxDB常用函数使用
<a name="Av8sn"></a>
#### 3.6.1 聚合类函数

- **count()函数**

返回一个（field）字段中的非空值的数量。

```
SELECT COUNT(<field_key>) FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

示例：

```
>SELECT COUNT(water_level) FROM h2o_feet
name: h2o_feet
--------------
time                           count
1970-01-01T00:00:00Z     15258
```

如果想要指定时间:<br />示例：

```
> SELECT COUNT(water_level) FROM h2o_feet WHERE time >= '2015-08-18T00:00:00Z' AND time < '2015-09-18T17:00:00Z' GROUP BY time(4d)
name: h2o_feet
--------------
time                           count
2015-08-17T00:00:00Z     1440
2015-08-21T00:00:00Z     1920
2015-08-25T00:00:00Z     1920
2015-08-29T00:00:00Z     1920
2015-09-02T00:00:00Z     1915
2015-09-06T00:00:00Z     1920
2015-09-10T00:00:00Z     1920
2015-09-14T00:00:00Z     1920
2015-09-18T00:00:00Z     335
```

这样结果是按照时间划分的。
- **Distinct()函数**

返回一个字段（field）的唯一值。<br />语法：

```
SELECT DISTINCT(<field_key>) FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

使用示例：

```
> SELECT DISTINCT("level description") FROM h2o_feet
name: h2o_feet
--------------
time                           distinct
1970-01-01T00:00:00Z     between 6 and 9 feet
1970-01-01T00:00:00Z     below 3 feet
1970-01-01T00:00:00Z     between 3 and 6 feet
1970-01-01T00:00:00Z     at or greater than 9 feet
```
这个例子显示level description这个字段共有四个值，然后将其显示了出来，时间为默认时间。

- **MEAN()函数**

返回一个字段（field）中的值的算术平均值（平均值）。字段类型必须是长整型或float64。<br />语法：

```
SELECT MEAN(<field_key>) FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

示例：

```
> SELECT MEAN(water_level) FROM h2o_feet
name: h2o_feet
--------------
time                           mean
1970-01-01T00:00:00Z     4.286791371454075
```

说明water_level字段的平均值为`4.286791371454075`
- **MEDIAN()函数**

从单个字段（field）中的排序值返回中间值（中位数）。字段值的类型必须是长整型或float64格式。<br />语法：

```
SELECT MEDIAN(<field_key>) FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

使用示例:

```
> SELECT MEDIAN(water_level) from h2o_feet
name: h2o_feet
--------------
time                           median
1970-01-01T00:00:00Z     4.124
```

说明表中water_level字段的中位数是4.124
- **SPREEAD()函数**

返回字段的最小值和最大值之间的差值。数据的类型必须是长整型或float64。<br />语法：

```
SELECT SPREAD(<field_key>) FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

使用示例：

```
> SELECT SPREAD(water_level) FROM h2o_feet
name: h2o_feet
--------------
time                            spread
1970-01-01T00:00:00Z      10.574
```

- **SUM()函数**

返回一个字段中的所有值的和。字段的类型必须是长整型或float64。<br />语法：

```
SELECT SUM(<field_key>) FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

使用示例：

```
> SELECT SUM(water_level) FROM h2o_feet
name: h2o_feet
--------------
time                           sum
1970-01-01T00:00:00Z     67777.66900000002
```
此语句计算出了 h2o_feet表中 所有 water_level 字段的和。
<a name="ZyrXM"></a>
#### 3.6.2 选择类函数

- **TOP()函数**

作用：返回一个字段中最大的N个值，字段类型必须是长整型或float64类型。<br />语法：

```
SELECT TOP( <field_key>[,<tag_key(s)>],<N> )[,<tag_key(s)>|<field_key(s)>] [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause]
```
使用示例：

```
> SELECT TOP("water_level",3) FROM "h2o_feet"

name: h2o_feet
time                   top
----                   ---
2015-08-29T07:18:00Z   9.957
2015-08-29T07:24:00Z   9.964
2015-08-29T07:30:00Z   9.954
```
这个示例返回表中water_level字段中最大的三个值。

- **BUTTOM()函数**

作用：返回一个字段中最小的N个值。字段类型必须是长整型或float64类型。<br />语法：

```
SELECT BOTTOM(<field_key>[,<tag_keys>],<N>)[,<tag_keys>] FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```
使用示例:

```
> SELECT BOTTOM(water_level,3) FROM h2o_feet
name: h2o_feet
--------------
time                           bottom
2015-08-29T14:30:00Z     -0.61
2015-08-29T14:36:00Z     -0.591
2015-08-30T15:18:00Z     -0.594
```
这个例子返回表中water_level字段中最小的三个值。<br />也可将关联的tag放在一起查询，但如果tag值少于N的值，则返回的值的个数只会取tag中字段值少的那个。<br />例如：

```
> SELECT BOTTOM(water_level,location,3) FROM h2o_feet
name: h2o_feet
--------------
time                           bottom     location
2015-08-29T10:36:00Z     -0.243     santa_monica
2015-08-29T14:30:00Z     -0.61      coyote_creek
```
语句取最小的三个值，然而结果只返回了2个值，因为location这个tag只有两个取值。

- **First()函数**

作用：返回一个字段中最老的取值。<br />语法：

```
SELECT FIRST(<field_key>)[,<tag_key(s)>] FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```
示例：

```
> SELECT FIRST(water_level) FROM h2o_feet WHERE location = 'santa_monica'
name: h2o_feet
--------------
time                           first
2015-08-18T00:00:00Z     2.064
```
这个语句返回了在location='santa_monica'条件下，最旧的那个water_level的取值和时间。

- **Last()函数**

作用：返回一个字段中最新的取值。<br />语法：

```
SELECT LAST(<field_key>)[,<tag_key(s)>] FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```
示例:

```
> SELECT LAST(water_level),location FROM h2o_feet WHERE time >= '2015-08-18T00:42:00Z' and time <= '2015-08-18T00:54:00Z'
name: h2o_feet
--------------
time                           last      location
2015-08-18T00:54:00Z     6.982     coyote_creek
```

- **MAX()函数**

作用：返回一个字段中的最大值。该字段类型必须是长整型，float64，或布尔类型。<br />语法：

```
SELECT MAX(<field_key>)[,<tag_key(s)>] FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```
示例:

```
> SELECT MAX(water_level),location FROM h2o_feet
name: h2o_feet
--------------
time                           max       location
2015-08-29T07:24:00Z     9.964     coyote_creek
```

- **MIN 函数**

作用：返回一个字段中的最小值。该字段类型必须是长整型，float64，或布尔类型。<br />语法：

```
SELECT MIN(<field_key>)[,<tag_key(s)>] FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```

示例:

```
> SELECT MIN(water_level),location FROM h2o_feet
name: h2o_feet
--------------
time                          min       location
2015-08-29T14:30:00Z    -0.61     coyote_creek
```

- **PRECENTILE()函数**

作用：返回排序值排位为N的百分值。字段的类型必须是长整型或float64。<br />百分值是介于100到0之间的整数或浮点数，包括100。

语法：

```
SELECT PERCENTILE(<field_key>, <N>)[,<tag_key(s)>] FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```
示例:

```
> SELECT PERCENTILE(water_level,5),location FROM h2o_feet
name: h2o_feet
--------------
time                      percentile     location
2015-08-28T12:06:00Z      1.122             santa_monica
```
就是将water_level字段按照不同的location求百分比，然后取第五位数据。
<a name="sUi2Z"></a>
#### 3.6.2 变换类函数

- **DERIVATIVE()函数**

作用：返回一个字段在一个series中的变化率。<br />InfluxDB会计算按照时间进行排序的字段值之间的[差异](https://www.linuxdaxue.com/tag/%e5%b7%ae%e5%bc%82/)，并将这些结果转化为单位变化率。其中，单位可以指定，默认为1s。<br />语法：

```
SELECT DERIVATIVE(<field_key>, [<unit>]) FROM <measurement_name> [WHERE <stuff>]
```
其中unit取值可以为以下几种：

```
u --microseconds
s --seconds
m --minutes
h --hours
d --days
w --weeks
```
DERIVATIVE()函数还可以在GROUP BY time()的条件下与聚合函数嵌套使用，格式如下：

```
SELECT DERIVATIVE(AGGREGATION_FUNCTION(<field_key>),[<unit>]) FROM <measurement_name> WHERE <stuff> GROUP BY time(<aggregation_interval>)
```
示例：<br />假设location = santa_monica 条件下数据有以下几条：

```
name: h2o_feet
--------------
time                           water_level
2015-08-18T00:00:00Z     2.064
2015-08-18T00:06:00Z     2.116
2015-08-18T00:12:00Z     2.028
2015-08-18T00:18:00Z     2.126
2015-08-18T00:24:00Z     2.041
2015-08-18T00:30:00Z     2.051
```
计算每一秒的变化率

```
> SELECT DERIVATIVE(water_level) FROM h2o_feet WHERE location = 'santa_monica' LIMIT 5
name: h2o_feet
--------------
time                           derivative
2015-08-18T00:06:00Z     0.00014444444444444457
2015-08-18T00:12:00Z     -0.00024444444444444465
2015-08-18T00:18:00Z     0.0002722222222222218
2015-08-18T00:24:00Z     -0.000236111111111111
2015-08-18T00:30:00Z     2.777777777777842e-05
```
第一行数据的计算公式为`(2.116 - 2.064) / (360s / 1s)`<br />计算每六分钟的变化率

```
> SELECT DERIVATIVE(water_level,6m) FROM h2o_feet WHERE location = 'santa_monica' LIMIT 5
name: h2o_feet
--------------
time                           derivative
2015-08-18T00:06:00Z     0.052000000000000046
2015-08-18T00:12:00Z     -0.08800000000000008
2015-08-18T00:18:00Z     0.09799999999999986
2015-08-18T00:24:00Z     -0.08499999999999996
2015-08-18T00:30:00Z     0.010000000000000231
```
第一行数据的计算过程如下：`(2.116 - 2.064) / (6m / 6m)`<br />计算每12分钟的变化率：

```
> SELECT DERIVATIVE(water_level,12m) FROM h2o_feet WHERE location = 'santa_monica' LIMIT 5
name: h2o_feet
--------------
time                           derivative
2015-08-18T00:06:00Z     0.10400000000000009
2015-08-18T00:12:00Z     -0.17600000000000016
2015-08-18T00:18:00Z     0.19599999999999973
2015-08-18T00:24:00Z     -0.16999999999999993
2015-08-18T00:30:00Z     0.020000000000000462
```
第一行数据计算过程为：`(2.116 - 2.064 / (6m / 12m)`<br />计算每12分钟最大值的变化率

```
> SELECT DERIVATIVE(MAX(water_level)) FROM h2o_feet WHERE location = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time < '2015-08-18T00:36:00Z' GROUP BY time(12m)
name: h2o_feet
--------------
time                           derivative
2015-08-18T00:12:00Z     0.009999999999999787
2015-08-18T00:24:00Z     -0.07499999999999973
```
这个函数功能非常多，也非常复杂，更多对于此功能的详细解释请看官网：https://docs.influxdata.com/influxdb/v0.13/query_language/functions/#derivative

- **DIFFERENCE()函数**

作用：返回一个字段中连续的时间值之间的[差异](https://www.linuxdaxue.com/tag/%e5%b7%ae%e5%bc%82/)。字段类型必须是长整型或float64。<br />最基本的语法：

```
SELECT DIFFERENCE(<field_key>) FROM <measurement_name> [WHERE <stuff>]
```
与GROUP BY time()以及其他嵌套函数一起使用的语法格式：

```
SELECT DIFFERENCE(<function>(<field_key>)) FROM <measurement_name> WHERE <stuff> GROUP BY time(<time_interval>)
```
其中，函数可以包含以下几个：<br />`COUNT()`, `MEAN()`, `MEDIAN()`,`SUM()`, `FIRST()`, `LAST()`, `MIN()`, `MAX()`, 和 `PERCENTILE()。`<br />使用示例<br />例子中使用的源数据如下所示：

```
> SELECT water_level FROM h2o_feet WHERE location='santa_monica' AND time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:36:00Z'
name: h2o_feet
--------------
time                            water_level
2015-08-18T00:00:00Z      2.064
2015-08-18T00:06:00Z      2.116
2015-08-18T00:12:00Z      2.028
2015-08-18T00:18:00Z      2.126
2015-08-18T00:24:00Z      2.041
2015-08-18T00:30:00Z      2.051
2015-08-18T00:36:00Z      2.067
```
计算water_level间的差异：

```
> SELECT DIFFERENCE(water_level) FROM h2o_feet WHERE location='santa_monica' AND time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:36:00Z'
name: h2o_feet
--------------
time                            difference
2015-08-18T00:06:00Z      0.052000000000000046
2015-08-18T00:12:00Z      -0.08800000000000008
2015-08-18T00:18:00Z      0.09799999999999986
2015-08-18T00:24:00Z      -0.08499999999999996
2015-08-18T00:30:00Z      0.010000000000000231
2015-08-18T00:36:00Z      0.016000000000000014
数据类型都为float类型。
```

- **ELAPSED()函数**

作用：返回一个字段在连续的时间间隔间的差异，间隔单位可选，默认为1纳秒。<br />单位可选项如下图：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557153024523-ad0d6616-acce-405a-84ec-5ba91e0f02ee.png#align=left&display=inline&height=372&name=image.png&originHeight=744&originWidth=468&size=60540&status=done&width=234)<br />语法：

```
SELECT ELAPSED(<field_key>, <unit>) FROM <measurement_name> [WHERE <stuff>]
```
示例：<br />计算h2o_feet字段在纳秒间隔下的差异。
```
> SELECT ELAPSED(water_level) FROM h2o_feet WHERE location = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:24:00Z'
name: h2o_feet
--------------
time                            elapsed
2015-08-18T00:06:00Z      360000000000
2015-08-18T00:12:00Z      360000000000
2015-08-18T00:18:00Z      360000000000
2015-08-18T00:24:00Z      360000000000
```
在一分钟间隔下的差异率

```
> SELECT ELAPSED(water_level,1m) FROM h2o_feet WHERE location = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:24:00Z'
name: h2o_feet
--------------
time                            elapsed
2015-08-18T00:06:00Z      6
2015-08-18T00:12:00Z      6
2015-08-18T00:18:00Z      6
2015-08-18T00:24:00Z      6
```
注意：如果设置的时间间隔比字段数据间的时间间隔更大时，则函数会返回0，如下所示：

```
> SELECT ELAPSED(water_level,1h) FROM h2o_feet WHERE location = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:24:00Z'
name: h2o_feet
--------------
time                            elapsed
2015-08-18T00:06:00Z      0
2015-08-18T00:12:00Z      0
2015-08-18T00:18:00Z      0
2015-08-18T00:24:00Z      0
```

- **MOVING_AVERAGE()函数**

作用：返回一个连续字段值的移动平均值，字段类型必须是长整形或者float64类型。<br />语法：<br />基本语法

```
SELECT MOVING_AVERAGE(<field_key>,<window>) FROM <measurement_name> [WHERE <stuff>]
```
与其他函数和GROUP BY time()语句一起使用时的语法

```
SELECT MOVING_AVERAGE(<function>(<field_key>),<window>) FROM <measurement_name> WHERE <stuff> GROUP BY time(<time_interval>)
```
此函数可以和以下函数一起使用：<br />COUNT(), MEAN(),MEDIAN(), SUM(), FIRST(), LAST(), MIN(), MAX(), and PERCENTILE().<br />示例：

```
> SELECT water_level FROM h2o_feet WHERE location = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' and time <= '2015-08-18T00:36:00Z'
name: h2o_feet
--------------
time                            water_level
2015-08-18T00:00:00Z      2.064
2015-08-18T00:06:00Z      2.116
2015-08-18T00:12:00Z      2.028
2015-08-18T00:18:00Z      2.126
2015-08-18T00:24:00Z      2.041
2015-08-18T00:30:00Z      2.051
2015-08-18T00:36:00Z      2.067

```

- **NON_NEGATIVE_DERIVATIVE()函数**

作用：返回在一个series中的一个字段中值的变化的非负速率。<br />语法：

```
SELECT NON_NEGATIVE_DERIVATIVE(<field_key>, [<unit>]) FROM <measurement_name> [WHERE <stuff>]
```
其中unit取值可以为以下几个：<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557153263507-1161b8ba-ea9a-4c34-a8bd-940beda416e3.png#align=left&display=inline&height=186&name=image.png&originHeight=372&originWidth=492&size=50988&status=done&width=246)<br />与聚合函数一块使用的语法：

```
SELECT NON_NEGATIVE_DERIVATIVE(AGGREGATION_FUNCTION(<field_key>),[<unit>]) FROM <measurement_name> WHERE <stuff> GROUP BY time(<aggregation_interval>)
```

- **STDDEV()函数**

作用：返回一个字段中的值的标准偏差。值的类型必须是长整型或float64类型。<br />语法：

```
SELECT STDDEV(<field_key>) FROM <measurement_name> [WHERE <stuff>] [GROUP BY <stuff>]
```
示例：

```
> SELECT STDDEV(water_level) FROM h2o_feet
name: h2o_feet
--------------
time                           stddev
1970-01-01T00:00:00Z     2.279144584196145
```

示例2：

```
> SELECT STDDEV(water_level) FROM h2o_feet WHERE time >= '2015-08-18T00:00:00Z' and time < '2015-09-18T12:06:00Z' GROUP BY time(1w), location
name: h2o_feet
tags: location = coyote_creek
time                           stddev
----                           ------
2015-08-13T00:00:00Z     2.2437263080193985
2015-08-20T00:00:00Z     2.121276150144719
2015-08-27T00:00:00Z     3.0416122170786215
2015-09-03T00:00:00Z     2.5348065025435207
2015-09-10T00:00:00Z     2.584003954882673
2015-09-17T00:00:00Z     2.2587514836274414

name: h2o_feet
tags: location = santa_monica
time                           stddev
----                           ------
2015-08-13T00:00:00Z     1.11156344587553
2015-08-20T00:00:00Z     1.0909849279082366
2015-08-27T00:00:00Z     1.9870116180096962
2015-09-03T00:00:00Z     1.3516778450902067
2015-09-10T00:00:00Z     1.4960573811500588
2015-09-17T00:00:00Z     1.075701669442093
```


<a name="hDCT3"></a>
## 四、API库操作
<a name="tMJrZ"></a>
### 4.1 安装node的API包
通过influx包，操作InfluxDB。<br />地址：[https://www.npmjs.com/package/influx](https://www.npmjs.com/package/influx)
<a name="P5UVO"></a>
### 4.2 新增database

```javascript
const influx = new Influx.InfluxDB({host: 'localhost'});
await influx.createDatabase('helloworld');
```

<a name="MlISg"></a>
### 4.3 删除database

```javascript
const influx = new Influx.InfluxDB({host: 'localhost'});
await influx.dropDatabase('helloworld');
```
<a name="gNP8B"></a>
### 4.4 判断是否存在database
下面的代码，判断如果不存在则取创建这个database
```javascript
const influx = new Influx.InfluxDB({host: 'localhost'});
if((await influx.getDatabaseNames()).filter(value=>{
		return value === 'helloworld';
}).length === 0){
		await influx.createDatabase('helloworld');
}
```
<a name="0XF6f"></a>
### 4.5 插入measurement数据

```javascript
const influx2 = new Influx.InfluxDB({host: 'localhost', database: 'helloworld'});
await influx2.writeMeasurement('hello', [{
  tags: {host: 'box1.example.com'},
  fields: {cpu: 123, mem: 234}
}])
```
<a name="BTpKH"></a>
### 4.6 查询measurement数据
```javascript
const rows = await influx2.query(`select * from response_times where host=${Influx.escape.stringLit(os.hostname())} order by time desc limit 10`)
rows.forEach(row => console.log(`A request to ${row.cpu} took ${row.mem}ms`))
```

<a name="rkSWO"></a>
## 五、数据备份和恢复
InfluxDB提供了数据的备份和恢复方法，在实际工作中，可以通过这些方法来实现数据的高可用。
<a name="fL7ql"></a>
### 5.1 数据备份
<a name="ynxzh"></a>
#### 5.1.1 备份元数据
influxDB本地备份元数据的语法如下，这只会备份InfluxDB的的internal库数据，包含那些最基本的系统信息、用户信息等。

```bash
influxd backup <path-to-backup>
```
例如：

```bash
influxd backup /tmp/backup
2016/02/01 17:15:03 backing up metastore to /tmp/backup/meta.00
2016/02/01 17:15:03 backup complete
```
<a name="K7ui4"></a>
#### 5.1.2 备份数据库
可以通过 -database 参数来指定备份的数据库。<br />语法：

```bash
influxd backup -database <mydatabase> <path-to-backup>
```
其他可选的参数:

```bash
-retention <retention policy name>
-shard <shard ID>
-since <date>
```
示例：

```bash
influxd backup -database telegraf -retention autogen -since 2016-02-01T00:00:00Z /tmp/backup
2016/02/01 18:02:36 backing up rp=default since 2016-02-01 00:00:00 +0000 UTC
2016/02/01 18:02:36 backing up metastore to /tmp/backup/meta.01
2016/02/01 18:02:36 backing up db=telegraf rp=default shard=2 to /tmp/backup/telegraf.default.00002.01 since 2016-02-01 00:00:00 +0000 UTC
2016/02/01 18:02:36 backup complete
```

<a name="iCZR2"></a>
#### 5.1.4 远程备份
InfluxDB可以使用 -host 参数实现数据的远程备份，端口一般是8088

```bash
 influxd backup -database mydatabase -host 10.0.0.1:8088 /tmp/mysnapshot
```

<a name="J7Mcz"></a>
### 5.2 数据恢复
语法：

```
influxd restore [ -metadir | -datadir ] <path-to-meta-or-data-directory> <path-to-backup>
```
必要参数：

```
-metadir <path-to-meta-directory>
或
-datadir <path-to-data-directory>
```
可选参数：

```
-database <database>
-retention <retention policy>
-shard <shard id>
```

示例，恢复数据库：

```
$ influxd restore -database telegraf -datadir /var/lib/influxdb/data /tmp/backup                                                                         
Restoring from backup /tmp/backup/telegraf.*
unpacking /var/lib/influxdb/data/telegraf/default/2/000000004-000000003.tsm
unpacking /var/lib/influxdb/data/telegraf/default/2/000000005-000000001.tsm
```

<a name="2FsTi"></a>
## 五、原理篇
这块研究的时候，可以看一下网易大神：范欣欣的文章<br />[http://hbasefly.com/category/%E6%97%B6%E5%BA%8F%E6%95%B0%E6%8D%AE%E5%BA%93/](http://hbasefly.com/category/%E6%97%B6%E5%BA%8F%E6%95%B0%E6%8D%AE%E5%BA%93/)
<a name="7iGlp"></a>
## 六、账号体系
<a name="jdzBF"></a>
### 6.1 查看账号

```
show users
```
<a name="PrK7D"></a>
### 6.2 新建账号

```
create user “username” with password ‘password’ 
```
此处注意，password必须用单引号引起来。
<a name="a3jJq"></a>
### 6.3 新建含有权限的账号

```
create user “username” with password ‘password’ with all privileges
```
<a name="QhrqL"></a>
### 6.4 修改用户密码

```
SET PASSWORD FOR admin =’influx@gpscloud’
```

<a name="mIyRQ"></a>
### 6.5 删除用户

```
drop user ‘username’ 删除用户
```

<a name="mI20p"></a>
## 七、常用问题篇
<a name="ymCTJ"></a>
### 7.1 集群搭建方式
官方的集群方案是收费的，但是由于influxDB是开源的，所以应该会有人做出来开源集群模式。

<a name="R3JaQ"></a>
### 7.2 UI 无法连接的问题
InfluxDB在0.13版本以后，就默认关闭了web管理页面。如果是0.13版本到1.1版本之间的版本，我们可以通过修改配置文件：<br />![](https://cdn.nlark.com/yuque/0/2019/jpeg/187105/1557153763170-0db6923e-1638-45e1-98e4-3030f36f453c.jpeg#align=left&display=inline&height=274&originHeight=274&originWidth=614&size=0&status=done&width=614)<br />将enabled改为true并把注释删除。然后https看应用场景。<br />但是如果使用的是1.1版本之后的，则没有web管理界面了。<br />版本如何查看，我们可以使用客户端软件 influx --version就能看到。

<a name="M18mb"></a>
### 7.3 单机InfluxDB能支撑多大
我们查看API，他可以批量插入和非批量插入。以下测试在自己电脑上进行测试，得出结论基本是数字的差别，规律是相同的。

<a name="eAgLh"></a>
#### 7.3.1 批量插入：
以下数据测试方法: 10次以下结果的平均。平均看到的情况，在5万附近，并不会随着数据量的增长，而出现明显的增长

```
1次1万条, 花费了151ms, 平均下来一秒插入66050条数据
1次2万条, 花费了342ms, 平均下来一秒插入58462条数据
1次3万条, 花费了652ms, 平均下来一秒插入46005条数据
1次4万条, 花费了766ms, 平均下来一秒插入52192条数据
1次5万条, 花费了882ms, 平均下来一秒插入56637条数据
1次6万条, 花费了1185ms, 平均下来一秒插入50603条数据
1次7万条, 花费了1277ms, 平均下来一秒插入54807条数据
1次8万条, 花费了1472ms, 平均下来一秒插入54336条数据
1次9万条, 花费了1597ms, 平均下来一秒插入56330条数据
1次10万条, 花费了1764ms, 平均下来一秒插入56663条数据
1次11万条, 花费了2181ms, 平均下来一秒插入50421条数据
1次12万条, 花费了2494ms, 平均下来一秒插入48107条数据
1次13万条, 花费了2642ms, 平均下来一秒插入49197条数据
1次14万条, 花费了2825ms, 平均下来一秒插入49557条数据
1次15万条以上，测试过程中会报错，所以单次插入量不能太大。先会出现Internal Server Error的情况
再往上增长，则会出现Request Entity Too Large.
```
所以批量插入的情况来看，量能承受比较大。

```javascript
const Influx = require('influx');

var result = []
async function write(num){
    const influx = new Influx.InfluxDB({host: 'localhost'});
    if((await influx.getDatabaseNames()).filter(value=>{
        return value === 'helloworld';
    }).length !== 0){
        await influx.dropDatabase('helloworld')
        await influx.createDatabase('helloworld');
    }
    const influx2 = new Influx.InfluxDB({host: 'localhost', database: 'helloworld'});
    let startTime = new Date().getTime();
    let data = [];
    for(let i = 0; i < num * 10000; i++){
        data.push({
            measurement: 'hello',
            tags: {host: 'box' + i + '.example.com'},
            fields: {cpu: 123, mem: 234}
        })
    }
    await influx2.writePoints(data);
    const useTime = new Date().getTime() - startTime;
    if(result.map(item=>{
        return item.num;
    }).filter(value => {
        return value === num;
    }).length === 0){
        result.push({
            num: num,
            useTime: [useTime]
        })
    }else{
        result.filter(value=>{
            return value.num === num;
        })[0].useTime.push(useTime);
    }
}

async function boostrap(){
    for(let i = 0; i < 10; i++){
        for(let j = 0; j < 15; j++){
            await write(j);
        }
    }
    result.map(item=>{
        let useTime = item.useTime.reduce((previous, current)=>{return previous + current}, 0)/item.useTime.length
        console.log('1次' + item.num + '万条, 花费了' + parseInt(useTime) + "ms, 平均下来一秒插入" + parseInt(item.num * 10000/useTime*1000) +"条数据")
    })
}
boostrap();
```

<a name="1bvIk"></a>
#### 7.3.2 非批量插入
非批量操作，简称，每次插入一条数据。那这种肯定是会比较慢的。应用场景：比如应用连接了对应的influxDB，然后来一条数据插入一条。<br />以下是测试代码：

```javascript
const Influx = require('influx');

var result = []
async function write(num){
    const influx = new Influx.InfluxDB({host: 'localhost'});
    if((await influx.getDatabaseNames()).filter(value=>{
        return value === 'helloworld';
    }).length !== 0){
        await influx.dropDatabase('helloworld')
        await influx.createDatabase('helloworld');
    }
    const influx2 = new Influx.InfluxDB({host: 'localhost', database: 'helloworld'});
    let startTime = new Date().getTime();
    for(let i = 0; i < num * 100; i++){
        await influx2.writePoints([{
            measurement: 'hello',
            tags: {host: 'box' + i + '.example.com'},
            fields: {cpu: 123, mem: 234}
        }]);
    }
    const useTime = new Date().getTime() - startTime;
    if(result.map(item=>{
        return item.num;
    }).filter(value => {
        return value === num;
    }).length === 0){
        result.push({
            num: num,
            useTime: [useTime]
        })
    }else{
        result.filter(value=>{
            return value.num === num;
        })[0].useTime.push(useTime);
    }
}

async function boostrap(){
    for(let i = 0; i < 10; i++){
        for(let j = 0; j < 15; j++){
            await write(j);
        }
    }
    result.map(item=>{
        let useTime = item.useTime.reduce((previous, current)=>{return previous + current}, 0)/item.useTime.length
        console.log('1次' + item.num*100 + '条, 花费了' + parseInt(useTime) + "ms, 平均下来一秒插入" + parseInt(item.num * 100/useTime*1000) +"条数据")
    })
}
boostrap();
```
数据结果：<br />测试结果，均是10次的平均值，还是比较准确反映了当前的性能的。
```
100条, 花费了771ms, 平均下来一秒插入129条数据
200条, 花费了1692ms, 平均下来一秒插入118条数据
300条, 花费了2319ms, 平均下来一秒插入129条数据
400条, 花费了2994ms, 平均下来一秒插入133条数据
500条, 花费了3776ms, 平均下来一秒插入132条数据
600条, 花费了4788ms, 平均下来一秒插入125条数据
700条, 花费了5250ms, 平均下来一秒插入133条数据
800条, 花费了5973ms, 平均下来一秒插入133条数据
900条, 花费了6765ms, 平均下来一秒插入133条数据
1000条, 花费了7491ms, 平均下来一秒插入133条数据
1100条, 花费了8161ms, 平均下来一秒插入134条数据
1200条, 花费了8733ms, 平均下来一秒插入137条数据
1300条, 花费了9492ms, 平均下来一秒插入136条数据
1400条, 花费了10373ms, 平均下来一秒插入134条数据
```

<a name="QJU9u"></a>
#### 7.3.3 存储数据
我们通过查看du -sh /var/lib/influxdb/data/helloworld，最后这个helloworld也就是对应database的名字。<br />数据情况:

```
1万条：504k
2万条：956k
3万条：1.4M
4万条：1.9M
5万条：2.3M
6万条：2.7M
7万条：3.2M
8万条：3.6M
9万条：4.1M
10万条：4.5M
11万条：4.9M
12万条：5.4M
13万条：5.8M
14万条：6.3M
15万条：6.7M
```
这个数据大小，一个point数据的大小有关，比如当前一条数据：<br />其中host字段是tag，而cpu和mem是fields，所以关于数据量的估量，也是监控系统需要考虑的问题。<br />![image.png](https://cdn.nlark.com/yuque/0/2019/png/187105/1557235302652-ee68b662-41ef-4aed-b22f-66a077d4fe72.png#align=left&display=inline&height=81&name=image.png&originHeight=81&originWidth=380&size=7928&status=done&width=380)
<a name="D9N0N"></a>
#### 7.3.4 读的性能
为什么要测试读的性能，我们会在Grafana上面配置很多图表，如果数据源是InfluxDB的话，一个图表意味着1~N次请求，那么一个dashboard其实是有好多请求。这个时候，如果有很多个人在看自己的Dashboard的话，那么对于InfluxDB的压力还是比较大的。<br />测试代码：
```javascript
const Influx = require('influx');

var result = []
async function write(num){
    const influx = new Influx.InfluxDB({host: 'localhost'});
    if((await influx.getDatabaseNames()).filter(value=>{
        return value === 'helloworld';
    }).length !== 0){
        await influx.dropDatabase('helloworld')
        await influx.createDatabase('helloworld');
    }
    
    let startTime = new Date().getTime();
    let data = [];
    for(let i = 0; i < num * 10000; i++){
        data.push({
            measurement: 'hello',
            tags: {host: 'box' + i + '.example.com'},
            fields: {cpu: 123, mem: 234}
        })
    }
    await influx2.writePoints(data);
    const useTime = new Date().getTime() - startTime;
    if(result.map(item=>{
        return item.num;
    }).filter(value => {
        return value === num;
    }).length === 0){
        result.push({
            num: num,
            useTime: [useTime]
        })
    }else{
        result.filter(value=>{
            return value.num === num;
        })[0].useTime.push(useTime);
    }
}

async function boostrap(){
    const influx2 = new Influx.InfluxDB({host: 'localhost', database: 'helloworld'});
    result = []
    for(let i = 0; i < 1; i++){
        for(let j = 1; j < 15; j++){
            let startTime = new Date().getTime();
            for(let z = 0; z < 1*j; z++){
                await influx2.query(`select count(mem) from hello limit 10000`)
            }
            let endTime = new Date().getTime();
            if(result.map(item=>{
                return item.num;
            }).filter(value=>{
                return value === 1 *j
            }).length === 0){
                result.push({
                    num: 1*j,
                    useTime: [endTime-startTime]
                })
            }else{
                result.filter(value=>{
                    return value.num === 1*j
                })[0].useTime.push(endTime-startTime)
            }
            console.log("====>" + j)
        }
    }
    result.map(item=>{
        console.log('一共' + item.num + "次查询, 平均花费" + parseInt(item.useTime.reduce((a,b)=>a+b, 0)/item.num/item.useTime.length) + "ms")
    })
}
boostrap();
```
结论数据：<br />在15万的数据表中，进行查询count的操作。select count(mem) from hello limit 10000
```
一共1次查询, 平均花费200ms
一共2次查询, 平均花费315ms
一共3次查询, 平均花费171ms
一共4次查询, 平均花费143ms
一共5次查询, 平均花费169ms
一共6次查询, 平均花费139ms
一共7次查询, 平均花费155ms
一共8次查询, 平均花费166ms
一共9次查询, 平均花费147ms
一共10次查询, 平均花费188ms
一共11次查询, 平均花费162ms
一共12次查询, 平均花费246ms
一共13次查询, 平均花费141ms
一共14次查询, 平均花费160ms
```
然后得出上面的结论，说明当数据量比较大的时候，查询一次操作，需要花费的时间还是蛮大的。<br />所以这块如何提升性能，还是比较重要的。不然图表多了，那必然响应不过来了。<br />15w数据量是多大的数据量：假设我们5秒一条数据，1天的量：12 * 60 * 24=17280，差不多一天2万的量，也就是有一周的数据量。<br />当我们在一个一周数据量的数据里面，捞我们想要的数据，假如一个grafana的dashboard是10个图表，假设10个接口，那按照上面的算法，毛估估还是要花费比较长时间的。<br />然后用ab测试了一下性能：

```
ab -n 20 -c 5 -H "Cookie: email=hzjinbing%40163.com; username=jinbing; grafana_session=af2db0101608c095d7ea7b55e7bc0ddb" http://127.0.0.1:32768/api/datasources/proxy/6/query\?db\=helloworld\&q\=SELECT%20count\(%22mem%22\)%20FROM%20%22hello%22%20WHERE%20time%20%3E%3D%20now\(\)%20-%206h%20GROUP%20BY%20time\(20s\)%20fill\(null\)\&epoch\=ms
```
此处我拿了一个grafana图表的接口进行压测。<br />效果：大概每秒只能处理两个请求，说明，真实的时候，当量大了之后，确实这个反应应该如何处理，否则grafana去InfluxDB数据源去展示的时候，人多了完全不能看呀。
<a name="oe4XK"></a>
#### 7.3.5 结论
如果在单机的情况下，我们可以做对应的数据聚合层，然后将数据聚合然后再存储到对应的influxDB。

1. 减少了跟InfluxDB的连接数量
1. 单次插入一条的性能，要远远小于单次批量插入多条的性能。

这块，influxdb-relay 是官方提供的高可用方案，但是它只提供简单的写入功能。初期使用问题不大，因为对于写的问题暴露的不明显，但是随着业务方的增长，对于读这块的请求比较大的时候，又会引起InfluxDB的性能瓶颈。
## 八、参考文档

1. influxDB系列教程 [https://www.linuxdaxue.com/how-to-install-influxdb.html](https://www.linuxdaxue.com/how-to-install-influxdb.html)
1. influxDB原理：[http://hbasefly.com/category/%E6%97%B6%E5%BA%8F%E6%95%B0%E6%8D%AE%E5%BA%93/](http://hbasefly.com/category/%E6%97%B6%E5%BA%8F%E6%95%B0%E6%8D%AE%E5%BA%93/)
