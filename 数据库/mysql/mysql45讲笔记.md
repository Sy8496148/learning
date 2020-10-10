# 第一章 基础架构

![img](https://static001.geekbang.org/resource/image/0d/d9/0d2070e8f84c4801adbfa03bda1f98d9.png)

- 客户端 >> 连接器（管理连接，权限验证） >> 分析器（验证语法） >> 优化器(执行计划生成，索引选择) >> 执行器（操作引擎，返回结果） --->引擎

- 默认创建表是使用的**InnoDB**，同时可以是用MyISAM，Memory等其他引擎
- 默认连接是8个小时，如果是用长连接倒是内存增长过快，导致异常重启。可以定期断开长连接，或者在执行了一个比较大的操作的时候执行**mysql_reset_connection**来重新初始化连接资源。
- 在数据库的慢查询日志中看到一个**rows_examined**的字段，表示这个语句执行过程中扫描了 多少行。





# 第二章 系统日志

- mysql 有两个重要的日志模块**redo log**（记录在引擎中InnoDB）**bin log**（记录在server层） 都是用来记录事物提交的状态

- 这两种日志有以下三点不同。 
  1. redo log是InnoDB引擎特有的；binlog是MySQL的Server层实现的，所有引擎都可以使用。
  2. redo log是物理日志，记录的是“在某个数据页上做了什么修改”；binlog是逻辑日志，记录的 是这个语句的原始逻辑，比如“给ID=2这一行的c字段加1 ”。 
  3. redo log是循环写的，空间固定会用完；binlog是可以追加写入的。“追加写”是指binlog文件 写到一定大小后会切换到下一个，并不会覆盖以前的日志。

- redo log来实现crash-safe能力（可以保证即使数据库发生异常重启，之前提交的记录都不会丢失）

- 两阶段提交，保证redolog和binlog的逻辑上的一致

- 两阶段提交过程

  1，执行器先找到引擎取ID=2一行（假设要更新的数据ID=2），如过ID=2所在这一行的数据页是否在内存中，如果在就返回，如果不在就将数据页写入内存，然后再返回

  2，执行器拿到引擎给的行数据，把这个值加上1，比如原来是N，现在就是N+1，得到新的一行数据，再调用引擎接口写入这行新数据。

  3，引擎将这行要更新的数据更新到内存中，同时将这个更新操作记录到redo log里面，此时redo log处 

  于prepare状态。然后告知执行器执行完成了，随时可以提交事务。 

  4，执行器生成这个操作的binlog，并把binlog写入磁盘。 

  5，执行器调用引擎的提交事务接口，引擎把刚刚写入的redo log改成提交（commit）状态，更 

  新完成。 





# 第三章 事务隔离

- 事务就是要保证一组数据库操作，要么全部成功，要么全部失败。

- SQL标准的事务隔离级别包括：读未提交（read uncommitted）、 读提交（read committed ）、**可重复读（repeatable read 默认）**和串行化（serializable ）。

  读未提交：一个事务还没提交时，它做的变更就能被别的事务看到。 

  读提交：一个事务提交之后，它做的变更才会被其他事务看到。 

  可重复读：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。 

  串行化：顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

- 在MySQL中，实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。

- 长事务存在的风险

  1，回滚段的影响，会导致占用大量的储存空间

  2，会占用锁资源

- 事务的启动方式，**set autocommit = 1** 通过显示语句的方式来启动事务，用begin显式启动的事务，如果执行commit则提交事务。

- 如何避免长事务对业务的影响？ 

  1，确认是否使用了set autocommit=0。

  2，确认是否有不必要的只读事务。

  3，业务连接数据库的时候，根据业务本身的预估，通过SETMAX_EXECUTION_TIME命令， 

  来控制每个语句执行的最长时间，避免单个语句意外执行太长时间。





# 第四章 索引（上）

- 在MySQL中，索引是在存储引擎层实现的
- 在InooDB中使用的是B+树的模型，数据都是存储在B+树中，每一个索引对应一颗B+树。
- 索引类型分为主键索引和非主键索引，非主键索引有一个回表的过程，也就是说非主键索引的查询需要多扫描一颗索引树。因此我们需要尽可能的使用主键查询
- 为了索引的维护方便，性能的高效，应该使用自增主键的插入模式，每次都是插入一条新的纪录，都是追加的操作，都不会涉及到移动其他的记录，同时也不会触发叶子节点的分裂
- 由于每个非主键索引的叶子节点上都是主键的值，主键的长度越小，普通索引占用的空间就越小，因此从性能和空间方面考虑，自增主键都是一个好的选择
- 业务字段做主键的场景：KV





# 第五章 索引（下）

- 覆盖索引可以减少树搜索的次数，显著的提升性能，所以使用覆盖索引是一个常用的性能优化手段

- 覆盖索引上的信息足够满足查询的请求，不需要在会到主键索引上去取数据

- B+树这种索引结构，可以利用最左前缀原则来定位记录

- 当已经有了（A,B）这个联合索引后，一般就不需要在A上单独建立索引了，如果查询条件中只有B语句，是无法使用B索引的，这个时候就要考虑空间，选择空间大的在前位置

- MySQL 5.6 引入的索引下推优化（index condition pushdown)， 可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。

  ***

  现有索引（name，age）

  mysql > select * from tuser where name like '张%' and age=10 and ismale=1;

  所以这个语句在搜索索引树的时候，只能用 “张”，找到第一个满足条件的记录

  InnoDB在(name,age)索引内部就判断了age是否等于10，对于不等于10的记录，直接判断并跳过。在我们的这个例子中，只需要对ID4、ID5这两条记录回表取数据判断，就只需要回表2次。

  ***






