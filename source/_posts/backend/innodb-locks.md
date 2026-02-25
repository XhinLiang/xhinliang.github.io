title: MySQL InnoDB Locks
date: 2021-09-13
tags: [Backend,MySQL,InnoDB,Lock]
categories: Backend
toc: true
---

![MySQL](/uploads/persister-innodb-locks-MySQL-1200px-MySQL.svg.png)

InnoDB is a storage engine for MySQL. 
After more than ten years of evolution, InnoDB has become the most widely used storage engine for internet applications.

There are many articles about InnoDB locks, and today I want to revisit the topic.

## HOW TO VIEW LOCKS IN MYSQL

To learn the locks effectively, you should learn to show the locks of current database.

Typing this command in mysql client, and you will get all of the locks that InnoDB is currently holding.

```sql
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140616303283528:1157:140616462868960
ENGINE_TRANSACTION_ID: 4492
            THREAD_ID: 58
             EVENT_ID: 75
        OBJECT_SCHEMA: xhinliang_test --- the database you used
          OBJECT_NAME: locking_test --- the table which of the lock occur
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140616462868960
            LOCK_TYPE: TABLE --- TABLE or RECORD, we will explain it below
            LOCK_MODE: IX --- several options, we will explain it below
          LOCK_STATUS: GRANTED --- whether the lock is granted or waiting?
            LOCK_DATA: NULL -- index of the lock using
```

As shown above, we can inspect current locks in the database. There are several key fields we should pay attention to.

- INDEX_NAME The name of the locked index, always non-NULL for innoDB tables.
- LOCK_TYPE 
- LOCK_MODE
- LOCK_STATUS
- LOCK_DATA

From these five fields, we can get almost all the lock details we care about.

### LOCK_TYPE

LOCK_TYPE is the first and easiest field to understand. It can be TABLE or RECORD and indicates the scope affected by the lock.

### LOCK_MODE

LOCK_MODE is the most difficult field in this post, people always make it mixed with LOCK_TYPE.

LOCK_MODE has several options

- IX -> Intention Exclusive Lock
- IS -> Intention Share Lock
- X,REC_NOT_GAP -> Exclusive Record Lock
- X,GAP -> Exclusive Gap Lock
- X -> Exclusive Next-Key Lock
- S,REC_NOT_GAP -> Share Record Lock
- S,GAP -> Share Gap Lock
- S -> Share Next-Key Lock

### LOCK_STATUS

LOCK_STATUS shows the acquisition state of this lock: GRANTED or WAITING.
When the LOCK_STATUS is GRANTED means that the session acquired this lock, otherwise means that the session of this lock is waiting.

### LOCK_DATA

LOCK_DATA indicates which rows are this lock affected for.

For Example, when there are a table named `child` and have some record initially.

```
mysql> desc child;
+-------+------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+-------+------+------+-----+---------+-------+
| id    | int  | NO   | PRI | NULL    |       |
+-------+------+------+-----+---------+-------+
1 row in set (0.00 sec)

mysql> select * from child
    -> ;
+-----+
| id  |
+-----+
|  89 |
|  90 |
| 102 |
| 151 |
+-----+
4 rows in set (0.00 sec)
```

Then we start a session and lock a non-existent row.
```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from child where id = 100 for update;
Empty set (0.00 sec)
```

When we query the locks of this database, we can see that the (90, 102) has been locked as an Exclusive Gap Lock.

```
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140459538615624:1158:140458948850848
ENGINE_TRANSACTION_ID: 5640
            THREAD_ID: 49
             EVENT_ID: 25
        OBJECT_SCHEMA: xhinliang_test
          OBJECT_NAME: child
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140458948850848
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140459538615624:3:4:3:140459473245216
ENGINE_TRANSACTION_ID: 5640
            THREAD_ID: 49
             EVENT_ID: 25
        OBJECT_SCHEMA: xhinliang_test
          OBJECT_NAME: child
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140459473245216
            LOCK_TYPE: RECORD
            LOCK_MODE: X,GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 102
2 rows in set (0.01 sec)
```
