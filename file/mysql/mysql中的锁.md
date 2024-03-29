[toc]



## MySQL有哪些锁🔒

根据加锁范围，MySQL中包含了全局锁、表级锁和行级锁三类。

### 全局锁

通过下面这条语句来使用全局锁

```sql
flush table with read lock
```

执行之后，**整个数据库就处于只读状态**了，这时其他线程执行增删改或者更改表结构等操作都会被阻塞

```sql
unlock tables  # 释放全局锁
```

如果会话断开，全局锁会自动释放

全局锁的主要应用场景在**全库逻辑备份**，使用全局锁在备份数据库期间，不会因为数据或表结构的更新，而出现备份文件的数据和预期不一致的情况。比如一个bad case：

一般用户的购买行为涉及到多张表的更新，比如用户表中更新用户的余额，商品表中减少某件商品的库存，那么在全库逻辑备份期间，如果没有使用全局锁，可能会出现这样的情况：

1. 先备份了用户表数据
2. 然后用户发生了购买行为
3. 在备份商品表

这种情况下，备份的结果是用户的余额并没有减少，而商品却减少了一件，这显然是不能接受的。



但是通过上面的描述又能够看出**全局锁的缺点：**加上全局锁后，这个数据库都处于只读状态，因此在备份期间只能发生读操作，而不能更新数据，这样造成了业务停滞。

那么**如何在不使用全局锁的情况下来完成全库逻辑备份的任务：**如果数据库的引擎支持的事务支持**可重复读的隔离级别**，那么在备份数据库之前先开启事务，会先创建Read View，然后在整个事务的执行期间都是用这一个Read View，本质上使用的是MVCC的快照读，由于MVCC的支持，备份期间业务仍然可以对数据进行更新操作，但是对于不支持事务的引擎比如MyISAM，就只能使用全局锁了

> 备份数据库的工具是mysqldump，在使用mysqldump时加上`-single-transaction`参数的时候，就会在备份数据库之前先开启事务，这种方式只适用于支持可重复读隔离级别的事务的存储引擎



### 表级锁

MySQL中的表级锁有以下几种，分别是：表锁、元数据锁（MDL）、意向锁和AUTO-INC锁

#### 表锁

- 表级别的共享锁，就是读锁
- 表级别的独占锁，就是写锁

```sql
//读锁；
lock tables t_student read;
//写锁；
lock tables t_stuent write;
```

**表锁除了会限制其他线程的读写操作之后，也会限制本线程接下来的读写操作**

要释放表锁，可以使用下面这条命令，会释放当前会话的所有表锁

```sql
unlock tables;
```

值得注意的是，**在使用InnoDB引擎的时候尽量避免使用表锁，因为表锁的力度太大会影响到数据库的并发性能，而InnoDB引擎厉害的一个点就在于实现了更细粒度的行级锁**



#### 元数据锁（MDL）

全称应该是Meta Data Lock

我们不需要显示的使用MDL，因为当我们对数据库表进行操作时，会自动给这个表加上MDL

- 当我们进行CRUD操作时，加的是**MDL读锁**
- 当我们对表结构进行更改时，加的是**MDL写锁**

**元数据锁的作用就是为了避免CRUD操作和表结构更改操作两者之间相互影响，**具体如下：

- 当我们进行CRUD操作时，会自动加上MDL读锁，此时，如果有线程想要修改表结构，那么这个线程会被阻塞住
- 同理，当我们在更新表结构的时候，会自动加上MDL写锁，此时，如果有线程想要进行CRUD操作，那么这个线程也会被阻塞住



**MDL不需要显式调用，那么他什么时候被释放掉呢？MDL在事务提交之后才会被释放，这意味着，在事务执行期间，MDL是一直持有的**

对于MDL需要注意长事务的问题，所谓长事务，指的就是开启了事务，但是一直还没提交。

存在下面这种case

- 事务A启用了事务，但一直没提交，然后执行了select方法，这时会对表加上MDL读锁
- 然后线程B也执行了同样的select语句，此时并不会阻塞，因为读读不冲突
- 接着，线程C想要修改表结构，那么由于MDL读锁一直没有被释放，这个线程无法申请到MDL写锁，因此会被阻塞住

这时候，问题就来了，**线程C阻塞之后，后续所有对表的select操作都会被阻塞住**，此时如果有大量针对该表的select操作到来，就会有大量的线程被阻塞住，数据库的线程池很快就会被占满