# 第六章 全局锁和表锁

- 根据加锁的范围，mysql里面的锁可以分为**全局锁**，**表级锁**，**行锁**

- **全局锁**就是对整个数据库进行加锁，命令为flush tables with read lock，枷锁后整个库为只读状态。

- **全局锁**的使用场景为全库做逻辑备份，如果不加全局锁的话，备份系统得到的库不是一个逻辑时间点，导致数据不一致

- mysql 如何进行备份，在加锁的情况，同时保持表中的数据能够更新

  官方自带的逻辑备份工具是mysqldump，当mysqldump使用参数–single-transaction的时候，导 

  数据之前就会启动一个事务（可重复读：一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一 致的），来确保拿到一致性视图。而由于MVCC的支持，这个过程中数据是 可以正常更新的。 

- MySQL里面表级别的锁有两种：一种是**表锁**，一种是**元数据锁（meta data lock，MDL)**。

- **表锁**的语法是lock tables..read/write 与FTWRL类似，可以用unlock tables主动释放锁， 也可以在客户端断开的时候自动释放。需要注意，lock tables语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象。 

- **MDL**不需要显式使用，在访问一个表的时候会被 自动加上。MDL的作用是，保证读写的正确性。当对一个表做增删改查操作的时候，加MDL读锁；当要对表做结构变更操作的时候，加MDL写锁。

- 如何安全的给小表加字段：

  在alter table语句里面 设定等待时间，如果在这个指定的等待时间里面能够拿到MDL写锁最好，拿不到也不要阻塞后面的业务语句，先放弃。之后开发人员或者DBA再通过重试命令重复这个过程。






# 第七章 行锁

- 两阶段锁：在innodb事务中，行锁是要等到事务结束后才释放。

- 如果事务中需要锁住多个行，要把可能造成锁冲突，最有可能影响并发度的锁放到后面。

- 当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致 

  这几个线程都进入无限等待的状态，称为死锁。

- 解决死锁的两种策略

  1，直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout来设置。（不推荐使用）

  2，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数innodb_deadlock_detect设置为on。开启后每个新来的被堵住的线程，都要判断会不会由于自己的加入导致了死锁。（会很消耗CPU）

