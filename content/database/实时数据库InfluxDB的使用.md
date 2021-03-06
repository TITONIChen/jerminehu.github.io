---
title: "实时数据库InfluxDB的使用"
date: 2018-09-07T14:00:54+08:00
categories: ["All","docker","database"]
tags: ["docker","database","influxdb"]
toc: true
author: "Jermine"
author_homepage:  "/"
weight: 70
keywords: ["docker","database","influxdb" ]
description: "实时数据库InfluxDB的使用"
---

## InfluxDB介绍：

InfluxDB是一个开源的时序数据库，使用GO语言开发，特别适合用于处理和分析资源监控数据这种时序相关数据。而InfluxDB自带的各种特殊函数如求标准差，随机取样数据，统计数据变化比等，使数据统计和实时分析变得十分方便。在我们的容器资源监控系统中，就采用了InfluxDB存储cadvisor的监控数据。本文对InfluxDB的基本概念和一些特色功能做一个详细介绍，内容主要是翻译整理自官网文档，如有错漏，请指正。

## 安装配置
这里说一下使用docker容器运行influxdb的步骤，物理机安装请参照官方文档。拉取镜像文件后运行即可，当前最新版本是1.3.5。启动容器时设置挂载的数据目录和开放端口。InfluxDB的操作语法InfluxQL与SQL基本一致，也提供了一个类似mysql-client的名为influx的CLI。InfluxDB本身是支持分布式部署多副本存储的，本文介绍都是针对的单节点单副本。

```
# docker pull influxdb
# docker run -idt --name influxdb -p 8086:8086 -v /Users/ssj/influxdb:/var/lib/influxdb influxdb
f216e9be15bff545befecb30d1d275552026216a939cc20c042b17419e3bde31
# docker exec -it influxdb /bin/bash 
root@f216e9be15bf:/# influx
Connected to http://localhost:8086 version 1.3.5
InfluxDB shell version: 1.3.5
> create database cadvisor  ## 创建数据库cadvisor
> show databases           
name: databases
name
----
_internal
cadvisor
> CREATE USER testuser WITH PASSWORD 'testpwd' ## 创建用户和设置密码
> GRANT ALL PRIVILEGES ON cadvisor TO testuser ## 授权数据库给指定用户
> CREATE RETENTION POLICY "cadvisor_retention" ON "cadvisor" DURATION 30d REPLICATION 1 DEFAULT ## 创建默认的数据保留策略，设置保存时间30天，副本为1
```

##  重要概念
influxdb里面有一些重要概念：`database，timestamp，field key， field value， field set，tag key，tag value，tag set，measurement， retention policy ，series，point`。结合下面的例子数据来说明这几个概念：

```
name: census
-————————————
time                     butterflies     honeybees     location   scientist
2015-08-18T00:00:00Z      12                23           1         langstroth
2015-08-18T00:00:00Z      1                 30           1         perpetua
2015-08-18T00:06:00Z      11                28           1         langstroth
2015-08-18T00:06:00Z      3                 28           1         perpetua
2015-08-18T05:54:00Z      2                 11           2         langstroth
2015-08-18T06:00:00Z      1                 10           2         langstroth
2015-08-18T06:06:00Z      8                 23           2         perpetua
2015-08-18T06:12:00Z      7                 22           2         perpetua
```

### timestamp

既然是时间序列数据库，influxdb的数据都有一列名为time的列，里面存储UTC时间戳。

### field key，field value，field set
butterflies和honeybees两列数据称为字段(fields)，influxdb的字段由field key和field value组成。其中butterflies和honeybees为field key，它们为string类型，用于存储元数据。

而butterflies这一列的数据12-7为butterflies的field value，同理，honeybees这一列的23-22为honeybees的field value。field value可以为string，float，integer或boolean类型。field value通常都是与时间关联的。

field key和field value对组成的集合称之为field set。如下：

```
butterflies = 12 honeybees = 23
butterflies = 1 honeybees = 30
butterflies = 11 honeybees = 28
butterflies = 3 honeybees = 28
butterflies = 2 honeybees = 11
butterflies = 1 honeybees = 10
butterflies = 8 honeybees = 23
butterflies = 7 honeybees = 22

```

在influxdb中，字段必须存在。注意，字段是没有索引的。如果使用字段作为查询条件，会扫描符合查询条件的所有字段值，性能不及tag。类比一下，fields相当于SQL的没有索引的列。