> 为什么线程C阻塞之后，后续所有对表的select操作都会被阻塞住呢？
>
> 答案是所有申请MDL锁的线程会形成一个队列，队列中写锁的优先级要高于读锁，一旦出现MDL写锁等待，就会阻塞后续所有的CRUD操作

所以，为了能够安全的变更表结构，在正式开始变更之前，先要看看数据库中的长事务，是否有事务已经对表加上了MDL读锁，如果有的话，可以考虑kill掉这个长事务之后再进行表结构变更



#### 意向锁

接着，说一下意向锁。

- 在使用InnoDB引擎的表里对某些记录加上共享锁之前，需要先在表级别加上一个意向共享锁
- 在使用InnoDB引擎的表里对某些记录加上独占锁之前，需要先在表级别加上一个意向独占锁

也就是，**当执行插入、更新、删除操作，需要先对表加上「意向独占锁」，然后对该记录加独占锁。**

而普通的select是不会加行级锁的，普通select的实现使用过MVCC实现的一致性读，是无锁的

不过select也是可以对记录加共享锁和独占锁的，如下：

```sql
// 先在表上加上意向共享锁，然后对读取的记录加共享锁
select ... lock in share mode;
// 先在表上加上意向独占锁，然后对更新得记录加独占锁
select ... for update;
```

**意向共享锁和意向独占锁都是表锁级，并不会和行级别的共享锁和独占锁发生冲突，而且意向锁之间也不会发生冲突，只会和共享表锁（lock tables ... read）和独占表锁（lock tables ... write）发生冲突**

表锁和行锁是满足读读共享、读写共享、写写互斥的。

*如果没有意向锁的话，那么加独占表锁时，就需要遍历表里所有的记录，查看是否有记录存在独占锁，这样效率会很慢。*

那么有了意向锁，由于在对记录加独占锁前，会先加上表级别的意向独占锁，那么在加独占表锁时，直接查找该表是否有意向独占锁，如果有，那么就意味着表里已经有记录加上了独占锁，这样就不用去遍历所有的记录了。

**因此，意向锁的目的是为了快速判断表里是否有记录被加锁**，应用场景就像上面说的那样，加独占表锁之前，先看看数据库表中有没有记录加上了独占锁



#### AUTO-INC锁

表里的主键通常会设置成自增的，这是通过对主键字段声明`AUTO_INCREMENT`属性实现的

之后可以在插入数据时，可以不指定主键的值，数据库会自动给主键赋值递增的值，这主要是通过`AUTO-INC`锁实现的。

`AUTO-INC`锁是特殊的表锁机制，锁不是在一个事务提交之后释放，而是在执行完插入语句之后就会立刻释放

**<font color=red>pre：</font>在插入数据时，会加一个表级别的的`AUTO-INC`锁，然后为被`AUTO_INCREMENT`修饰的字段赋值递增的值，等插入语句执行完毕之后，才会把`AUTO-INC`锁释放掉**

那么，一个事务在持有AUTO-INC锁的过程，其他事务如果要想该表插入语句都会被阻塞，从而保证插入数据时，被`AUTO-INCREMENT`修饰的字段的值都是连续递增的。

但是，`AUTO-INC`锁在面对大量数据插入的场景时，会严重影响插入性能，因为另一个事务中的插入会被阻塞

**<font color=red>cur：</font>**因此，在MySQL 5.1.22版本开始，InnoDB存储引擎提供了一种**轻量级的锁**来实现自增

> `AUTO-INC`  ------> 更轻量级的锁

同样地，**在插入数据的时候，MySQL会为被`AUTO_INCREMENT`修饰的字段加上轻量级锁，然后给该字段赋值（自增 ），做完这个操作之后就会把锁给释放掉，而不需要再等插入语句执行完毕。**

<font color=red>这样提高了数据库的性能，但是可能会造成主键字段不连续的问题，在主从复制的场景下从而可能出现主从数据不一致的情况</font>，

> 为什么主键字段会不连续在其他文件中有描述了，这里不再多赘述

主从数据不一致的case，比如下面这个例子：

<img src="../../image/mysql/轻量级的AUTO-INC锁bad case.webp" style="zoom:50%;" />

sessionA往表t中插入了四行数据，然后sessionB创建了一个相同表结构的表t2，然后两个session同时向表t2中插入数据

如果使用我们前面说的轻量级锁【主键自增之后锁就会被释放掉，不需要等待插入语句执行完毕】，那么可能出现这样的场景：

