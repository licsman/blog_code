---
title: 使用Oracle GoldenGate同步oracle变化到kafka
date: 2020-09-23 16:16:33
categories: cdc
tags: [数据同步,oracle]
---

# 1. 版本说明

- **Oracle版本**：`11.2.0.1.0`
- **Linux版本**：`Centos 7.3`
- **Oracle GoldenGate版本**：`122022_fbo_ggs_Linux_x64_shiphome`
- **Oracle GoldenGate for Big Data版本**：`12.3.2.1.0`
- **官方文档地址**：https://www.oracle.com/middleware/technologies/goldengate.html

<!--more-->

# 2. Oracle环境准备工作

1. 启动oracle监听器

   ```
   lsnrctl start
   ```

2. sqlplus命令行登录方法

   ```sql
   sqlplus / as sysdba
   ```

3. 查看数据库版本信息

   ```sql
   select * from v$version;
   ```

4. 确认会话时区是否正确（如果是北京时间的话就是+8）

   ```sql
   select sessiontimezone from dual;
   select tz_offset(sessiontimezone), tz_offset(dbtimezone) from dual;
   ```

5. 设置时区为北京时间

   ```sql
   alter database set time_zone='+8:00';
   ```

6. 打开控制文件

   ```shell
   startup mount;
   ```

7. 查看语言ogg设置

   ```sql
   select * from v$nls_parameters;
   select userenv('language') from dual;
   ```

8. 查看是否归档

   ```properties
   archive log list;
   
   Database log mode              No Archive Mode     #数据库日志模式             非存档模式
   Automatic archival             Disabled            #自动存档             禁用
   Archive destination            USE_DB_RECOVERY_FILE_DEST    #存档终点            USE_DB_RECOVERY_FILE_DES
   Oldest online log sequence     13            #最早的联机日志序列        13
   Current log sequence           15            #当前日志序列             15
   ```

9. 开启数据库日志归档模式

   ```sql
   alter database archivelog;
   ```

10. 调整归档文件大小(为了压测)

    ```sql
    alter system set db_recovery_file_dest_size = 50g;
    
    alter system set db_recovery_file_dest = '/home/oracle/app/oradata/recovery_area' scope=spfile;
    ```

11. 查看闪回恢复区的信息

    ```sql
    show parameter db_recove;
    ```

12. 查看当前数据库强制日志模式的状态

    ```sql
    select force_logging from v$database;
    ```

13. 强制记录日志，确保Oracle无论什么操作都进行redo的写入

    ```sql
    alter database force logging;
    ```

14. 打开数据库

    ```sql
    alter database open;
    ```

15. 将OGG绑定到DB中

    ```sql
    alter system set enable_goldengate_replication=true scope=both;
    ```

16. 查看最小补全日志是否已经开启

    ```sql
    select supplemental_log_data_min FROM v$database;
    ```

17. 开启最小补全日志功能

    ```sql
    alter database add supplemental log data (PRIMARY KEY,UNIQUE) COLUMNS;
    ```

18. 查看是否开启ddl记录功能,默认是不开启的

    ```sql
    show parameter enable_ddl_logging;
    ```

19. 开启DDL日志记录功能

    ```sql
    alter system set ENABLE_DDL_LOGGING=TRUE;
    ```

20. 关闭回收站里的每个会话,只适用于当前会话窗口(回收站是数据文件里面的动态参数，需要添加spoce=spfile，重启数据库才能修改成功)

    ```sql
    alter system SET recyclebin = OFF scope=spfile;
    ```

    



# 2. 创建表空间和用户

1. 创建namespace

   ```sql
   create tablespace ogg2 datafile '/home/oracle/app/oradata/ogg/ogg2.dbf' size 100m autoextend on next 10m;
   ```

2. 创建用户并指定表空间以及临时表空间

   ```sql
   create user ogg2 identified by ogg2 default tablespace ogg2 temporary tablespace TEMP quota unlimited on ogg2;
   ```

3. 给刚刚创建好的用户授权

   ```sql
   grant dba to ogg2;
   grant select any table to ogg2;
   GRANT CREATE SESSION TO ogg2;
   GRANT connect, resource TO ogg2;
   ```

# 4. OGG的安装与配置-silent

1. 修改oggcore.rsp文件

   ```sh
   vi /home/oracle/app/install/response/oggcore.rsp
   
   #设置ogg对应的数据库版本为11g
   INSTALL_OPTION=ORA11g
   #设置ogg的安装目录
   SOFTWARE_LOCATION=/home/oracle/app/ogg
   #设置mgr默认不初始化启动
   START_MANAGER=false
   #设置组名
   UNIX_GROUP_NAME=oinstall
   ```