### tag key，tag value，tag set
location和scientist这两列称为标签(tags)，标签由tag key和tag value组成。location这个tag key有两个tag value：1和2，scientist有两个tag value：langstroth和perpetua。tag key和tag value对组成了tag set，示例中的tag set如下：

```
location = 1, scientist = langstroth
location = 2, scientist = langstroth
location = 1, scientist = perpetua
location = 2, scientist = perpetua

```

tags是可选的，但是强烈建议你用上它，因为tag是有索引的，tags相当于SQL中的有索引的列。tag value只能是string类型 如果你的常用场景是根据butterflies和honeybees来查询，那么你可以将这两个列设置为tag，而其他两列设置为field，tag和field依据具体查询需求来定。

### measurement
measurement是fields，tags以及time列的容器，measurement的名字用于描述存储在其中的字段数据，类似mysql的表名。如上面例子中的measurement为census。measurement相当于SQL中的表，本文中我在部分地方会用表来指代measurement。

### retention policy
retention policy指数据保留策略，示例数据中的retention policy为默认的autogen。它表示数据一直保留永不过期，副本数量为1。你也可以指定数据的保留时间，如30天。

### series
series是共享同一个retention policy，measurement以及tag set的数据集合。示例中数据有4个series，如下:

Arbitrary series number | Retention policy | Measurement | Tag set
------------------------|------------------|-------------|--------
series 1 | autogen | census | location = 1,scientist = langstroth
series 2 | autogen | census | location = 2,scientist = langstroth
series 3 | autogen | census | location = 1,scientist = perpetua
series 4 | autogen | census | location = 2,scientist = perpetua

### point
point则是同一个series中具有相同时间的field set，points相当于SQL中的数据行。如下面就是一个point：

```
name: census
-----------------
time                  butterflies    honeybees   location    scientist
2015-08-18T00:00:00Z       1            30           1        perpetua

```

### database

上面提到的结构都存储在数据库中，示例的数据库为my_database。一个数据库可以有多个measurement，retention policy， continuous queries以及user。influxdb是一个无模式的数据库，可以很容易的添加新的measurement，tags，fields等。而它的操作却和传统的数据库一样，可以使用类SQL语言查询和修改数据。

influxdb不是一个完整的CRUD数据库，它更像是一个CR-ud数据库。它优先考虑的是增加和读取数据而不是更新和删除数据的性能，而且它阻止了某些更新和删除行为使得创建和读取数据更加高效。

## 特色函数
influxdb函数分为聚合函数，选择函数，转换函数，预测函数等。除了与普通数据库一样提供了基本操作函数外，还提供了一些特色函数以方便数据统计计算，下面会一一介绍其中一些常用的特色函数。

* 聚合函数：`FILL(), INTEGRAL()，SPREAD()， STDDEV()，MEAN(), MEDIAN()`等。
* 选择函数: `SAMPLE(), PERCENTILE(), FIRST(), LAST(), TOP(), BOTTOM()`等。
* 转换函数: `DERIVATIVE(), DIFFERENCE()`等。
* 预测函数：`HOLT_WINTERS()`。

先从官网导入测试数据（注：这里测试用的版本是1.3.1，最新版本是1.3.5）:

```
$ curl https://s3.amazonaws.com/noaa.water-database/NOAA_data.txt -o NOAA_data.txt
$ influx -import -path=NOAA_data.txt -precision=s -database=NOAA_water_database
$ influx -precision rfc3339 -database NOAA_water_database
Connected to http://localhost:8086 version 1.3.1
InfluxDB shell 1.3.1
> show measurements
name: measurements
name
----
average_temperature
distincts
h2o_feet
h2o_pH
h2o_quality
h2o_temperature

> show series from h2o_feet;
key
---
h2o_feet,location=coyote_creek
h2o_feet,location=santa_monica

```

下面的例子都以官方示例数据库来测试，这里只用部分数据以方便观察。measurement为`h2o_feet`，`tag key`为`location`，`field key`有`level description`和`water_level`两个。

