## bin_log

- binlog只会记录数据库结构变更以及表数据修改的二进制日志，不会记录SELECT和SHOW操作，主要用于数据恢复和主从同步，在事务提交时写入。
### 记录模式

STATEMENT/ROW/MIXED

STATEMENT记录每次执行的sql，日志小，但是在使用`now()`这种函数时，在使用binlog的时候获取的时间可能与先前执行sql语句时不同。

针对这种情况，可以在sql前先set times-tamp


## redo_log

用于数据恢复，保证数据二点一致性和持久性，当Mysql发生修改时，redo_log会把这些操作写入磁盘，通过重放redo_log就可以恢复数据库

## undo_log

用于事务回滚，在发生事务回滚时，直接重放undo_log即可