- sessionB先插入了两条记录，（1，1，1）和（2，2，2）
- 然后，sessionA 来申请自增 id 得到 id=3，插入了（3,5,5)
- 之后，sessionB 继续执行，插入两条记录 (4,3,3)、 (5,4,4)

可以看到，**sessionB的insert语句，生成的id是不连续的**

当主库发生了上面的变化是，binlog记录的内容是这两个session的insert语句，如果binlog_format=statement，记录的语句就是原始语句，记录的顺序要么先是sessionA的insert语句，要么就先是sessionB的insert语句

但无论是哪一种，这个binlog会被拿去到从库上执行，从库会顺序执行这两条insert语句，所以这种情况下**sessionB的insert语句生成的id是连续的**，这就造成了**<font color=red>主库和从库上的数据不一致</font>**问题

为了解决这个问题，**binlog的日志记录格式可以设置成row，**这样在 binlog 里面记录的是主库分配的自增值，到备库执行的时候，主库的自增值是什么，从库的自增值就是什么。



事实上，InnoDB存储引擎提供了一个系统变量`innodb_autoinc_lock_mode`用来控制选择使用的是`AUTO-INC`锁，还是轻量级的锁

- 当 `innodb_autoinc_lock_mode = 0`，就采用 `AUTO-INC` 锁，语句执行结束后才释放锁；
- 当` innodb_autoinc_lock_mode = 2`，就采用轻量级锁，申请自增主键后就释放锁，并不需要等语句执行后才释放。
- 当` innodb_autoinc_lock_mode = 1`：
  - 普通 insert 语句，自增锁在申请之后就马上释放；
  - 类似 insert … select 这样的批量插入数据的语句，自增锁还是要等语句结束后才被释放；

所以，**在不考虑主键严格连续递增的情况下，可以将`innodb_autoinc_lock_mode`设置成2，binlog_format设置成row，这样既能够提高数据库的并发能力，也能够保证数据的一致性，只是binlog会变得比较大了**



### 行级锁

**写在前面：**InnoDB支持行级锁，MyISAM不支持行级锁

前面也提到，普通的select语句是不会对记录加锁的（通过MVCC实现），因为它属于**快照读**，如果要在查询时对记录加行锁，可以使用下面这两种方式，这种查询会加锁的语句成为**锁定读**。

```sql
// 对读取的数据加共享锁
select ... lock in share mode;
// 对读取的数据加独占锁
select ... for update;
```

上面这两条语句必须在事务中，因为当事务提交了，锁就会被释放，所以在使用这两条语句的时候，要加上begin、start transaction或者set autocommit=0

**共享锁（S锁）满足读读共享、读写互斥，独占锁（X锁）满足写写互斥、读写互斥**

行级锁的类型主要有三类，分别是`Record Lock`，`Gap Lock`，`Next-Key Lock`



- `Next-Key Lock`：`Record Lock + Gap Lock`的组合，锁定一个范围，并且锁定记录本身

#### Record Lock

记录锁，锁住的是一条记录，**记录锁又分成了S锁和X锁，这里的S锁和X锁同样满足我们上面说到的读写关系**



#### Gap Lock

间隙锁，锁定一个范围，但是不包含记录本身

这个锁只存在于可重复读（Repeat Read）隔离级别下，目的是为了解决该隔离级别下存在的幻读问题

**间隙锁中也存在S锁和X锁，但是并没有什么区别，间隙锁之间是兼容的，即<font color=red>两个事务可以同时持有包含共同间隙范围的间隙锁，并不存在互斥关系，因为间隙锁的目的是防止插入幻影记录</font>**



#### Next-Key Lock

临键锁，Record Lock和Gap Lock的组合（前开后闭），锁定的是一个范围，和记录本身

所以，Next-Key Lock既能够阻止其他数据向这个间隙内的插入/数据，又能够保护这条记录不被修改

**虽然相同范围的间隙锁是兼容的，但是Next-Key Lock中还包含了记录锁，因此如果一个事务获取了 X 型的 next-key lock，那么另外一个事务在获取相同范围的 X 型的 next-key lock 时，是会被阻塞的**。



#### 插入意向锁

除了上面三个常见的行级锁外，还有一个行级锁，插入意向锁，注意和前面的表级锁—意向（共享/独占）锁的区分

我们知道，一个事务再插入一条记录的时候，需要判断插入位置是否已经被其他事务加了间隙锁，如果有的话，插入操作就会发生阻塞，直到拥有间隙锁的那个事务提交为止（释放间隙锁），在此期间会生成一个**插入意向锁，表明有事务想在某个区间插入新记录，但是现在处于<font color=red>等待状态</font>。**因此，虽然插入意向锁的名字中带有意向，但他并不是意向锁，而是一种特殊的间隙锁，属于行锁级别，因为间隙锁锁住的是一个区间，插入意向锁锁住的其实是一个点