```

> SELECT * FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z'
name: h2o_feet
time                 level description    location     water_level
----                 -----------------    --------     -----------
2015-08-18T00:00:00Z between 6 and 9 feet coyote_creek 8.12
2015-08-18T00:00:00Z below 3 feet         santa_monica 2.064
2015-08-18T00:06:00Z between 6 and 9 feet coyote_creek 8.005
2015-08-18T00:06:00Z below 3 feet         santa_monica 2.116
2015-08-18T00:12:00Z between 6 and 9 feet coyote_creek 7.887
2015-08-18T00:12:00Z below 3 feet         santa_monica 2.028
2015-08-18T00:18:00Z between 6 and 9 feet coyote_creek 7.762
2015-08-18T00:18:00Z below 3 feet         santa_monica 2.126
2015-08-18T00:24:00Z between 6 and 9 feet coyote_creek 7.635
2015-08-18T00:24:00Z below 3 feet         santa_monica 2.041
2015-08-18T00:30:00Z between 6 and 9 feet coyote_creek 7.5
2015-08-18T00:30:00Z below 3 feet         santa_monica 2.051

```

### GROUP BY，FILL()
如下语句中`GROUP BY time(12m)`,* 表示以每12分钟和tag(location)分组(如果是`GROUP BY time(12m)`则表示仅每12分钟分组，`GROUP BY` 参数只能是time和tag)。然后fill(200)表示如果这个时间段没有数据，以200填充，mean(field_key)求该范围内数据的平均值(注意：这是依据series来计算。其他还有SUM求和，MEDIAN求中位数)。LIMIT 7表示限制返回的point(记录数)最多为7条，而SLIMIT 1则是限制返回的series为1个。

注意这里的时间区间，起始时间为整点前包含这个区间第一个12m的时间，比如这里为 `2015-08-17T:23:48:00Z`，第一条为 `2015-08-17T23:48:00Z <= t < 2015-08-18T00:00:00Z`这个区间的`location=coyote_creek`的`water_level`的平均值，这里没有数据，于是填充的200。第二条为 `2015-08-18T00:00:00Z <= t < 2015-08-18T00:12:00Z`区间的`location=coyote_creek`的`water_level`平均值，这里为 `（8.12+8.005）/ 2 = 8.0625`，其他以此类推。

而`GROUP BY time(10m)`则表示以10分钟分组，起始时间为包含这个区间的第一个10m的时间，即 `2015-08-17T23:40:00Z`。默认返回的是第一个series，如果要计算另外那个series，可以在SQL语句后面加上 `SOFFSET 1`。

那如果时间小于数据本身采集的时间间隔呢，比如`GROUP BY time(10s)`呢？这样的话，就会按10s取一个点，没有数值的为空或者FILL填充，对应时间点有数据则保持不变。
```

## GROUP BY time(12m)
> SELECT mean("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m),* fill(200) LIMIT 7 SLIMIT 1
name: h2o_feet
tags: location=coyote_creek
time                 mean
----                 ----
2015-08-17T23:48:00Z 200
2015-08-18T00:00:00Z 8.0625
2015-08-18T00:12:00Z 7.8245
2015-08-18T00:24:00Z 7.5675

## GROUP BY time(10m)，SOFFSET设置为1
> SELECT mean("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(10m),* fill(200) LIMIT 7 SLIMIT 1 SOFFSET 1
name: h2o_feet
tags: location=santa_monica
time                 mean
----                 ----
2015-08-17T23:40:00Z 200
2015-08-17T23:50:00Z 200
2015-08-18T00:00:00Z 2.09
2015-08-18T00:10:00Z 2.077
2015-08-18T00:20:00Z 2.041
2015-08-18T00:30:00Z 2.051

```

### INTEGRAL(field_key, unit)

计算数值字段值覆盖的曲面的面积值并得到面积之和。测试数据如下：

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'

name: h2o_feet
time                   water_level
----                   -----------
2015-08-18T00:00:00Z   2.064
2015-08-18T00:06:00Z   2.116
2015-08-18T00:12:00Z   2.028
2015-08-18T00:18:00Z   2.126
2015-08-18T00:24:00Z   2.041
2015-08-18T00:30:00Z   2.051

```

使用INTERGRAL计算面积。注意，这个面积就是这些点连接起来后与时间围成的不规则图形的面积，注意unit默认是以1秒计算，所以下面语句计算结果为`3732.66=2.028*1800+`分割出来的梯形和三角形面积。如果`unit`改为1分，则结果为`3732.66/60 = 62.211`。unit为2分，则结果为`3732.66/120 = 31.1055`。以此类推。

```
# unit为默认的1秒
> SELECT INTEGRAL("water_level") FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'
name: h2o_feet
time                 integral
----                 --------
1970-01-01T00:00:00Z 3732.66