2. 静默安装

   ```shell
   /home/oracle/app/install/runInstaller -silent -responseFile /home/oracle/app/install/response/oggcore.rsp
   ```

3. root用户执行（安装补丁包）

   ```shell
   rpm -ivh oracle-instantclient19.3-basic-19.3.0.0.0-1.x86_64.rpm
   /home/oracle/app/oracle/product/11.2.0/dbhome_1/lib
   ```

4. 登录oracle用户，添加环境变量

   ```shell
   export LD_LIBRARY_PATH=/usr/lib/oracle/19.3/client64/lib:$ORACLE_HOME/lib:/usr/lib
   source ~/.bash_profile
   
   ln -s /usr/lib/oracle/19.3/client64/lib/libnnz19.so /usr/lib/oracle/19.3/client64/lib/libnnz11.so
   ##ln -s /usr/lib/oracle/19.3/client64/lib/libociei.so /usr/lib/oracle/19.3/client64/lib/libociei.so
   ##ln -s /usr/lib/oracle/19.3/client64/lib/libclntsh.so.11.1 $ORACLE_HOME/lib/libclntsh.so.11.1
   ##ln -s /usr/lib/oracle/19.3/client64/lib/libclntshcore.so $ORACLE_HOME/lib/libclntshcore.so
   ```

# 5. 源端的配置

为11g创建ogg checkpoint需要的表（使用Oracle用户进行操作）

```shell
##必须到ogg目录,即与ddl_setup.sql文件同目录下
cd /home/oracle/app/ogg/
sqlplus / as sysdba

##输入刚刚上方创建的ogg用户名称:ogg2
@marker_setup.sql
@ddl_setup.sql
@role_setup.sql
GRANT GGS_GGSUSER_ROLE TO ogg;
@ddl_enable.sql
@?\rdbms\admin\dbmspool.sql
@ddl_pin ogg;
@ddl_staymetadata_on.sql
@marker_status.sql
--@chkpt_ora_create.sql
```

注意：这里如果报错：`INITIALSETUP used, but DDLREPLICATION package exists under different schema (OGG)`

需要移除之前的设置：

> 参考文档：https://docs.oracle.com/goldengate/1212/gg-winux/GIORA/uninstall.htm#GIORA466
>
> 1. Log on as the system administrator or as a user with permission to issue Oracle GoldenGate commands and delete files and directories from the operating system.
>
> 2. Run GGSCI from the Oracle GoldenGate directory.
>
> 3. Stop all Oracle GoldenGate processes.
>
>    ```
>    STOP ER *
>    ```
>
> 4. Log in to SQL*Plus as a user that has `SYSDBA` privileges.
>
> 5. Disconnect all sessions that ever issued DDL, including those of Oracle GoldenGate processes, SQL*Plus, business applications, and any other software that uses Oracle. Otherwise the database might generate an ORA-04021 error.
>
> 6. Run the `ddl_disable` script to disable the DDL trigger.
>
> 7. Run the `ddl_remove` script to remove the Oracle GoldenGate DDL trigger, the DDL history and marker tables, and other associated objects. This script produces a `ddl_remove_spool.txt` file that logs the script output and a `ddl_remove_set.txt` file that logs environment settings in case they are needed for debugging.
>
> 8. Run the `marker_remove` script to remove the Oracle GoldenGate marker support system. This script produces a `marker_remove_spool.txt` file that logs the script output and a `marker_remove_set.txt` file that logs environment settings in case they are needed for debugging.





# 6. 配置EXTRACT抽取脚本

## 6.1 ggsci初始化

- 通过ogg命令行模式登录orcl数据库

  - ```
    DBLOGIN USERID ogg2@orcl, PASSWORD ogg2
    ```

- 初始化,创建OGG的子目录

  - ```
    create subdirs
    ```

- 创建检查点表

  - ```
    add CHECKPOINTTABLE ogg.checkpointtable
    ```