> MySQL加锁时，是先生成锁结构，然后设置锁的状态，如果锁状态是等待状态，并不意味着事务成功获取到了锁，只有当锁状态为正常状态时，才代表事务成功获取到了锁

**注意：**尽管插入意向锁也属于间隙锁，但两个事务却不能在同一时间内，一个拥有间隙锁，一个**拥有**该间隙区间的插入意向锁

> 需要注意区分拥有和等待，出于等待状态说明还没成功获取到锁，当成功获取到插入意向锁之后，就要执行插入操作了。





## MySQL是怎么加锁的

这里我们说的加锁，指的是加行级锁的场景，因为全局锁和表锁的场景都比较简单。而行级锁加锁的加锁规则比较复杂，不同的场景，加锁形式是不同的。

> 在说 MySQL 是怎么加行级锁的时候，其实是在说 InnoDB 引擎是怎么加行级锁的。因为MyISAM不支持行级锁

**InnoDB中加锁的对象是索引，加锁的基本单位是Next-Key Lock，但是在使用记录锁或间隙锁就能够避免幻读的场景下，Next-Key Lock就会退化成记录锁或者间隙锁**

### 什么sql语句会让InnoDB加上行级锁

- `UPDATE`和`DELETE`语句

- 锁定读

  ```sql
  select ... lock in share mode;
  select ... for update;
  ```





下面都在这个表的基础上举例子

<img src="../../image/mysql/示例表.webp" style="zoom:67%;" />

**可以通过`select * from performance_schema.data_locks\G;`这条语句来查看事务执行过程中加了哪些锁**

一些重要的字段

- `LOCK_TYPE`：`TABLE`表示的是表锁，`RECORD`表示的是行锁
- `LOCK_MODE`：`X`表示的是`next-key lock`锁，`X, REC_NOT_GAP`说明是记录锁，`X, GAP`说明是间隙锁
- `LOCK_STATUS`：`GRANTED`正在占有这个锁



### 唯一索引 等值查询

当我们使用唯一索引进行等值查询的时候，查询的记录存在与否，加锁的规则也大不相同

- **当查询的记录是存在的**，在索引树上定位到这一条记录之后，将该记录的索引中的next-key lock会退化成**记录锁**

  > 如果这个唯一索引不是主键索引，是二级索引的话，那么在二级索引的索引项上会加上一个记录锁，对应的主键索引项上也会加上记录锁

- **当查询的记录不存在时，**会在索引树上找到第一条大于该查询记录的记录后，该记录的索引中的next-key lock退化成**间隙锁**

其实，这个很好理解，首先需要明确我们在可重复读级别使用Next-Key Lock的目的是什么？目的就是解决幻读问题，幻读其实就是两次读取得到的记录数不一致。

针对唯一索引等值查询，如果存在记录，那么也就这一条记录，我只需要保证这一条记录不被删除就好了，所以临键锁会退化成记录锁

如果不存在记录，那么我的目的就是在事务结束之前，这条不存在的记录不能被插入进来，所以临键锁退化成一个间隙锁，锁的范围就是找到第一条比等值条件大的记录，范围是这条记录和他前面的那条记录，如下：

```sql
select * from user where id = 2 for update;
```

此时间隙锁的范围是`(1,5)`



### 唯一索引 范围查询

当唯一索引执行范围查询时，会对每一个扫描到的索引加next-key 锁，然后如果遇到下面这些情况，会退化成记录锁或者间隙锁

- 针对大于等于的范围查询，因为存在等值查询的条件，那么如果等值查询的记录是存在于表中，那么该记录的索引中的next-key锁会退化成**记录锁**
- 针对「小于或者小于等于」的范围查询，要看条件值的记录是否存在于表中：
  - 当条件值的记录不在表中，那么不管是「小于」还是「小于等于」条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁。
  - 当条件值的记录在表中，如果是「小于」条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁；如果「小于等于」条件的范围查询，扫描到终止范围查询的记录时，该记录的索引 next-key 锁不会退化成间隙锁。其他扫描到的记录，都是在这些记录的索引上加 next-key 锁

> 针对大于的范围查询