- 减少死锁的主要方向，就是控制访问相同资源的并发事务量。

  例如：可以考虑通过将一行改成逻辑上的多行来减少锁冲突。还是以影院账户为例，可以考虑放在多条记录上，比如10个记录，影院的账户总额等于这10个记录的值的总和。这样每次要给影院账户加金额的时候，随机选其中一条记录来加。这样每次冲突概率变成原来的1/10，可以减少锁等待个数，也就减少了死锁检测的CPU消耗。





# 第八章 事务隔离详解

- InnoDB在实现MVCC时用到的一致性读视图，用于支持RC（Read Committed，读提交）和RR（Repeatable Read，可重复读）隔离级别的实现。

- 在可重复读隔离级别下，事务在启动的时候就“拍了个快照”。注意，这个快照是基于整库的。 

- InnoDB里面每个事务有一个唯一的事务ID，叫作transaction id。而每行数据也都是有多个版本的。每次事务更新数据的时候，都会生成一个新的数据版本，并且把transaction id赋值给这个数据版本的事务ID，记为rowtrx_id。

-  InnoDB为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在“活跃”的所有事务ID。“活跃”指的就是，启动了但还没提交。数组里面事务ID的最小值记为低水位，当前系统里面已经创建过的事务ID的最大值加1记为高水位。这个视图数组和高水位，就组成了当前事务的一致性视图（read-view）。

- 一致性读的逻辑

  1，版本未提交，不可见（如果是当前的版本可见）； 

  2，版本已提交，但是是在视图创建后提交的，不可见； 

  3，版本已提交，而且是在视图创建前提交的，可见。

- 更新逻辑

  更新数据都是先读后写，这个读是“当前读”，总是读取已经提交完成的最新版本。

- update语句外，select语句如果加锁（悲观锁），也是当前读。 

  select k from t where id=1 for update; X锁

  select k from t where id=1 lock in share mode; S锁

- 读提交的逻辑和可重复读的逻辑类似，它们最主要的区别是

  在可重复读隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图；查询只承认在**事务**启动前就已经提交完成的数据；

  在读提交隔离级别下，每一个语句执行前都会重新算出一个新的视图。查询只承认在**语句**启动前就已经提交完成的数据； 





# 第九章 唯一索引和普通索引

- **查询的过程**

  普通索引：查找到满足条件的第一个记录(5,500)后，需要查找下一个记录，直到碰到第一个不满足k=5条件的记录。

  唯一索引：由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止继续检索。

  InnoDB的数据是按数据页为单位来读写的。也就是说，当需要读一条记录的时候，并不是将这个记录本身从磁盘读出来，而是以页为单位，将其整体读入内存。

- changebuffer ：当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB会将这些更新操作缓存在change buffer中，这样就不需要从磁盘中读入这个数据页了。在下次查询需要访问这个数据页的时候，将数据页读入内存，然后执行change buffer中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。 如果能够将更新操作先记录在change buffer，减少读磁盘，语句的执行速度会得到明显的提升。

- **更新的过程**

  1，更新的内容在内存中

  唯一索引：判断到没有冲突，插入这个值，语句执行结束；

  普通索引：插入这个值，语句执行结束。 

  2，更新的内容不在内存中

  唯一索引：需要将数据页读入内存，判断到没有冲突，插入这个值，语句执行结束；

  普通索引：将更新记录在change buffer，语句执行就结束了。

- 将数据从磁盘读入内存涉及随机IO的访问，是数据库里面成本最高的操作之一。change buffer因为减少了随机磁盘访问，所以对更新性能的提升是会很明显的。

- 对于**写多读少**的业务来说，页面在写完以后马上被访问到的概率比较小，此时change buffer的使用效果最好。这种业务模型常见的就是账单类、日志类的系统。 