- 编辑mgr

  - ```
    edit param mgr
    ```

  - ```
    PORT 7809
    --动态端口范围
    DYNAMICPORTLIST 7810-7860
    --指定在mgr启动时自动启动那些进程
    --自动启动所有的EXTRACT进程
    AUTOSTART EXTRACT *
    --指定在mgr可以定时重启那些进程.可以在网络中断等故障恢复后自动重起,避免人工干预
    --每隔3分钟尝试启动一次,尝试5次,等待10分钟后再尝试
    AUTORESTART EXTRACT *, RETRIES 5, WAITMINUTES 3, RESETMINUTES 10
    --定义自动删除过时的队列以节省硬盘空间.本处设置表示对于超过3天的trail文件进行删除。
    PURGEOLDEXTRACTS ./dirdat/*, usecheckpoints, minkeepdays 3
    --每隔1小时检查一次传输延迟情况
    LAGREPORTHOURS 1
    --传输延时超过30分钟将写入错误日志
    LAGINFOMINUTES 30
    --传输延时超过45分钟将写入警告日志
    LAGCRITICALMINUTES 45
    ```

- 创建ogg的检查点(checkpoint)表

  - ```
    ADD CHECKPOINTTABLE ogg.checkpointtable
    ```

- 设置数据库表级补全日志,注意需要在最小补全日志打开的情况下才起作用

  - ```
    add trandata TEST001.*
    info trandata TEST001.*
    ```

## 6.2 全量抽取的配置（简易）

编辑全量(init load)抽取进程配置文件

```shell
edit params send
```

配置全量抽取脚本

```sql
--SOURCEISTABLE
EXTRACT send
--SETENV (NLS_LANG = "AMERICAN_AMERICA.AL32UTF8")
--SETENV (ORACLE_SID=orcl)
--SETENV (ORACLE_HOME=/home/oracle/app/oracle/product/11.2.0/dbhome_1)
USERID ogg2, PASSWORD ogg2
RMTHOST 172.20.3.191, MGRPORT 7809
--注意:RMTFILE参数指定的文件只支持2位字符,超过2位字符则replicat则无法识别
--GROUP后面的名字receive为接收方(即ogg bigdata)设置的REPLICAT的name,该REPLICAT不需要手动启动,由当前的ogg发送给接收方后自行启动
RMTFILE ./dirdat/bb,maxfiles 100,megabytes 1024,purge
--需要采集的表,注意的是必须由;结尾
TABLE "test001".*;
```

增加send为extract(sourceIsTable全量)类型

```
ADD EXTRACT send,sourceistable
```



## 6.3增量抽取的配置（较复杂）

编辑增量抽取脚本：`exta1`

```
edit params exta1
```

配置脚本：

```sql
--指定是EXTRACT类型和名称exta1
EXTRACT exta1
--如果系统中存在多个数据库有时候会用参数SETENV设置ORACLE_HOME、ORACLE_SID等
SETENV (ORACLE_SID = "orcl")
--设置字符集环境变量为UTF8
SETENV (NLS_LANG = "american_america.AL32UTF8")
--指定所要登陆的数据库名称,用户名和密码.对于oracle无需指定sourcedb,直接指定用户名和密码即可
USERID ogg2, PASSWORD ogg2
--目标数据库服务器地址和GG服务端口号
RMTHOST 172.20.3.191, MGRPORT 7809
--远程队列的位置,由于oggbigdata和ogg目录未必一样,所以最好写相对路径
RMTTRAIL ./dirdat/ll
--捕捉所有ddl数据
DDL INCLUDE ALL
--复制delete操作，缺省复制(不复制是IGNOREDELETES)
GETDELETES
--复制insert操作，缺省复制(不复制是IGNOREINSERTS)
GETINSERTS
--捕捉truncat数据(默认是不捕捉,IGNORETRUNCATES)
GETTRUNCATES
--让replicat在同步DDL语句时若出现问题，将该问题的详细情况记录到该replicat的report 文件中，以便找出DDL复制失败的root cause
DDLOPTIONS REPORT
--是否在队列中写入前影像，缺省不复制(IGNOREUPDATEBEFORES)
GETUPDATEBEFORES
NOCOMPRESSDELETES
NOCOMPRESSUPDATES
--记录所有附加日志信息并写入trail文件中
LOGALLSUPCOLS
--控制抽取进程兼容更新操作的前镜像和后镜像信息并写入到一个trail文件中,一定要注意的是该参数必须在数据库版本是11.2.0.4或者12c版本
UPDATERECORDFORMAT COMPACT
--需要采集的表,注意的是必须由;结尾
TABLE "test001".*;
--TABLE SCHEMA_NAME.TABLE_NAME, COLS (COL1,COL2,COL3);
```

创建一个新的增量抽取进程，从redo日志中抽取数据

集成抽取(Integrated Capture)模式与传统抽取模式(Classic Capture)间的切换