假设事务A执行了下面的语句

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id > 15 for update;
+----+-----------+-----+
| id | name      | age |
+----+-----------+-----+
| 20 | 香克斯    |  39 |
+----+-----------+-----+
1 row in set (0.01 sec)
```

事务A的加锁变化过程如下：

1. 最开始要找的第一行是`id=20`，由于查询该记录不是一个等值查询（不是大于等于条件查询），所以对该主键索引加的是范围为（15, 20]的next-key锁
2. 由于是范围查询，会继续往后找存在的记录，虽然我们看见表中最后一条记录是id=20的记录，但是实际上在InnoDB存储引擎中，会用一个特殊的记录（`supremum pseudo-record`）来标识最后一条记录，所以扫描第二行的时侯，也就是扫描到了这个特殊记录的时候，会对该主键索引加一个范围是`(20,+∞]`的next-key锁

综上，事务A在主键索引上加了两个X型的next-key lock锁

<img src="../../image/MySQL/唯一索引范围查询大于15.drawio.webp" style="zoom:80%;" />

> 针对大于等于的查询情况

假设事务A执行了以下的范围查询语句

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id >= 15 for update;
+----+-----------+-----+
| id | name      | age |
+----+-----------+-----+
| 15 | 乌索普    |  20 |
| 20 | 香克斯    |  39 |
+----+-----------+-----+
2 rows in set (0.00 sec)
```

事务 A 加锁变化过程如下：

1. 最开始要找的第一行是 id = 15，由于查询该记录是一个等值查询（等于 15），所以该主键索引的 next-key 锁会**退化成记录锁**，也就是仅锁住 id = 15 这一行记录。
2. 由于是范围查找，就会继续往后找存在的记录，扫描到的第二行是 id = 20，于是对该主键索引加的是范围为 (15, 20] 的 next-key 锁；
3. 接着扫描到第三行的时候，扫描到了特殊记录（ supremum pseudo-record），于是对该主键索引加的是范围为 (20, +∞] 的 next-key 锁。
4. 停止扫描。

综上，事务A在主键索引上会加3个X型的锁

<img src="../../image/MySQL/唯一索引范围查询大于等于15.drawio.webp" style="zoom:80%;" />

> 针对小于的范围查询，查询条件值的记录不存在表中的情况

假设事务A执行了这条范围查询语句，注意查询条件值的记录（id为6）并不存在于表中

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id < 6 for update;
+----+--------+-----+
| id | name   | age |
+----+--------+-----+
|  1 | 路飞   |  19 |
|  5 | 索隆   |  21 |
+----+--------+-----+
3 rows in set (0.00 sec)
```

事务A加锁变化过程如下：

1. 最开始要找的第一行是id=1，于是对该主键索引加的是(-∞,1]的next-key锁
2. 由于是范围查找，就会继续往后找存在的记录，扫描到的第二行是id=5，所以对该主键索引加的是范围为(1,5]的next-key锁
3. 由于第二行扫描到的记录是id=5，满足id<6，且没有到终止条件，所以会继续扫描
4. 扫描到了第三行是id=10，该记录不满足id<6条件的记录，所以id=10这一行会退化成间隙锁，于是对该主键加一个(5,10)的间隙锁，由于扫描到的第三行记录（id = 10），不满足 id < 6 条件，达到了终止扫描的条件，于是停止扫描

![](../../image/MySQL/唯一索引范围查询小于等于6.drawio.webp)



**针对「小于或者小于等于」的唯一索引范围查询，如果条件值的记录不在表中，那么不管是「小于」还是「小于等于」的范围查询，扫描到终止范围查询的记录时，该记录中索引的 next-key 锁会退化成间隙锁，其他扫描的记录，则是在这些记录的索引上加 next-key 锁**。



> 针对小于的范围查询，查询条件值的记录不存在表中的

假设事务A执行了一下语句

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id <= 5 for update;
+----+--------+-----+
| id | name   | age |
+----+--------+-----+
|  1 | 路飞   |  19 |
|  5 | 索隆   |  21 |
+----+--------+-----+
2 rows in set (0.00 sec)
```

事务A的加锁变化过程如下：

1. 最开始要找的第一行是 id = 1，于是对该记录加的是范围为 (-∞, 1] 的 next-key 锁；
2. 由于是范围查找，就会继续往后找存在的记录，扫描到的第二行是 id = 5，于是对该记录加的是范围为 (1, 5] 的 next-key 锁。
3. 由于主键索引具有唯一性，不会存在两个 id = 5 的记录，所以不会再继续扫描，于是停止扫描

<img src="../../image/MySQL/唯一索引范围查询小于等于5.drawio.webp" style="zoom:80%;" />

