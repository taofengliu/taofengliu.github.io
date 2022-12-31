# InnoDB中的锁

> ## InnoDB有哪些锁？   
### 行锁   
&emsp;&emsp;InnoDB存储引擎中有如下两种行锁：
+ 共享锁(S Lock)  
+ 排他锁(X Lock)       

&emsp;&emsp;其相互兼容性如下表所示：         
|   | X | S |
|---|---|---|
| X |不兼容|不兼容|
| S |不兼容|兼容|   

### 意向锁
&emsp;&emsp;InnoDB存储引擎支持多粒度锁定，这种锁定允许事务在行级上的锁与表级上的锁同时存在。为了支持在不同粒度上的锁定，InnoDB引入了意向锁(Intention Lock)。意向锁将锁定对象分为多个层次，若希望在细粒度层次上加锁(行锁)，则需要先在粗粒度上加锁(意向锁)，可将意向锁理解为表级别的锁。     
&emsp;&emsp;InnoDB中有两种意向锁：   
+ 意向共享锁(IS Lock)   
+ 意向排他锁(IX Lock)    

&emsp;&emsp;其兼容性如下(表中的S和X为表级别的共享、排他锁)：    
| |IS|IX|S|X|
|-|-|-|-|-|
|IS|兼容|兼容|兼容|不兼容|
|IX|兼容|兼容|不兼容|不兼容|
|S|兼容|不兼容|兼容|不兼容|
|X|不兼容|不兼容|不兼容|不兼容|   

> ## 行锁    
### 行锁算法    
&emsp;&emsp;InnoDB中有三种行锁算法：     
+ Record Lock：单个行记录上的锁     
+ Gap Lock：间隙锁，锁定一个范围，但是不包括行记录本身    
+ Next-Key Lock：Gap Lock + Record Lock     

### 解决幻像(Phantom Problem)问题
&emsp;&emsp;幻像问题(Phantom Problem)在MySQL官方文档中给出了定义：***The so-called phantom problem occurs within a transaction when the same query produces different sets of rows at different times.***。大致意思是：***当同一条查询在不同的时间产生不同的结果集，所谓的Phantom Problem就会在事务中发生***。例如事务A按照一定搜索条件进行数据读取，期间事务B插入(删除)或更改了相同搜索条件的数据，事务A再次按照原先条件进行读取时，发现与第一次读取的结果出现了不同，好像出现了幻觉(Phantom)。在很多书中又给出了两个名词，分别是幻读(Phantom Read)和不可重复读(Nonrepeatable Read)。解释为：不可重复读指事务B Update数据造成的问题，幻读指事务B Insert或Delete数据造成的问题，这两者的定义也均与***维基百科***中的一致。这里的“幻读”其实和官方文档中的幻行(Phantom Row)问题是一样的，在官方文档中有这句话：***If a [SELECT] is executed twice, but returns a row the second time that was not returned the first time, the row is a “phantom” row***。在解决了幻像问题的Repeatable Read级别下，使用Next-Key Lock：Gap Lock + Record Lock来锁定读取到的数据行及其检索范围。    
&emsp;&emsp;**这里注意，进行“一致性非锁定读”(快照读)时均使用“多版本并发控制(MVCC)”来避免幻像问题，在“一致性锁定读”(当前读)的情况下才会使用锁算法来避免此问题。**                
### 存储引擎层面进行的锁的优化    
#### 锁降级    
&emsp;&emsp;当查询的索引具有唯一属性时，InnoDB会对Next-Key Lock进行优化，将其降级为Record Lock，即锁住索引本身，而不是范围。这里举个例子，首先创建表:     
```sql
    CREATE TABLE t (a INT PRIMARY KEY);
    INSERT INTO t SELECT 1;
    INSERT INTO t SELECT 2;
    INSERT INTO t SELECT 5;
```     

&emsp;&emsp;接下来执行表中的语句:      
|时间|会话A|会话B|
|---|------|-----|
1|```BEGIN;```||
2|```SELECT * FROM t WHERE a=5 FOR UPDATE;```||
3||```BEGIN;```|
4||```INSERT INTO t SELECT 4;```|
5||```COMMIT;```|
6|```COMMIT;```||

&emsp;&emsp;由于a是具有唯一属性的索引，在上面的例子中，会话B的```COMMIT```会直接成功，不需要阻塞。这种锁降级会大大提高数据库的并发性。如果查询的索引不具有唯一属性，则会对该索引加Next-Key Lock，但是对应的聚簇索引依旧为Record Lock。为了说明这一点，我们再创建一个表:     
```sql
    CREATE TABLE z (a INT,b INT, PRIMARY KEY(a),KEY(b));
    INSERT INTO z SELECT 1,1;
    INSERT INTO z SELECT 3,1;
    INSERT INTO z SELECT 5,3;
    INSERT INTO z SELECT 7,6;
```     

&emsp;&emsp;接着执行下面的SQL语句:      
```sql
    SELECT * FROM z WHERE b=3 FOR UPDATE;
```     

&emsp;&emsp;此时则会对索引b加一个Next-key Lock，其范围是(1,6),对索引a加一个Record Lock，加在了a=5的记录上。因此，下面的SQL语句都会被阻塞：     
```sql
    SELECT * FROM z WHERE a=5 LOCK IN SHARE MODE;
    INSERT INTO z SELECT 4,2;
    INSERT INTO z SELECT 6,5;
```

#### 锁升级     
&emsp;&emsp;维护锁信息是一个占用内存的操作，为了避免锁信息占用过多内存，在Microsoft SQL Server中有锁升级的策略，即在必要的时候将行锁升级为页锁，或将页锁升级为表锁。然而，***InnoDB存储引擎不存在锁升级的情况***，这得益于InnoDB采用位图的方式记录每个数据页的锁信息，占用内存极少。     
> ## 意向锁有什么用   
&emsp;&emsp;直接看例子：事务A锁住了表中的一行，让这一行只能读，不能写。之后，事务B申请整个表的写锁。如果事务B申请成功，那么理论上它就能修改表中的任意一行，这与A持有的行锁是冲突的。事务B申请表锁的时候，需要确认表中的所有行都没有被上锁，这时候如果没有意向锁，则需要遍历每一行，很费时间。如果事务A在加行锁前对表加上意向锁，事务B只需判断是否有意向锁来决定是否阻塞，效率明显提高。         
&emsp;&emsp;总结一下就是：意向锁是一种快速判断表锁是否与之前可能存在的行锁冲突的机制。         

> ## 解决死锁问题   
&emsp;&emsp;死锁是指两个以上的事务在执行过程中，因争夺锁而造成的一种互相等待的现象。     
&emsp;&emsp;解决死锁问题最简单的方法是超时机制，即两个事务互相等待时，当一个事务等待的时间超过了设置的阈值，便对其进行回滚，使另一个事务得以进行。参数innodb_lock_wait_timeout可以用来设置超时时间。这里就涉及到另外一个参数：innodb_rollback_on_timeout，该参数的决定了当前请求锁超时之后回滚的是整个事物还是仅当前语句，默认值是off，即回滚当前语句。         
&emsp;&emsp;还有另一种方式便是等待图(wait-for graph)。在等待图中，事务为图中的节点，节点T1指向节点T2定义为：事务T1在等待事务T2占用的资源；或者是：事务T1与事务T2等待相同的资源，而事务T1发生在事务T2后面。可以想到，若图中存在回路，即可认为存在死锁，此时InnoDB会在回路中选择回滚量最小的事务进行回滚。innodb_deadlock_detect参数可以开启或关闭这种检测方法。     