# unit为1分
> SELECT INTEGRAL("water_level", 1m) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'
name: h2o_feet
time                 integral
----                 --------
1970-01-01T00:00:00Z 62.211

```

### SPREAD(field_key)
计算数值字段的最大值和最小值的差值。
```
> SELECT SPREAD("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m),* fill(18) LIMIT 3 SLIMIT 1 SOFFSET 1
name: h2o_feet
tags: location=santa_monica
time                 spread
----                 ------
2015-08-17T23:48:00Z 18
2015-08-18T00:00:00Z 0.052000000000000046
2015-08-18T00:12:00Z 0.09799999999999986

```

### STDDEV(field_key)
计算字段的标准差。influxdb用的是贝塞尔修正的标准差计算公式 ，如下：

    mean=(v1+v2+...+vn)/n;
    stddev = math.sqrt(
    ((v1-mean)2 + (v2-mean)2 + ...+(vn-mean)2)/(n-1)
    )

```
> SELECT STDDEV("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m),* fill(18) SLIMIT 1;
name: h2o_feet
tags: location=coyote_creek
time                 stddev
----                 ------
2015-08-17T23:48:00Z 18
2015-08-18T00:00:00Z 0.08131727983645186
2015-08-18T00:12:00Z 0.08838834764831845
2015-08-18T00:24:00Z 0.09545941546018377

```

### PERCENTILE(field_key, N)
选取某个字段中大于N%的这个字段值。

如果一共有4条记录，N为10，则`10%*4=0.4`，四舍五入为0，则查询结果为空。N为20，则 `20% * 4 = 0.8`，四舍五入为1，选取的是4个数中最小的数。如果N为`40，40% * 4 = 1.6`，四舍五入为2，则选取的是4个数中第二小的数。由此可以看出N=100时，就跟`MAX(field_key)`是一样的，而当N=50时，与`MEDIAN(field_key)`在字段值为奇数个时是一样的。

```

> SELECT PERCENTILE("water_level",20) FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)
name: h2o_feet
time                 percentile
----                 ----------
2015-08-17T23:48:00Z 
2015-08-18T00:00:00Z 2.064
2015-08-18T00:12:00Z 2.028
2015-08-18T00:24:00Z 2.041

> SELECT PERCENTILE("water_level",40) FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)
name: h2o_feet
time                 percentile
----                 ----------
2015-08-17T23:48:00Z 
2015-08-18T00:00:00Z 2.116
2015-08-18T00:12:00Z 2.126
2015-08-18T00:24:00Z 2.051

```

### SAMPLE(field_key, N)

随机返回field key的N个值。如果语句中有GROUP BY time()，则每组数据随机返回N个值。

```

> SELECT SAMPLE("water_level",2) FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z';
name: h2o_feet
time                 sample
----                 ------
2015-08-18T00:00:00Z 2.064
2015-08-18T00:12:00Z 2.028

> SELECT SAMPLE("water_level",2) FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m);
name: h2o_feet
time                 sample
----                 ------
2015-08-18T00:06:00Z 2.116
2015-08-18T00:06:00Z 8.005
2015-08-18T00:12:00Z 7.887
2015-08-18T00:18:00Z 7.762
2015-08-18T00:24:00Z 7.635
2015-08-18T00:30:00Z 2.051

```
### CUMULATIVE_SUM(field_key)
计算字段值的递增和。

```
> SELECT CUMULATIVE_SUM("water_level") FROM "h2o_feet" WHERE time >= '2015-08-17T23:48:00Z' AND time <= '2015-08-18T00:30:00Z';
name: h2o_feet
time                 cumulative_sum
----                 --------------
2015-08-18T00:00:00Z 8.12
2015-08-18T00:00:00Z 10.184
2015-08-18T00:06:00Z 18.189
2015-08-18T00:06:00Z 20.305
2015-08-18T00:12:00Z 28.192
2015-08-18T00:12:00Z 30.22
2015-08-18T00:18:00Z 37.982
2015-08-18T00:18:00Z 40.108
2015-08-18T00:24:00Z 47.742999999999995
2015-08-18T00:24:00Z 49.78399999999999
2015-08-18T00:30:00Z 57.28399999999999
2015-08-18T00:30:00Z 59.334999999999994