> 针对小于的范围查询时，查询条件值的记录存在表中的情况

假设事务A执行了以下语句：

```sql
select * from user where id < 5 for update;
```

事务 A 加锁变化过程如下：

1. 最开始要找的第一行是 id = 1，于是对该记录加的是范围为 (-∞, 1] 的 next-key 锁；
2. 由于是范围查找，就会继续往后找存在的记录，扫描到的第二行是 id = 5，该记录是第一条不满足 id < 5 条件的记录，于是**该记录的锁会退化为间隙锁，锁范围是 (1,5)**。
3. 由于找到了第一条不满足 id < 5 条件的记录，于是停止扫描。

<img src="../../image/MySQL/唯一索引范围查询小于5.drawio.webp" style="zoom:80%;" />





### 非唯一索引 等值查询

当我们用非唯一索引进行等值查询的时候，**因为存在两个索引，一个是主键索引，一个是非唯一索引（二级索引），所以在加锁时，同时会对这两个索引都加锁，但是对主键索引加锁的时候，只有满足查询条件的记录才会对它们的主键索引加锁**。

针对非唯一索引等值查询时，查询的记录存不存在，加锁的规则也会不同：

- 当查询的记录「存在」时，由于不是唯一索引，所以肯定存在索引值相同的记录，于是**非唯一索引等值查询的过程是一个扫描的过程，直到扫描到第一个不符合条件的二级索引记录就停止扫描，然后在扫描的过程中，对扫描到的二级索引记录加的是 next-key 锁，而对于第一个不符合条件的二级索引记录，该二级索引的 next-key 锁会退化成间隙锁。同时，在符合查询条件的记录的主键索引上加记录锁**。
- 当查询的记录「不存在」时，**扫描到第一条不符合条件的二级索引记录，该二级索引的 next-key 锁会退化成间隙锁。因为不存在满足查询条件的记录，所以不会对主键索引加锁**。

> 针对非唯一索引等值查询时，查询的值不存在的情况

假设事务A对非唯一索引（age）进行了等值查询，且表中不存在id=25这条记录

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where age = 25 for update;
Empty set (0.00 sec)
```

事务A加锁变化过程如下：

- 定位到第一条不符合查询条件的二级索引记录，即扫描到age=39，于是**二级索引的next-key锁就会退化成间隙锁**，范围是（22, 39）

<img src="../../image/MySQL/非唯一索引等值查询age=25.drawio.webp" style="zoom:80%;" />



注意：是**在二级索引上加的锁**。

*当有一个事务持有二级索引的间隙锁 (22, 39) 时，什么情况下，可以让其他事务的插入 age = 22 或者 age = 39 记录的语句成功？又是什么情况下，插入 age = 22 或者 age = 39 记录时的语句会被阻塞*？

我们首先要明确一个前提：**插入语句在插入一条记录之前，需要先定位到该记录在 B+树 的位置，如果插入的位置的下一条记录的索引上有锁，才会发生阻塞**。

而且在分析二级索引的间隙锁是否可以成功插入记录时，我们要先知道的是二级索引树是如何存放记录的？

**二级索引树是先按照二级索引（age列）按顺序排放的，在相同的二级索引值情况下，再按主键id的顺序存放。**

插入age=22成功和失败的情况分别如下：

- 当其他事务插入一条 age = 22，id = 3 的记录的时候，在二级索引树上定位到插入的位置，而**该位置的下一条是 id = 10、age = 22 的记录，该记录的二级索引上没有间隙锁，所以这条插入语句可以执行成功**。
- 当其他事务插入一条 age = 22，id = 12 的记录的时候，在二级索引树上定位到插入的位置，而**该位置的下一条是 id = 20、age = 39 的记录，正好该记录的二级索引上有间隙锁，所以这条插入语句会被阻塞，无法插入成功**。

插入age=39成功和失败的情况分别如下：

- 当其他事务插入一条 age = 39，id = 3 的记录的时候，在二级索引树上定位到插入的位置，而**该位置的下一条是 id = 20、age = 39 的记录，正好该记录的二级索引上有间隙锁，所以这条插入语句会被阻塞，无法插入成功**。
- 当其他事务插入一条 age = 39，id = 21 的记录的时候，在二级索引树上定位到插入的位置，而**该位置的下一条记录不存在，也就没有间隙锁了，所以这条插入语句可以插入成功**。



所以，**当有一个事务持有二级索引的间隙锁 (22, 39) 时，插入 age = 22 或者 age = 39 记录的语句是否可以执行成功，关键还要考虑插入记录的主键值，因为「二级索引值（age列）+主键值（id列）」才可以确定插入的位置，确定了插入位置后，就要看插入的位置的下一条记录是否有间隙锁，如果有间隙锁，就会发生阻塞，如果没有间隙锁，则可以插入成功**。



> 针对非唯一索引等值查询时，查询的值存在的情况

假设事务A对非唯一索引进行了等值查询，且表中存在age=22的记录

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where age = 22 for update;
+----+--------+-----+
| id | name   | age |
+----+--------+-----+
| 10 | 山治   |  22 |
+----+--------+-----+
1 row in set (0.00 sec)
```