```
ADD EXTRACT ext2hd, INTEGRATED TRANLOG, BEGIN NOW
```

传统抽取：

```
add extract exta1, tranlog, begin now
```

![image-20200722155020073](11G2Kafka.assets/image-20200722155020073.png)



创建一个抽取进程抽取的数据保存路径并与新建的抽取进程进行关联

注意目录要两个字符

```
add exttrail ./dirdat/ll, extract exta1, MEGABYTES 200
info exta1
```



# 7.目标端BigData的安装配置

## 7.1.OGG For BigData的安装

1. 将OGG_BigData_Linux_x64_12.3.2.1.0.zip传输到kafka端并解压缩

2. 导入环境变量

   ```shell
   vi /etc/profile
   
   export OGG_HOME=/opt/ogg
   export LD_LIBRARY_PATH=$JAVA_HOME/jre/lib/amd64:$JAVA_HOME/jre/lib/amd64/server:$JAVA_HOME/jre/lib/amd64/libjsig.so:$JAVA_HOME/jre/lib/amd64/server/libjvm.so:$ORACLE_HOME/lib:$OGG_HOME/lib
   export PATH=$OGG_HOME:$PATH
   
   source /etc/profile
   ```



## 7.2.BigData的初始化和接收进程配置

初始化OGG子目录

```
执行ggsci命令,进入ogg控制台
/opt/ogg/ggsci

#初始化,创建OGG的子目录
create subdirs
```

### 7.2.1.Mgr进程的配置

```sql
edit prarm mgr

PORT 7809
--动态端口范围
DYNAMICPORTLIST 7810-7860
--设置在目标端,允许被172.20.3网段的ip地址访问--主要用于被动启动REPLICAT
ACCESSRULE, PROG *, IPADDR 172.20.3.*, ALLOW
--每隔3分钟尝试启动一次,尝试5次,等待10分钟后再尝试
AUTORESTART EXTRACT *, RETRIES 5, WAITMINUTES 3, RESETMINUTES 10
--定期清理trail文件设置：本处设置表示对于超过3天的trail文件进行删除
PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 3
```

设置接收端的checkpoint

```sh
add CHECKPOINTTABLE ogg.checkpointtable
```



### 7.2.2.增量接收脚本的配置

```
edit params btkfk
```

编辑增量接收数据脚本

```sql
REPLICAT btkfk
--sourcedefs用于接收端和发送端表结构不一致的情况
---sourcedefs ./dirdef/oracle11g
--捕捉所有ddl数据
DDL INCLUDE ALL
--复制delete操作，缺省复制(不复制是IGNOREDELETES)
GETDELETES
--复制insert操作，缺省复制(不复制是IGNOREINSERTS)
GETINSERTS
--捕捉truncat数据(默认是不捕捉,IGNORETRUNCATES)
GETTRUNCATES
--让replicat在同步DDL语句时若出现问题，将该问题的详细情况记录到该replicat的report 文件中，以便找出DDL复制失败的root cause
DDLOPTIONS REPORT
--指定kafka配置文件
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafkaconnect.properties
--每隔30分钟报告一次从程序开始到现在的抽取进程或者复制进程的事物记录数，并汇报进程的统计信息
REPORTCOUNT EVERY 30 MINUTES, RATE
--控制每次发送的跟踪记录数为10000,GROUPTRANSOPS跟性能有很大的关系
GROUPTRANSOPS 10000
MAP "test001".*,TARGET "test001".*;
```

添加一个回放线程并与源端pump进程传输过来的trail文件关联，并使用checkpoint表确保数据不丢失

```
add replicat btkfk, exttrail ./dirdat/rr, checkpointtable ogg2.checkpointtable
```

![image-20200722161214020](11G2Kafka.assets/image-20200722161214020.png)



### 7.2.3.启动kafka消费进程消费数据

```shell
/opt/confluent/bin/kafka-topics --zookeeper localhost:2181 --list

/opt/confluent/bin/kafka-console-consumer --bootstrap-server localhost:9092 --topic ora-ogg-pdb-ORCLDB_OGG-avro --from-beginning
```

![image-20200722170235398](11G2Kafka.assets/image-20200722170235398.png)



### 7.2.4.全量接收脚本的配置

增加send为extract(sourceIsTable全量)类型