```
### DERIVATIVE(field_key, unit) 和 NON_NEGATIVE_DERIVATIVE(field_key, unit)
计算字段值的变化比。unit默认为1s，即计算的是1秒内的变化比。

如下面的第一个数据计算方法是 `(2.116-2.064)/(6*60) = 0.00014..`，其他计算方式同理。虽然原始数据是6m收集一次，但是这里的变化比默认是按秒来计算的。如果要按6m计算，则设置unit为6m即可。

```
> SELECT DERIVATIVE("water_level") FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'
name: h2o_feet
time                 derivative
----                 ----------
2015-08-18T00:06:00Z 0.00014444444444444457
2015-08-18T00:12:00Z -0.00024444444444444465
2015-08-18T00:18:00Z 0.0002722222222222218
2015-08-18T00:24:00Z -0.000236111111111111
2015-08-18T00:30:00Z 0.00002777777777777842

> SELECT DERIVATIVE("water_level", 6m) FROM "h2o_feet" WHERE "location" = 'santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z'
name: h2o_feet
time                 derivative
----                 ----------
2015-08-18T00:06:00Z 0.052000000000000046
2015-08-18T00:12:00Z -0.08800000000000008
2015-08-18T00:18:00Z 0.09799999999999986
2015-08-18T00:24:00Z -0.08499999999999996
2015-08-18T00:30:00Z 0.010000000000000231
```

而DERIVATIVE结合`GROUP BY time`，以及mean可以构造更加复杂的查询，如下所示:

```
> SELECT DERIVATIVE(mean("water_level"), 6m) FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' group by time(12m), *
name: h2o_feet
tags: location=coyote_creek
time                 derivative
----                 ----------
2015-08-18T00:12:00Z -0.11900000000000022
2015-08-18T00:24:00Z -0.12849999999999984

name: h2o_feet
tags: location=santa_monica
time                 derivative
----                 ----------
2015-08-18T00:12:00Z -0.00649999999999995
2015-08-18T00:24:00Z -0.015499999999999847
```

这个计算其实是先根据`GROUP BY time`求平均值，然后对这个平均值再做变化比的计算。因为数据是按12分钟分组的，而变化比的unit是6分钟，所以差值除以2(12/6)才得到变化比。如第一个值是 `(7.8245-8.0625)/2 = -0.1190`。
```
> SELECT mean("water_level") FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' group by time(12m), *
name: h2o_feet
tags: location=coyote_creek
time                 mean
----                 ----
2015-08-18T00:00:00Z 8.0625
2015-08-18T00:12:00Z 7.8245
2015-08-18T00:24:00Z 7.5675

name: h2o_feet
tags: location=santa_monica
time                 mean
----                 ----
2015-08-18T00:00:00Z 2.09
2015-08-18T00:12:00Z 2.077
2015-08-18T00:24:00Z 2.0460000000000003
```
`NON_NEGATIVE_DERIVATIVE`与`DERIVATIVE`不同的是它只返回的是非负的变化比:
```
> SELECT DERIVATIVE(mean("water_level"), 6m) FROM "h2o_feet" WHERE location='santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' group by time(6m), *
name: h2o_feet
tags: location=santa_monica
time                 derivative
----                 ----------
2015-08-18T00:06:00Z 0.052000000000000046
2015-08-18T00:12:00Z -0.08800000000000008
2015-08-18T00:18:00Z 0.09799999999999986
2015-08-18T00:24:00Z -0.08499999999999996
2015-08-18T00:30:00Z 0.010000000000000231

> SELECT NON_NEGATIVE_DERIVATIVE(mean("water_level"), 6m) FROM "h2o_feet" WHERE location='santa_monica' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' group by time(6m), *
name: h2o_feet
tags: location=santa_monica
time                 non_negative_derivative
----                 -----------------------
2015-08-18T00:06:00Z 0.052000000000000046
2015-08-18T00:18:00Z 0.09799999999999986
2015-08-18T00:30:00Z 0.010000000000000231
```

## 连续查询
### 基本语法
连续查询(`CONTINUOUS QUERY`，简写为CQ)是指定时自动在实时数据上进行的`InfluxQL`查询，查询结果可以存储到指定的`measurement`中。基本语法格式如下：
```
CREATE CONTINUOUS QUERY <cq_name> ON <database_name>
BEGIN
  <cq_query>
END

# cq_query格式：

