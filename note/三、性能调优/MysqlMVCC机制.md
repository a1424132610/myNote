## 原理：
```
mysql使用了undo日志、事务id字段以及roll_pointer字段来实现的数据回滚。

修改：
    Mysql会给每个表新增两个字段 trx_id 事务id以及 roll_pointer 回滚指针。如果mysql插入或者修改（删除）一条数据就会创建一个事务id；如果执行修改语句会先插入数据并且
    删除原来老的数据并且将其放到undo日志中，然后新的数据roll_pointer字段就存放undo日志中的指针。

查询：
    当mysql开启一个查询时 遇到的select语句会创建一个 read-view一致性视图，这个视图里面保存 一个未提交事务的数组（数组中最小的id是min_id）以及已经创建的（包括已经提交和未提交）最大的事务id（max_id）。
    当mysql遇到查询时，会把查询的数据结果跟read-view里面的事务id进行对比。read-view是用来解决可重复读的隔离机制
    对于全表只会生成一次。
    版本链对比规则：
        mysql会根据事务id进行划分成 已经提交事务、未提交事务与提交事务、未开始事务
        trx_id < min_id：证明是已经提交事务，那么证明数据可见
        min_id <= trx_id <= max_id：分为两种情况
            如果row的trx_id存在在read-view数组里面，那么就证明还没有提交事务，数据不可见
            如果row的trx_id不存在在read-view数组里面，那么这个版本是已经提交了的事务生成的，可见
        max_id < trx_id: 表示还未开始的事务生成的，不可见
        
删除：update的特殊情况，会将版本链中最新的数据复制一份，然后将trx_id修改为删除操作的trx_id，同时将数据头信息（record header）里面的（delete_flag）标记上true代表删除
参考：https://www.cnblogs.com/luozhiyun/p/11216287.html
```