```
REPLICAT allRe
--HandleCollisions主要用于初始化场景用于解决主键冲突问题
--如果表上有LOB字段，且存的内容超过缓冲区域(2kb或4kb)的大小,则HANDLECOLLISIONS会排除此LOB字段的处理,所以,如果有表包含有lob字段,且长度超过2Kb,则不应该使用 HANDLECOLLISIONS
HandleCollisions
--AssumeTargetDefs源端和目标端表结构一致的时候使用,开启ddl同步必须结构一致(AssumeTargetDefs告訴OGG目標端和源端需要同步的表的結構完全一致)
AssumeTargetDefs
--指定kafka配置文件
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafkaconnect.properties
--定义discardfile文件位置，如果处理中有记录出错会写入到此文件中
DISCARDFILE ./dirrpt/ll.dsc,purge
MAP test001.*,TARGET test001.*;
```

添加脚本，bigdata端只运行一次

```
#SpecialRun表示只运行一次
add replicat allRe, exttrail ./dirdat/ll, SpecialRun
```



**Example说明：**

```
REPLICAT bb
--HandleCollisions主要用于初始化场景用于解决主键冲突问题
--如果表上有LOB字段，且存的内容超过缓冲区域(2kb或4kb)的大小,则HANDLECOLLISIONS会排除此LOB字段的处理,所以,如果有表包含有lob字段,且长度超过2Kb,则不应该使用 HANDLECOLLISIONS
HandleCollisions
--AssumeTargetDefs源端和目标端表结构一致的时候使用,开启ddl同步必须结构一致(AssumeTargetDefs告訴OGG目標端和源端需要同步的表的結構完全一致)
AssumeTargetDefs
--指定kafka配置文件
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafkaconnect.properties
--定义discardfile文件位置，如果处理中有记录出错会写入到此文件中
DISCARDFILE ./dirrpt/bb.dsc,purge
MAP test001.*,TARGET test001.*;

add replicat bb, exttrail ./dirdat/bb, SpecialRun
```





```shell
/opt/confluent/bin/kafka-topics --zookeeper localhost:2181 --list

PDB：

/opt/confluent/bin/kafka-avro-console-consumer --bootstrap-server localhost:9092 --topic  OGBK_OGSB_CAICDB --property schema.registry.url="http://localhost:8091"  --from-beginning

CDB:

/opt/confluent/bin/kafka-avro-console-consumer --bootstrap-server localhost:9092 --topic  OGBK_OGSB_CAICDB1 --property schema.registry.url="http://localhost:8091"  --from-beginning
```



# 8.Ogg12上同时捕获CDB&PDB数据的实践

```
edit params exta2

EXTRACT exta2
SETENV (NLS_LANG = "american_america.AL32UTF8")
USERID c##ogsb, PASSWORD ogsb
RMTHOST 172.20.3.191, MGRPORT 7809
RMTTRAIL ./dirdat/as
DDL INCLUDE ALL
GETDELETES
GETINSERTS
GETTRUNCATES
DDLOPTIONS REPORT

LOGALLSUPCOLS
UPDATERECORDFORMAT COMPACT
TABLE CDB$ROOT.C##OGSB.DEF;
TABLE ORCLDB.C##OGSB.ABC;



REGISTER EXTRACT exta2 DATABASE CONTAINER(orcldb)

ADD EXTRACT exta2, INTEGRATED TRANLOG, BEGIN NOW

add exttrail ./dirdat/as, extract exta2, MEGABYTES 200



DELETE TRANDATA CDB$ROOT.C##OGSB.*

delete trandata ORCLDB.c##OGSB.*

add trandata CDB$ROOT.C##OGSB.*

add trandata ORCLDB.c##OGSB.*

info trandata CDB$ROOT.C##OGSB.*

info trandata ORCLDB.c##OGSB.*

info SCHEMATRANDATA CDB$ROOT.cdb




REPLICAT ogbk1
--sourcedefs用于接收端和发送端表结构不一致的情况
---sourcedefs ./dirdef/oracle12c
--指定kafka配置文件
TARGETDB LIBFILE libggjava.so SET property=dirprm/kafkaconnect.properties
--每隔30分钟报告一次从程序开始到现在的抽取进程或者复制进程的事物记录数，并汇报进程的统计信息
REPORTCOUNT EVERY 30 MINUTES, RATE
--控制每次发送的跟踪记录数为10000
GROUPTRANSOPS 10000

MAP CDB$ROOT.C##OGSB.DEF, TARGET AAA.CDB.DB1;
MAP orcldb.C##OGSB.*,TARGET BBB.OGSB.*;



add replicat ogbk1, exttrail ./dirdat/as, checkpointtable CDB$ROOT.c##ogsb.checkpointtable
```