SELECT <function[s]> INTO <destination_measurement> FROM <measurement> [WHERE <stuff>] GROUP BY time(<interval>)[,<tag_key[s]>]
```

CQ操作的是实时数据，它使用本地服务器的时间戳、`GROUP BY time()`时间间隔以及InfluxDB预先设置好的时间范围来确定什么时候开始查询以及查询覆盖的时间范围。注意CQ语句里面的WHERE条件是没有时间范围的，因为CQ会根据GROUP BY time()自动确定时间范围。

CQ执行的时间间隔和`GROUP BY time()`的时间间隔一样，它在InfluxDB预先设置的时间范围的起始时刻执行。如果`GROUP BY time(1h)`，则单次查询的时间范围为 `now()-GROUP BY time(1h)`到 `now()`，也就是说，如果当前时间为17点，这次查询的时间范围为 16:00到16:59.99999。

下面看几个示例，示例数据如下，这是数据库`transportation`中名为`bus_data`的`measurement`，每15分钟统计一次乘客数和投诉数。数据文件`bus_data.txt`如下：

```
# DDL
CREATE DATABASE transportation

# DML
# CONTEXT-DATABASE: transportation 

bus_data,complaints=9 passengers=5 1472367600
bus_data,complaints=9 passengers=8 1472368500
bus_data,complaints=9 passengers=8 1472369400
bus_data,complaints=9 passengers=7 1472370300
bus_data,complaints=9 passengers=8 1472371200
bus_data,complaints=7 passengers=15 1472372100
bus_data,complaints=7 passengers=15 1472373000
bus_data,complaints=7 passengers=17 1472373900
bus_data,complaints=7 passengers=20 1472374800

```

导入数据，命令如下：

```
root@f216e9be15bf:/# influx -import -path=bus_data.txt -precision=s
root@f216e9be15bf:/# influx -precision=rfc3339 -database=transportation
Connected to http://localhost:8086 version 1.3.5
InfluxDB shell version: 1.3.5
> select * from bus_data
name: bus_data
time                 complaints passengers
----                 ---------- ----------
2016-08-28T07:00:00Z 9          5
2016-08-28T07:15:00Z 9          8
2016-08-28T07:30:00Z 9          8
2016-08-28T07:45:00Z 9          7
2016-08-28T08:00:00Z 9          8
2016-08-28T08:15:00Z 7          15
2016-08-28T08:30:00Z 7          15
2016-08-28T08:45:00Z 7          17
2016-08-28T09:00:00Z 7          20
```

#### 示例1 自动缩小取样存储到新的measurement中

对单个字段自动缩小取样并存储到新的measurement中。

```
CREATE CONTINUOUS QUERY "cq_basic" ON "transportation"
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h)
END
```

这个CQ的意思就是对`bus_data`每小时自动计算取样数据的平均乘客数并存储到 `average_passengers`中。那么在2016-08-28这天早上会执行如下流程：
```
At 8:00 cq_basic 执行查询，查询时间范围 time >= '7:00' AND time < '08:00'.
cq_basic写入一条记录到 average_passengers:
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
At 9:00 cq_basic 执行查询，查询时间范围 time >= '8:00' AND time < '9:00'.
cq_basic写入一条记录到 average_passengers:
name: average_passengers
------------------------
time                   mean
2016-08-28T08:00:00Z   13.75

# Results
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
2016-08-28T08:00:00Z   13.75
```
#### 示例2 自动缩小取样并存储到新的保留策略（Retention Policy）中

```
CREATE CONTINUOUS QUERY "cq_basic_rp" ON "transportation"
BEGIN
  SELECT mean("passengers") INTO "transportation"."three_weeks"."average_passengers" FROM "bus_data" GROUP BY time(1h)
END
```
与示例1类似，不同的是保留的策略不是`autogen`，而是改成了`three_weeks`(创建保留策略语法 `CREATE RETENTION POLICY "three_weeks" ON "transportation" DURATION 3w REPLICATION 1)`。
```
> SELECT * FROM "transportation"."three_weeks"."average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
2016-08-28T08:00:00Z   13.75
```

#### 示例3 使用后向引用(backreferencing)自动缩小取样并存储到新的数据库中
```
CREATE CONTINUOUS QUERY "cq_basic_br" ON "transportation"
BEGIN
  SELECT mean(*) INTO "downsampled_transportation"."autogen".:MEASUREMENT FROM /.*/ GROUP BY time(30m),*