- 一个业务的更新模式是写入之后马上会做查询，那么即使满足了条件，将更新先记录在change buffer，但之后由于马上要访问这个数据页，会立即触发merge过程。这样随机访问IO的次数不会减少，反而增加了change buffer的维护代价。所以，对于这种业务模式来说，change buffer反而起到了副作用。

- redo log 主要节省的是随机写磁盘的IO消耗（转成顺序写），而changebuffer 主要节省的则是随机读磁盘的IO消耗

- 事物提交的时候change buffer会把操作记录到redolog里面，在崩溃的时候也能够找回来

-  merge的执行流程：

  1，从磁盘读入数据页到内存（老版本的数据页）

  2，从change buffer里找出这个数据页的change buffer 记录(可能有多个），依次应用，得到新版数据页
  
  3， 写redo log。这个redo log包含了数据的变更和change buffer的变更。
  
  
# 第十章 Mysql为什么会选错索引

- 优化器选择索引的判断依据有，扫描的行数，是否使用临时表，是否排序等因素进行综合判断

- 扫描行数如何判断，是根据索引的区分度，一个索引上的不通值越多，这个索引的区分度就越好

- 索引选错异常处理：

  1，采用force index 来强行选择一个索引

  2，考虑修改语句，来引导mysql使用我们期望的索引

  3，新建一个更合适的索引，或者删掉误用的索引

  



# 第十一章 怎么给字符字段加索引

- 使用前缀索引，定义好长度，可以做到既节省空间，有不用额外增加太多的查询成本。

- 如何确定前缀索引的长度：

  建立索引时关注的是区分度，区分度越高越好重复的键值越少，因此我们可以通过统计索引上有多少个不同的值来判断要使用多长的前缀。

  ***

  select count(distinct email) as L from SUser; （email 列上有多少个不同的值）

  select count(distinct left(email,4)）as L4, （email 列上前四个值有多少个不同的值）

  select count(distinct left(email,5)）as L5, （email 列上前五个值有多少个不同的值）

  从中间选择出大的值

- 使用前缀索引就用不上覆盖索引对查询性能的优化了，这也是你在选择是否使用前缀索引时需要考虑的一个因素。 

- 字符串字段创建索引的场景：

  1，直接创建完整索引，这样可能比较占用空间； 

  2，创建前缀索引，节省空间，但会增加查询扫描次数，并且不能使用覆盖索引； 

  3，倒序存储，再创建前缀索引，用于绕过字符串本身前缀的区分度不够的问题； 

  3，创建hash字段索引，查询性能稳定，有额外的存储和计算消耗，跟第三种方式一样，都不支持范围扫描。






# 第十二章 为什么我的MySQL会“抖”一下？ 

- InnoDB在处理更新语句的时候，只做了写日志这一个磁盘操作。这个日志叫作redo log（重做日志），在更新内存写完redo log后，就返回给客户端，本次更新成功。

- 当内存数据页跟磁盘数据页内容不一致的时候，我们称这个数据页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据页的内容也就一致了，称为”干净页“。

- MySQL偶尔“抖”一下的那个瞬间，可能就是在刷脏页（flush）。

- 引发数据库flush的情况

  1，InnoDB的redo log写满了。这时候系统会停止所有更新操作，把checkpoint往前推进，redo log留出空间可以继续写。

  2，系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。

  3，MySQL认为系统“空闲”的时候

  4，MySQL正常关闭的情况

- innodb用缓冲池来管理内存，缓冲池中有三种状态

  1，还没有使用的

  2，使用了并且是干净页

  3，使用了并且是脏页

- innodb的脏页控制策略

  1，正确地设置innodb_io_capacity参数

  2，控制藏页比例 innodb_max_dirty_pages_pct是脏页比例上限，默认值是75%

  3，设置innodb_flush_neighbors为0，只会刷新跟自己相关的藏页不会刷邻居脏页

  



# 第十三章 删除表中的数据，空间大小为什么没变？

- 插入和删除数据不会造成表空间的回收，而是造成了空洞，下次有数据再插入进来的时候就会在复用这些空洞

- 重建表可以把表中的空洞去掉

- online DDL 重建表的流程

  1， 建立一个临时文件，扫描表A主键的所有数据页； 

  2，用数据页中表A的记录生成B+树，存储到临时文件中；

  3，生成临时文件的过程中，将所有对A的操作记录在一个日志文件（rowlog）中，对应的是图 

  中state2的状态；

  4，临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表A相同的 

  数据文件，对应的就是图中state3的状态；

  5，用临时文件替换表A的数据文件。

***
![img](https://static001.geekbang.org/resource/image/2d/f0/2d1cfbbeb013b851a56390d38b5321f0.png)
***





# 第十四章 count（*）很慢，该怎么办？

- count（\*）慢的原因是因为每一行记录都要判断自己是否对这个会话可见，因此对count（*）来说只能够把数据一行一行的都读出来一次判断
- InnoDB是索引组织表，主键索引树的叶子节点是数据，而普通索引树的叶子节点是主键值。所以，普通索引树比主键索引树小很多。
- 按照效率排序的话，count(字段)<count(主键id)<count(1)≈count(\*)，所以我建议你，尽量使用count(*)。





#  第十五章 答疑

业务设计问题 场景A、B两个用户，如果互相关注，则成为好友设计上是有两张表，一个是like表，一个是friend表，like表有user_id、liker_id两个字段，我设置为复合唯一索引即uk_user_id_liker_id。

以A关注B为例

第一步，先查询对方有没有关注自己（B有没有关注A）

select *fromlike where user_id =B and liker_id =A;

如果有，则成为好友

insert into friend; 

没有，则只是单向关注关系

insert into like; 

但是如果A、B同时关注对方，会出现不会成为好友的情况。因为上面第1步，双方都没关注对方。第1步即使使用了排他锁也不行，因为记录不存在，行锁无法生效。

解决方法：

首先，要给“like”表增加一个字段，比如叫作 relation_ship，并设为整型，取值1、2、3。

值是1的时候，表示user_id 关注 liker_id; 

值是2的时候，表示liker_id 关注 user_id; 

值是3的时候，表示互相关注

当 A关注B的时候

如果A<B

mysql> begin; /*启动事务*/

insert into `like`(user_id, liker_id, relation_ship) values(A, B, 1) on duplicate key update relation_ship=relation_ship | 1;

select relation_ship from `like` where user_id=A and liker_id=B; 

/*

代码中判断返回的 relation_ship，  

如果是1，事务结束，执行 commit  

如果是3，则执行下面这两个语句：  

*/

insert ignore into friend(friend_1_id, friend_2_id) values(A,B); 

commit;

如果A>B

mysql> begin; /*启动事务*/ 

insert into `like`(user_id, liker_id, relation_ship) values(B, A, 2) on duplicate key update relation_ship=relation_ship | 2;

select relation_ship from `like` where user_id=B and liker_id=A; 

/*

代码中判断返回的 relation_ship，  

如果是2，事务结束，执行 commit  

如果是3，则执行下面这两个语句： 

*/ 

insert ignore into friend(friend_1_id, friend_2_id) values(B,A); 

commit;

即使在双方“同时”执行关注操作，最终数据库里的结果，也是**like表里面有一条关于A和B的记录**，让“like”表里的数据保证user_id < liker_id，这样不论是A关注B，还是B关注A，在操作“like”表的时候，如果反向的关系已经存在，就会出现行锁冲突。

然后，insert … on duplicate语句，确保了在事务内部，执行了这个SQL语句后，就强行占住了这个行锁，之后的select 判断relation_ship这个逻辑时就确保了是在行锁保护下的读操作。

**取消关注逻辑：**

A单项关注B：

A>B

delete from like WHERE like_id=A and user_id=B;

A<B

delete from like WHERE like_id=B and user_id=A;

AB互相关注(也就是B单项关注A)：

A>B

insert into `like`(user_id, liker_id, relation_ship) values(B, A, 1) on duplicate key update relation_ship=relation_ship | 1;

delete from friend where friend_1_id=A and friend_2_id=B

B>A

insert into `like`(user_id, liker_id, relation_ship) values(A, B, 1) on duplicate key update relation_ship=relation_ship | 2;

delete from friend where friend_1_id=B and friend_2_id=A





# 第十六章 orderby是怎么样工作的

- select city,name,age from t where city='杭州' order by name limit 1000  ;

  1，初始化sort_buffer，确定放入name、city、age这三个字段；

  2，从索引city找到第一个满足city='杭州’条件的主键id，也就是图中的ID_X；

  3，到主键id索引取出整行，取name、city、age三个字段的值，存入sort_buffer中；

  4，从索引city取下一个记录的主键id；

  5，重复步骤3、4直到city的值不满足查询条件为止，对应的主键id也就是图中的ID_Y；

  6，对sort_buffer中的数据按照字段name做快速排序；

  7，按照排序结果取前1000行返回给客户端。

![img](https://static001.geekbang.org/resource/image/6c/72/6c821828cddf46670f9d56e126e3e772.jpg)

“按name排序”这个动作，可能在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和参数sort_buffer_size。number_of_tmp_files表示的是，排序过程中使用的临时文件数。你一定奇怪，为什么需要12个文件？内存放不下时，就需要使用外部排序，外部排序一般使用归并排序算法。

- **当单行数据很大时 会采用rowid排序**

  max_length_for_sort_data，是MySQL中专门控制用于排序的行数据的长度的一个参数。它的意思是，如果单行的长度超过这个值，MySQL就认为单行太大，要换一个算法。

  city、name、age 这三个字段的定义总长度是36，我把max_length_for_sort_data设置为16，我们再来看看计算过程有什么改变。

  select city,name,age from t where city='杭州' order by name limit 1000  ;

  1，初始化sort_buffer，确定放入两个字段，即name和id；

  2，从索引city找到第一个满足city='杭州’条件的主键id，也就是图中的ID_X；

  3，到主键id索引取出整行，取name、id这两个字段，存入sort_buffer中；

  4，从索引city取下一个记录的主键id；

  5，重复步骤3、4直到不满足city='杭州’条件为止，也就是图中的ID_Y；

  6，对sort_buffer中的数据按照字段name进行排序；

  7，遍历排序结果，取前1000行，并按照id的值回到原表中取出city、name和age三个字段返回给客户端。

  ![img](https://static001.geekbang.org/resource/image/dc/6d/dc92b67721171206a302eb679c83e86d.jpg)

- 如果name本来就有序，就可以不用排序了，创建一个city和name的联合索引

  1，从索引(city,name)找到第一个满足city='杭州’条件的主键id；

  2，到主键id索引取出整行，取name、city、age三个字段的值，作为结果集的一部分直接返回；

  3，从索引(city,name)取下一个记录主键id；

  4，重复步骤2、3，直到查到第1000条记录，或者是不满足city='杭州’条件时循环结束。

  ![img](https://static001.geekbang.org/resource/image/3f/92/3f590c3a14f9236f2d8e1e2cb9686692.jpg)

- 创建一个city、name和age的联合索引
  
  1，从索引(city,name,age)找到第一个满足city='杭州’条件的记录，取出其中的city、name和age这三个字段的值，作为结果集的一部分直接返回；
  
  2，从索引(city,name,age)取下一个记录，同样取出这三个字段的值，作为结果集的一部分直接返回；
  
  3，重复执行步骤2，直到查到第1000条记录，或者是不满足city='杭州’条件时循环结束。

![img](https://static001.geekbang.org/resource/image/df/d6/df4b8e445a59c53df1f2e0f115f02cd6.jpg)

  



# 第十七章 如何正确显示随机消息

- select word from words order by rand() limit 3;
  
  该语句执行的流程
  
  ![img](https://static001.geekbang.org/resource/image/2a/fc/2abe849faa7dcad0189b61238b849ffc.png)
  
  order by rand() 使用了内存临时表，内存临时表排序的时候使用了rowid排序方法
  
- tmp_table_size这个配置限制了内存临时表的大小，默认值是16M。如果临时表大小超过了tmp_table_size，那么内存临时表就会转成磁盘临时表。

- 优先队列排序示意图：

  ![img](https://static001.geekbang.org/resource/image/e9/97/e9c29cb20bf9668deba8981e444f6897.png)

- 随机取3个word值,sql语句应该如何编写

  1，取得整个表的行数，记为C；

  2，根据相同的随机方法得到Y1、Y2、Y3；

  3，再执行三个limit Y, 1语句得到三行数据。（limit Y，1的意思就是忽略前Y行数据，取后面一行）

  mysql> select count(*) into @C from t;

  set @Y1 = floor(@C * rand());

  set @Y2 = floor(@C * rand()); 

  set @Y3 = floor(@C * rand()); 

  select * from t limit @Y1，1； //在应用代码里面取Y1、Y2、Y3值，拼出SQL后执行 

  select * from t limit @Y2，1；

  select * from t limit @Y3，1；





# 第十八章 为什么SQL语句逻辑相同性能差距大

- 对索引字段做函数操作，可能会破坏索引值的有序性，因此优化其就决定放弃走树搜索的功能

- 在MySQL中，字符串和数字做比较的话，是将字符串转换成数字。如果**字段值为字符串**，查询的条件是**整型**则数据库会将字符串转化为数字

  mysql> select * from tradelog where tradeid=110717;（tradeid为varchar）

  优化器优化后：

  mysql> select * from tradelog where  CAST(tradid AS signed int) = 110717;

- 如果查询字段的编码与查询条件的编码方式不相同，也不能用到索引，mysql会把查询字段的编码强制转化为与查询条件相同的编码，CONVERT()函数进行转化

  



# 第十九章 为什么我只查一行的语句，也执行这么慢？

- 查询长时间不返回

  **语句：mysql> select * from t where id=1;**

  1，等待mdl锁，现在有一个线程正在表t上请求或者持有mdl写锁，把select语句堵住，解决办法找到mdl写锁然后kill掉，可以使用**show processlist** 查看状态，通过查询sys.schema_table_lock_waits这张表，我们就可以直接找出造成阻塞的process id，把这个连接用kill 命令断开即可。**select blocking_pid from sys.schema_table_lock_waits;**

  **语句：mysql> select * from information_schema.processlist where id=1;**

  2，等flush，出现Waiting for table flush状态的可能情况是：有一个flush tables命令被别的语句堵住了，然后它又堵住了我们的select语句。

  **语句：mysql> select * from t where id=1 lock in share mode**

  3，等行锁， 由于访问id=1这个记录时要加读锁，如果这时候已经有一个事务在这行记录上持有一个写锁，我们的select语句就会被堵住。 通过sys.innodb_lock_waits 查行锁

  mysql> **select * from t sys.innodb_lock_waits where locked_table=`'test'.'t'`\G** 

- 查询慢

  **语句：mysql> select * from t where id=1；**

  ![img](https://static001.geekbang.org/resource/image/46/8c/46bb9f5e27854678bfcaeaf0c3b8a98c.png)

  session B更新完100万次，生成了100万个回滚日志(undo log)。

  带lock in share mode的SQL语句，是当前读，因此会直接读到1000001这个结果，所以速度很快；而select * from t where id=1这个语句，是一致性读，因此需要从1000001开始，依次执行undo log，执行了100万次以后，才将1这个结果返回。

  注意，undo log里记录的其实是“把2改成1”，“把3改成2”这样的操作逻辑，画成减1的目的是方便你看图。





# 第二十章 幻读是什么

- 幻读指的是一个事务在前后两次查 询同一个范围的时候，后一次查询看到了前一次查询没有看到的行。
- 幻读只有在在可重复读的隔离级别下，普通的查询是**快照读**，是不会看到别的事务插入的数据的。因此， 幻读在“**当前读**”下才会出现。幻读仅专指“新插入的行”。
- 解决幻读的方法，引入间隙锁
- 间隙锁存在冲突的关系的是，往这个间隙中插入一条记录





# 第二十四章 mysql怎么保证主备一致

- 备库B跟主库A之间维持了一个长连接。主库A内部有一个线程，专门用于服务备库B的这个长连 接。一个事务日志同步的完整过程是这样的：
  1. 在备库B上通过change master命令，设置主库A的IP、端口、用户名、密码，以及要从哪个 位置开始请求binlog，这个位置包含文件名和日志偏移量。

  2. 在备库B上执行start slave命令，这时候备库会启动两个线程，就是图中的io_thread和 sql_thread。其中io_thread负责与主库建立连接。

  3. 主库A校验完用户名、密码后，开始按照备库B传过来的位置，从本地读取binlog，发给B。

  4. 备库B拿到binlog后，写到本地文件，称为中转日志（relay log）。

  5. sql_thread读取中转日志，解析出日志里的命令，并执行。

       ![img](https://static001.geekbang.org/resource/image/a6/a3/a66c154c1bc51e071dd2cc8c1d6ca6a3.png)

- binlog有三种格式

  1，statement：记录到binlog里的是语句原文，可能出现主备不一致的情况（索引使用错误）

  2，row：记录每个信息的详细情况，不会出现主备不一致，缺点是，很占空间

  3，mixed：可以利用statment格式的优点，同时又避免了数据不一致的风险。根据语句判断是否存在可能不一致的风险如果有则使用row，没有则使用statement

- 用binlog来恢复数据的标准做法是，用 mysqlbinlog工具解析出来，然后把解析结果整个发给MySQL执行。

  mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;

- 双M结构下解决循环复制的问题

  1. 规定两个库的server id必须不同，如果相同，则它们之间不能设定为主备关系；

  2. 一个备库接到binlog并在重放的过程中，生成与原binlog的server id相同的新的binlog；

  3. 每个库在收到从自己的主库发过来的日志后，先判断server id，如果跟自己的相同，表示这个日志是自己生成的，就直接丢弃这个日志。

![img](https://static001.geekbang.org/resource/image/20/56/20ad4e163115198dc6cf372d5116c956.png)



# 第二十五章 mysql是如何保持高可用的

- 数据同步有关的时间点主要包括以下三个： 

  1. 主库A执行完成一个事务，写入binlog，我们把这个时刻记为T1; 

  2. 之后传给备库B，我们把备库B接收完这个binlog的时刻记为T2; 

  3. 备库B执行完成这个事务，我们把这个时刻记为T3。 

  所谓主备延迟，就是同一个事务，在备库执行完成的时间和主库执行完成的时间之间的差值，也 

  就是T3-T1，可以在备库上执行showslave status命令，它的返回结果里面会显示**seconds_behind_master**，用于表示当前备库延迟了多少秒

- 主库和备库的时间不一致，备库会执行SELECTUNIX_TIMESTAMP()函数获取主库的时间

- 主备延迟来源

  1，主库于备库的性能差异

  2，备库压力大于（备库执行很多读的操作）

  3，执行了大的事务（主库上必须等事务执行完成才会写入binlog，再传给备库）

- 主备切换的策略

  1，**可靠性优先策略**（会存在一定的不可用时间）

  2，可用性有限策略（说不等主备数据同步，直接把连接切到备库，可能会导致主备不一致）

  