- 由于不是唯一索引，所以肯定存在值相同的记录，于是非唯一索引等值查询的过程是一个扫描的过程，最开始要找的第一行是 age = 22，于是**对该二级索引记录加上范围为 (21, 22] 的 next-key 锁**。同时，因为 age = 22 符合查询条件，**于是对 age = 22 的记录的主键索引加上记录锁**，即对 id = 10 这一行加记录锁。
- 接着继续扫描，扫描到的第二行是 age = 39，该记录是第一个不符合条件的二级索引记录，所以该二级索引的 next-key 锁会**退化成间隙锁**，范围是 (22, 39)。
- 停止查询。

<img src="../../image/MySQL/非唯一索引等值查询存在.drawio.webp" style="zoom:0%;" />





### 非唯一索引 范围查询

非唯一索引和主键索引的范围查询的加锁也有所不同，不同之处在于**非唯一索引范围查询，索引的 next-key lock 不会有退化为间隙锁和记录锁的情况**，也就是非唯一索引进行范围查询时，对二级索引记录加锁都是加 next-key 锁。

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where age >= 22  for update;
+----+-----------+-----+
| id | name      | age |
+----+-----------+-----+
| 10 | 山治      |  22 |
| 20 | 香克斯    |  39 |
+----+-----------+-----+
2 rows in set (0.01 sec)
```

事务 A 的加锁变化：

- 最开始要找的第一行是 age = 22，虽然范围查询语句包含等值查询，但是这里不是唯一索引范围查询，所以是不会发生退化锁的现象，因此对该二级索引记录加 next-key 锁，范围是 (21, 22]。同时，对 age = 22 这条记录的主键索引加记录锁，即对 id = 10 这一行记录的主键索引加记录锁。
- 由于是范围查询，接着继续扫描已经存在的二级索引记录。扫面的第二行是 age = 39 的二级索引记录，于是对该二级索引记录加 next-key 锁，范围是 (22, 39]，同时，对 age = 39 这条记录的主键索引加记录锁，即对 id = 20 这一行记录的主键索引加记录锁。
- 虽然我们看见表中最后一条二级索引记录是 age = 39 的记录，但是实际在 Innodb 存储引擎中，会用一个特殊的记录来标识最后一条记录，该特殊的记录的名字叫 supremum pseudo-record ，所以扫描第二行的时候，也就扫描到了这个特殊记录的时候，会对该二级索引记录加的是范围为 (39, +∞] 的 next-key 锁。
- 停止查询

<img src="../../image/MySQL/非唯一索引范围查询age大于等于22.drawio.webp" style="zoom:80%;" />



> 在 age >= 22 的范围查询中，明明查询 age = 22 的记录存在并且属于等值查询，为什么不会像唯一索引那样，将 age = 22 记录的二级索引上的 next-key 锁退化为记录锁？

**因为 age 字段是非唯一索引，不具有唯一性**，所以如果只加记录锁（记录锁无法防止插入，只能防止删除或者修改），就会导致其他事务插入一条 age = 22 的记录，这样前后两次查询的结果集就不相同了，出现了幻读现象





### 没有加索引的查询

前面的案例，我们的查询语句都有使用索引查询，也就是查询记录的时候，是通过索引扫描的方式查询的，然后对扫描出来的记录进行加锁。

**如果锁定读查询语句，没有使用索引列作为查询条件，或者查询语句没有走索引查询，导致扫描是全表扫描。那么，每一条记录的索引上都会加 next-key 锁，这样就相当于锁住的全表，这时如果其他事务对该表进行增、删、改操作的时候，都会被阻塞**。

**不只是锁定读查询语句不加索引才会导致这种情况，update 和 delete 语句如果查询条件不加索引，那么由于扫描的方式是全表扫描，于是就会对每一条记录的索引上都会加 next-key 锁，这样就相当于锁住的全表。**

因此，**<font color=red>在线上在执行 update、delete、select ... for update 等具有加锁性质的语句，一定要检查语句是否走了索引，如果是全表扫描的话，会对每一个索引加 next-key 锁，相当于把整个表锁住了</font>**，这是挺严重的问题

*那update语句的where带上索引就能避免全表记录加锁了吗？*

并不是，关键还是得看这条语句在执行过程中优化器最终选择的是索引扫描还是全表扫描，如果走了全表扫描，就会对全表的记录加锁了，相当于把整个表都锁住，但并不是加的表锁。因为InnoDB在扫描记录的时候都是针对索引项这个单位加锁的，update 不带索引就是全表扫扫描，也就是表里的索引项都加锁，相当于锁了整张表。

*如何避免由于update等语句造成的上述问题呢？*

将` sql_safe_updates` 设置为 1 ，开启安全更新模式。

update 语句必须满足如下条件之一才能执行成功：

- 使用 where，并且 where 条件中必须有索引列；
- 使用 limit；
- 同时使用 where 和 limit，此时 where 条件中可以没有索引列；

delete 语句必须满足以下条件能执行成功：

- 同时使用 where 和 limit，此时 where 条件中可以没有索引列；

如果 where 条件带上了索引列，但是优化器最终扫描选择的是全表，而不是索引的话，我们可以使用 `force index([index_name])` 可以告诉优化器使用哪个索引，以此避免有几率锁全表带来的隐患



## MySQL死锁

给定一个场景，有数据表如下：

<img src="../../image/MySQL/t_student.webp" style="zoom:80%;" />

事务A和事务B分别在不同的时刻执行如下操作，会发生什么？

<img src="../../image/MySQL/ab事务死锁.drawio.webp" style="zoom:80%;" />

可以看到，事务 A 和 事务 B 都在执行 insert 语句后，都陷入了等待状态（前提没有打开死锁检测），也就是发生了**死锁**，因为都在相互等待对方释放锁

**那么为什么会出现死锁呢？**

- 在TIme1阶段，事务A执行以下语句：

  ```sql
  # 事务 A
  mysql> begin;
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> update t_student set score = 100 where id = 25;
  Query OK, 0 rows affected (0.01 sec)
  Rows matched: 0  Changed: 0  Warnings: 0
  ```

  因为表中没有id=25的字段，在可重复读隔离级别下，为了避免幻读，MySQL会在主键索引处加上一个间隙锁，锁的范围是（20，30）

- 在Time2阶段，事务B执行以下语句：

  ```sql
  # 事务 B
  mysql> begin;
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> update t_student set score = 100 where id = 26;
  Query OK, 0 rows affected (0.01 sec)
  Rows matched: 0  Changed: 0  Warnings: 0
  ```

  因为表中也没有id=26的字段，同理，MySQL会在主键索引处加上一个间隙锁，锁的范围是（20，30）


  这里需要注意的是，**不同事务的间隙锁之间是相互兼容的，**因为间隙锁的意义只在于阻止区间被插入，因此一个事务获取的间隙锁不会阻止另一个事务获取同一个间隙范围的间隙锁。

- 在Time3阶段，事务A插入了一条记录，这时候*事务A就会进入等待状态了*。这是因为事务A的插入操作会生成一个插入意向锁，虽然插入意向锁也属于间隙锁，但是两个事务并不能在同一时间内一个拥有间隙锁，另一个拥有该间隙区间内的插入意向锁（当然，插入意向锁如果不在间隙锁区间内是可以的），所以插入意向锁和间隙锁之间的冲突的

  > 插入意向锁其实可以看做是一种特殊的间隙锁，间隙锁是控制一个范围不被插入，插入意向锁是聚焦于一个点不被插入。
  > 每插入一条新记录，都需要看一下待插入记录的下一条记录上是否已经被加了间隙锁，如果已加间隙锁，此时会生成一个插入意向锁，然后锁的状态设置为等待状态（*PS：MySQL 加锁时，是先生成锁结构，然后设置锁的状态，如果锁状态是等待状态，并不是意味着事务成功获取到了锁，只有当锁状态为正常状态时，才代表事务成功获取到了锁*），现象就是 Insert 语句会被阻塞

- 在Time4阶段，事务B页插入了一条记录，这时候事务B也会进入等待状态了，同理，事务B的插入同样会生成一个插入意向锁。

**所以，事务A和事务B都在等待对方的间隙锁释放，因此就进入了死锁状态。**







