END
```
使用后向引用语法自动缩小取样并存储到新的数据库中。语法 `:MEASUREMENT` 用来指代后面的表，而 `/.*/`则是分别查询所有的表。这句CQ的含义就是每30分钟自动查询`transportation`的所有表(这里只有`bus_data`一个表)，并将30分钟内数字字段(`passengers和complaints`)求平均值存储到新的数据库 `downsampled_transportation`中。

最终结果如下：

```
> SELECT * FROM "downsampled_transportation."autogen"."bus_data"
name: bus_data
--------------
time                   mean_complaints   mean_passengers
2016-08-28T07:00:00Z   9                 6.5
2016-08-28T07:30:00Z   9                 7.5
2016-08-28T08:00:00Z   8                 11.5
2016-08-28T08:30:00Z   7                 16
```

#### 示例4 自动缩小取样以及配置CQ的时间范围
```
CREATE CONTINUOUS QUERY "cq_basic_offset" ON "transportation"
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h,15m)
END
```

与前面几个示例不同的是，这里的`GROUP BY time(1h, 15m)`指定了一个时间偏移，也就是说  `cq_basic_offset`执行的时间不再是整点，而是往后偏移15分钟。执行流程如下:
```
At 8:15 cq_basic_offset 执行查询的时间范围 time >= '7:15' AND time < '8:15'.
name: average_passengers
------------------------
time                   mean
2016-08-28T07:15:00Z   7.75
At 9:15 cq_basic_offset 执行查询的时间范围 time >= '8:15' AND time < '9:15'.
name: average_passengers
------------------------
time                   mean
2016-08-28T08:15:00Z   16.75
```
最终结果:
```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:15:00Z   7.75
2016-08-28T08:15:00Z   16.75
```
### 高级语法
InfluxDB连续查询的高级语法如下：
```
CREATE CONTINUOUS QUERY <cq_name> ON <database_name>
RESAMPLE EVERY <interval> FOR <interval>
BEGIN
  <cq_query>
END
```
与基本语法不同的是，多了`RESAMPLE`关键字。高级语法里CQ的执行时间和查询时间范围则与`RESAMPLE`里面的两个`interval`有关系。

高级语法中CQ以EVERY interval的时间间隔执行，执行时查询的时间范围则是FOR interval来确定。如果FOR interval为2h，当前时间为17:00，则查询的时间范围为`15:00-16:59.999999`。RESAMPLE的EVERY和FOR两个关键字可以只有一个。

示例的数据表如下，比之前的多了几条记录为了示例3和示例4的测试:
```

name: bus_data
--------------
time                   passengers
2016-08-28T06:30:00Z   2
2016-08-28T06:45:00Z   4
2016-08-28T07:00:00Z   5
2016-08-28T07:15:00Z   8
2016-08-28T07:30:00Z   8
2016-08-28T07:45:00Z   7
2016-08-28T08:00:00Z   8
2016-08-28T08:15:00Z   15
2016-08-28T08:30:00Z   15
2016-08-28T08:45:00Z   17
2016-08-28T09:00:00Z   20

