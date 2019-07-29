# 2. 一条SQL更新语句是如何执行的？

```sql
mysql> create table T(ID int primary key, c int);
mysql> update T set c=c+1 where ID=2;
```

![](./pic/2-update执行过程.png)