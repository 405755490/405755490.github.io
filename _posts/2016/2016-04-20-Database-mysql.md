---
layout: post
title:      "Mysql操作"
subtitle:   "数据库 -- Mysql"
tags:       [MYSQL]
mathjax:    false
date:       2016-04-20
author:     "Undefined"
excerpt:	"数据库操作"
---



## 1.C程序调用MYSQL

``` mysql
所需头文件： #include <mysql/mysql.h>
功能：  获得或初始化一个MYSQL结构
函数原型： MYSQL *mysql_init(MYSQL *mysql)
函数返回值： 一个被始化的MYSQL*句柄
备注：  在内存不足的情况下，返回NULL

所需头文件： #include <mysql/mysql.h>
函数功能： 关闭一个服务器连接，并释放与连接相关的内存
函数原型： void mysql_close(MYSQL *mysql);
函数传入值： MYSQL:类型的指针
函数返回值： 无

所需头文件： #include <mysql/mysql.h>
函数功能： 连接一个MySQL服务器
函数原型： MYSQL * mysql_connect(MYSQL *mysql,const char *host,const char *user,const char *passwd);
函数传入值： mysql表示一个现存mysql结构的地址
  host表示MYSQL服务器的主机名或IP
  user表示登录的用户名
  passwd表示登录的密码
函数返回值： 如果连接成功，一个MYSQL *连接句柄：如果连接失败，NULL
备注：  该函数不推荐，使用mysql_real_connect()代替

所需文件： #include <mysql/mysql.h>
函数功能： MYSQL *mysql_real_connect(MYSQL *mysql,const char *host,const char *user,const char *passwd,const char *db,
					unsigned int port,const char *unix_socket,unsigned int client_flag);
函数传入值： mysql表示一个现存mysql结构的地址
  host表示MYSQL服务器的主机名或IP
  user表示登录的用户名
  passwd表示登录的密码
  db表示要连接的数据库
  port表示MySQL服务器的TCP/IP端口
  unix_socket表示连接类型
  client_flag表示MySQL运行ODBC数据库的标记
函数返回值： 如果连接成功，一个MYSQL*连接句柄：如果连接失败，NULL

所需头文件： #include <mysql/mysql.h>
函数功能： 返回最新的UPDATE，DELETE或INSERT查询影响的行数
函数传入值： MYSQL:类型指针
函数返回值： 大于零的一个整数表示受到影响或检索出来的行数。零表示没有区配查序中WHERE子句的记录
				或目前还没有查询被执行;-1表示查询返回一个错误，或对于一个SELECT查询

所需头文件： #include <mysql/mysql.h>
函数功能： 对指定的连接执行查询
函数原型： int mysql_query(MYSQL *mysql,const char *query);
函数传入值： query表示执行的SQL语句
函数返回值： 如果查询成功，为零，出错为非零。
相关函数： mysql_real_query

所需头文件： #include <mysql/mysql.h>
函数功能： 为无缓冲的结果集获得结果标识符
函数原形： MYSQL_RES *mysql_use_result(MYSQL *mysql);
函数传入值： MYSQL:类型的指针
函数返回值： 一个MYSQL_RES结果结构，如果发生一个错误发NULL

#incluee <mysql/mysql.h>
检索一个结果集合的下一行
MYSQL_ROW mysql_fetch_row(MYSQL_RES *result);
MYSQL_RES:结构的指针
下一行的一个MYSQL_ROW结构。如果没有更多的行可检索或如果出现一个错误，NULL

#include <mysql/mysql.h>
返回指定结果集中列的数量
unsigned int mysql_num_fields(MYSQL_RES *res);
MYSQL_RES 结构的指针
结果集合中字段数量的一个无符号整数

#include <mysql/mysql.h>
创建一个数据库
int mysql_create_db(MYSQL *mysql,const char *db);
MYSQL：类型的指针
db:要创建的数据库名
如果数据库成功地被创建，返回零，如果发生错误，为非零。
#include <mysql/mysql.h>
选择一个数据库
int mysql_select_db(MYSQL *mysql,const char *db);
MYSQL：类型的指针
db:要创建的数据库名
如果数据库成功地被创建，返回零，如果发生错误，为非零。
```

[查看更多](http://www.cnblogs.com/nliao/archive/2010/09/09/1822660.html)


---

## 2.MYSQL的常规用法


+ ① 时间戳转换成日期  `FROM_UNIXTIME`(invest_time,'%Y年%m月%d') 

+ ② 把日期转换为时间戳`UNIX_TIMESTAMP`，和 FROM_UNIXTIME 正好相反  SELECT UNIX_TIMESTAMP('2015-04-29')


---

## 3.MYSQL的示例

**1、自行创建测试数据**

+ 创建表

``` mysql
create table class(cid int ,caption char(50));
create table student(sid int ,sname char(30) , gender char(5) , class_id  char(5));
create table teacher(tid int primary key auto_increment , tname char(50));
create table course(cid int primary key auto_increment ,cname char(30),tearch_id int);
create table score(sid int ,student_id int ,corse_id int ,number int );
```

+ 修改表的属性

+ 修改表class中的cid为主键  alter table class add primary key(cid);

+ 修改student表中的字段属性 alter table student modify  class_id int ;

+ 修改外键属性 

	alter table student add   constraint   FK_classid_studentid  
	foreign key(class_id) references  class(cid); 

+ 插入数据

``` mysql
insert into class values(1 ,"三年二班");
insert into class values(2, "一年三班");
insert into class values(3, "三年一班");
```

**2、查询“生物”课程比“体育”课程成绩高的所有学生的学号**

``` mysql
select A.student_id,sw,ty from
	(select student_id,number as sw from score left join course on score.corse_id = course.cid where course.cname = '生物') as A
left join
	(select student_id,number  as ty from score left join course on score.corse_id = course.cid where course.cname = '体育') as B
on A.student_id = B.student_id where sw > if(isnull(ty),0,ty);

```


[查看更多](http://www.cnblogs.com/pythonxiaohu/p/5749864.html#)



## 4.MYSQL的注意事项

1.在创建好表后，再建立外检关联.注意字段类型是否一致，且被关联的字段`必须是主键`（cid必须要是class的主键）

`alter table student add constraint "class_stu" foreign key(class_id) REFERENCES class(cid); `

注意两个表中的字段`是否是类型一致`。 `即class_id 跟cid 要类型一致`。要不然报错

ERROR 1215 (HY000):` Cannot add foreign key constraint`



**2.数据库`事务的4个特性`**

+ 1. 原子性（Atomicity）

+ 2. 一致性（Consistancy）

+ 3. 分离性、独立性（Isolation）

+ 4. 持久性（Durability）



3.MySQL5.5以后默认`使用InnoDB存储引擎`；若要修改默认引擎，可以修改配置文件中的default-storage-engine；可以通过：

`show variables like 'default_storage_engine'`

查看当前数据库的默认引擎。


## 5.附录

1. [Mysql全程讲解](http://blog.csdn.net/aisemi/article/details/55271641?locationNum=10&fps=1)