```

#### 示例1 只配置执行时间间隔
```
CREATE CONTINUOUS QUERY "cq_advanced_every" ON "transportation"
RESAMPLE EVERY 30m
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h)
END
```
这里配置了30分钟执行一次CQ，没有指定FOR interval，于是查询的时间范围还是`GROUP BY time(1h)`指定的一个小时，执行流程如下：
```
At 8:00, cq_advanced_every 执行时间范围 time >= '7:00' AND time < '8:00'.
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
At 8:30, cq_advanced_every 执行时间范围 time >= '8:00' AND time < '9:00'.
name: average_passengers
------------------------
time                   mean
2016-08-28T08:00:00Z   12.6667
At 9:00, cq_advanced_every 执行时间范围 time >= '8:00' AND time < '9:00'.
name: average_passengers
------------------------
time                   mean
2016-08-28T08:00:00Z   13.75
```
需要注意的是，这里的 8点到9点这个区间执行了两次，第一次执行时时8:30，平均值是 `(8+15+15）/ 3 = 12.6667`，而第二次执行时间是9:00，平均值是 `(8+15+15+17) / 4=13.75`，而且最后第二个结果覆盖了第一个结果。InfluxDB如何处理重复的记录可以参见这个文档。

最终结果：
```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   7
2016-08-28T08:00:00Z   13.75
```
#### 示例2 只配置查询时间范围
```
CREATE CONTINUOUS QUERY "cq_advanced_for" ON "transportation"
RESAMPLE FOR 1h
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(30m)
END
```
只配置了时间范围，而没有配置EVERY interval。这样，执行的时间间隔与GROUP BY time(30m)一样为30分钟，而查询的时间范围为1小时，由于是按30分钟分组，所以每次会写入两条记录。执行流程如下：
```
At 8:00 cq_advanced_for 查询时间范围：time >= '7:00' AND time < '8:00'.
写入两条记录。
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   6.5
2016-08-28T07:30:00Z   7.5
At 8:30 cq_advanced_for 查询时间范围：time >= '7:30' AND time < '8:30'.
写入两条记录。
name: average_passengers
------------------------
time                   mean
2016-08-28T07:30:00Z   7.5
2016-08-28T08:00:00Z   11.5
At 9:00 cq_advanced_for 查询时间范围：time >= '8:00' AND time < '9:00'.
写入两条记录。
name: average_passengers
------------------------
time                   mean
2016-08-28T08:00:00Z   11.5
2016-08-28T08:30:00Z   16
```
需要注意的是，`cq_advanced_for`每次写入了两条记录，重复的记录会被覆盖。

最终结果：
```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T07:00:00Z   6.5
2016-08-28T07:30:00Z   7.5
2016-08-28T08:00:00Z   11.5
2016-08-28T08:30:00Z   16
```
#### 示例3 同时配置执行时间间隔和查询时间范围

```
CREATE CONTINUOUS QUERY "cq_advanced_every_for" ON "transportation"
RESAMPLE EVERY 1h FOR 90m
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(30m)
END
```

这里配置了执行间隔为1小时，而查询范围90分钟，最后分组是30分钟，每次插入了三条记录。执行流程如下：
```
At 8:00 cq_advanced_every_for 查询时间范围 time >= '6:30' AND time < '8:00'.
插入三条记录
name: average_passengers
------------------------
time                   mean
2016-08-28T06:30:00Z   3
2016-08-28T07:00:00Z   6.5
2016-08-28T07:30:00Z   7.5
At 9:00 cq_advanced_every_for 查询时间范围 time >= '7:30' AND time < '9:00'.
插入三条记录
name: average_passengers
------------------------
time                   mean
2016-08-28T07:30:00Z   7.5
2016-08-28T08:00:00Z   11.5
2016-08-28T08:30:00Z   16
```
最终结果：
```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T06:30:00Z   3
2016-08-28T07:00:00Z   6.5
2016-08-28T07:30:00Z   7.5
2016-08-28T08:00:00Z   11.5
2016-08-28T08:30:00Z   16
```
#### 示例4 配置查询时间范围和FILL填充
```
CREATE CONTINUOUS QUERY "cq_advanced_for_fill" ON "transportation"
RESAMPLE FOR 2h
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h) fill(1000)
END
```
在前面值配置查询时间范围的基础上，加上FILL填充空的记录。执行流程如下：
```
At 6:00, cq_advanced_for_fill 查询时间范围：time >= '4:00' AND time < '6:00'，没有数据，不填充。

At 7:00, cq_advanced_for_fill 查询时间范围：time >= '5:00' AND time < '7:00'. 写入两条记录，没有数据的时间点填充1000。
------------------------
time                   mean
2016-08-28T05:00:00Z   1000          <------ fill(1000)
2016-08-28T06:00:00Z   3             <------ average of 2 and 4

[…] At 11:00, cq_advanced_for_fill 查询时间范围：time >= '9:00' AND time < '11:00'.写入两条记录，没有数据的点填充1000。
name: average_passengers
------------------------
2016-08-28T09:00:00Z   20            <------ average of 20
2016-08-28T10:00:00Z   1000          <------ fill(1000)     

At 12:00, cq_advanced_for_fill 查询时间范围：time >= '10:00' AND time < '12:00'。没有数据，不填充。
```
最终结果:
```
> SELECT * FROM "average_passengers"
name: average_passengers
------------------------
time                   mean
2016-08-28T05:00:00Z   1000
2016-08-28T06:00:00Z   3
2016-08-28T07:00:00Z   7
2016-08-28T08:00:00Z   13.75
2016-08-28T09:00:00Z   20
2016-08-28T10:00:00Z   1000
```