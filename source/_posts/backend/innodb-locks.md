title: MySQL InnoDB Locks
date: 2021-09-13
tags: [Backend,MySQL,InnoDB,Lock]
categories: Backend
toc: true
---

![MySQL](/uploads/persister-innodb-locks-MySQL-1200px-MySQL.svg.png)

InnoDB is MySQL’s transactional storage engine. After more than a decade of development, it has become the default choice for most production workloads.

There are plenty of good deep-dives on InnoDB locks already; this post is a short refresher focused on how to inspect locks in a running database.

## HOW TO VIEW LOCKS IN MYSQL

To understand locking behavior, start by learning how to inspect the locks currently held in your database.

Run the following in the MySQL client to list all locks InnoDB is currently holding:

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

As shown above, you can inspect the current locks in the database. There are several key fields worth focusing on:

- `INDEX_NAME`: The index involved in the lock. For table locks, this can be `NULL`.
- `LOCK_TYPE`: Whether the lock is a `TABLE` lock or a `RECORD` lock.
- `LOCK_MODE`: The specific lock mode (e.g., intention locks, record locks, gap locks).
- `LOCK_STATUS`: Whether the lock is `GRANTED` or `WAITING`.
- `LOCK_DATA`: For record locks, what record (or boundary) the lock refers to.

From these fields you can usually tell what is locked, what kind of lock it is, and why a transaction is blocked.

### LOCK_TYPE

LOCK_TYPE is the easiest field to interpret. It is either `TABLE` or `RECORD` and tells you the scope of the lock.

### LOCK_MODE

LOCK_MODE is the trickiest field in this post; it’s also the one people most often confuse with LOCK_TYPE.

LOCK_MODE has several common values:

- IX -> Intention Exclusive Lock
- IS -> Intention Share Lock
- X,REC_NOT_GAP -> Exclusive Record Lock
- X,GAP -> Exclusive Gap Lock
- X -> Exclusive Next-Key Lock
- S,REC_NOT_GAP -> Share Record Lock
- S,GAP -> Share Gap Lock
- S -> Share Next-Key Lock

### LOCK_STATUS

LOCK_STATUS shows whether the lock is currently held or still being waited on:

- `GRANTED`: the session has acquired the lock.
- `WAITING`: the session is blocked, waiting to acquire it.

### LOCK_DATA

LOCK_DATA indicates what data this lock refers to.

For example, suppose we have a table named `child` with the following rows:

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

Now, in one session, try to lock a row that doesn’t exist:
```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from child where id = 100 for update;
Empty set (0.00 sec)
```

If we inspect `performance_schema.data_locks` now, we can see that the gap between `(90, 102)` is locked as an exclusive gap lock.

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

Once you get used to reading these fields, debugging lock contention becomes much more mechanical: take a snapshot of `performance_schema.data_locks`, identify `WAITING` locks, and then find the corresponding `GRANTED` locks on the same object/index.
