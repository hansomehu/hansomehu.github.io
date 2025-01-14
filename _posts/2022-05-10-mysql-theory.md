---
layout: post
title: "MySQL理论笔记"
permalink: /mysql-theory
---

从SELECT * 到删库跑路——MySQL最全(qiang)笔记

## 1. 数据库的范式理论



#### **<u>什么是和为什么要？</u>**

数据库的设计存在四个问题：**冗余、删除、修改、插入异常**， 引入范式的原因就是要消除上述数据库的潜在问题。原则就是依赖分解，分解到最后，一张表里的主键就一个，比如学号-姓名，这样就完全不存在上面谈到的问题， 这就是第5范式。但第5范式毫无疑问是低效的。

设计数据库的时候记住，一个业务一张表，后续表关联即可，不要一张表怼进去一堆相互依赖的键。

不同级别的区别主要在于**字段之间依赖关系**， 高级别范式的依赖于低级别的范式，1NF 是最低级别的范式。

在实际生产当中时最高也就**满足第三范式**即可，高范式意味着高约束，会带来额外的开销，比如降低查询的效率是肯定的。



#### <u>**五大范式的简要概括**</u>

1NF：

**字段不可以再被拆分**，但要明白这中定义是存在主观性的，比如地址需不需要把省份和市单独拆出来？

例如：第一章表，key_id ｜name｜info(phone, address)；第二张表，id｜phone｜address 。第一张表中的info还可以再拆分，通常来说不满足1NF。

所以，标准就是，如果一张表中的一个字段包含多项信息，其中的某一项信息在另外一张表中是一个独立的字段，这种情况不满足1NF。

2NF：

**一张表是一个独立的消息**，在满足1NF的基础上，每一条记录都是被主键（可以是多字段的联合主键）唯一标识。如果主键有两个，存在某一条记录只单独依赖于主键元组中的某一个而与另外一个没关系，则这种表设计不满足2NF。解决的办法，把不被依赖的那个主键单独抽取出来和其他字段建立一张表

e.g ｜<u>学号</u>｜<u>课程号</u>｜成绩｜ 学号和课程号唯一标识成绩，满足2NF

3NF：

**非主键之间相互不能存在依赖**，第二范式中存在着A -> B -> <u>C</u> 这种循环依赖

<img src="/assets/images/mysql-theory/image-20220416201046057.png" alt="image-20220416201046057" style="zoom:67%;" />

BCNF Boyce-Codd： 

**主键内的属性键不能存在依赖关系**。例如，在关系R中，主键U，A是主键U的一个属性，Y为U的主属性，如果A -> Y那么该表设计不符合BCNF

4NF：

消除了表中的多值依赖（非平凡的多值依赖）

<img src="/assets/images/mysql-theory/image-20220416201425526.png" alt="image-20220416201425526" style="zoom:70%;" />

5NF：

完美范式





#### <u>表连接</u>

<img src="https://www.runoob.com/wp-content/uploads/2019/01/sql-join.png" alt="img" style="zoom:50%;" />







## 2. 存储引擎

引擎是相对于表来说的（表处理器）一个表能指定一种引擎

#### 1. InnoDB和MyISAM的区别

MyISAM：

- MySQL5.1及之前，MyISAM 是默认存储引擎，MyISAM 提供了大量的特性，包括全文索引、压缩、空间函数等；

- **事务缺陷。**不支持事务和行锁，最大的缺陷就是崩溃后无法安全恢复。对于只读的数据或者表比较小、可以忍受修复操作的情况仍然可以使用 MyISAM。支持整张表的排他锁，但是在表有读取查询的同时，也支持并发往表中插入新的记录；

- **三份文件。**MyISAM 将表存储在数据文件和索引文件中，分别以 `.MYD` 和 `.MYI` 作为扩展名，同时还有一个表结构文件。MyISAM 表可以包含<u>动态或者静态行</u>，MySQL 会根据表的定义决定行格式。MyISAM 表可以存储的行记录数一般**受限于可用磁盘空间或者操作系统中单个文件的最大尺寸**。

- 对于MyISAM 表，MySQL 可以**手动或自动执行检查和修复操作**，这里的修复和事务恢复以及崩溃恢复的概念不同。执行表的修复可能导致一些数据丢失，而且修复操作很慢。

- 支持多类型的全文索引。对于 MyISAM 表，即使是 BLOB 和 TEXT 等长字段，也可以基于其前 500 个字符创建索引。MyISAM 也**支持全文索引**，这是一种基于分词创建的索引，可以支持复杂的查询。

- **表锁问题开销大。**MyISAM 设计简单，数据以**紧密格式存储**，所以在某些场景下性能很好。MyISAM 最典型的性能问题还是**表锁问题**，如果所有的查询长期处于 Locked 状态，那么原因毫无疑问就是表锁。



InnoDB：

- InnoDB 是 MySQL 的默认**事务型引擎**，用来处理大量短期事务。InnoDB 的性能和**自动崩溃恢复特性**使得它在非事务型存储需求中也很流行，除非有特别原因否则应该优先考虑 InnoDB。

- InnoDB 的数据存储在表空间中，表空间由一系列数据文件组成。MySQL4.1 后 InnoDB 可以将每个表的数据和索引放在单独的文件中。

- InnoDB 采用 **MVCC 来支持高并发**，并且实现了四个标准的隔离级别。其默认级别是 `REPEATABLE READ`，并通过间隙锁策略防止幻读，间隙锁使 InnoDB 不仅仅锁定查询涉及的行，还会对索引中的间隙进行锁定防止幻行的插入。

- InnoDB 表是基于**聚簇索引**建立的，InnoDB 的索引结构和其他存储引擎有很大不同，聚簇索引对主键查询有很高的性能，不过它的二级索引中必须包含主键列，所以如果主键很大的话其他所有索引都会很大，因此如果表上索引较多的话主键应当尽可能小。

- InnoDB 的存储格式是**平台无涉**的，可以将数据和索引文件从一个平台复制到另一个平台。

- InnoDB 内部做了很多优化，包括从磁盘读取数据时采用的**可预测性预读**，能够自动在内存中创建加速读操作的自适应哈希索引，以及能够加速插入操作的插入缓冲区等。



#### 2. 二者的选择

InnoDB适合用于存在**大量事务操作、大表、频繁删改**等业务当中，由于InnoDB采用的聚簇索引在高性能的同时需要大量的内存开销，因此需要综合考虑业务和成本来进行选型。如果业务主要是**增查操作、小表**的话，使用MyISAM的性价比更高。



## 3. 索引

主要知识点就是基于**聚簇索引**方式的**B+树索引**

B树本身特点没法做成聚簇索引

#### <u>非聚簇（二级索引）</u>

**回表** 我们根据这个以c2列大小排序的B+树只能确定我们要查找记录的主键值，所以如果我们想根 据c2列的值查找到完整的用户记录的话，仍然需要到聚簇索引中再查一遍，这个过程称为回表 。也就 是根据c2列的值查询一条完整的用户记录需要使用到 2 棵B+树!

聚簇就是B+树的叶子结点上存储的是数据，找到目标叶子结点就相当于找到了数据

非聚簇就是B树上叶子结点存储的是数据所在的内存地址



<img src="/assets/images/mysql-theory/image-20220417162650819.png" alt="image-20220417162650819" style="zoom:25%;" />

<img src="/assets/images/mysql-theory/image-20220417163516576.png" alt="image-20220417163516576" style="zoom:25%;" />

**牢记B+树索引的扩张永远都是根节点不变，变的是子节点的层级和深度**



#### <u>索引的数据结构：</u>

record_type:  0 表示普通记录、 2 表示最小记 录、 3 表示最大记录、1 表示目录项中的数据

next_type：记录头信息的一项属性，表示下一条地址相对于本条记录的地址偏移量，我们用箭头来表明下一条记录是谁

存放用户记录的**叶子节点**代表的数据页可以存放**100条**用户记录 ，所有存放目录项记录的内节点代表的数据页可以存放**1000条**目录项记录 



#### **<u>为什么使用索引？</u>**

- 大大减少了服务器需要扫描的数据行数，索引的使用**避免了全盘扫描**

- 帮助服务器**避免进行排序和分组**，也就**不需要创建临时表**(B+Tree 索引是有序的，可以用于 ORDER BY 和 GROUP BY 操作。临时表主要是在排序和分组过程中创建，因为不需要排序和分组，也就不需要创建临时表)

- 将随机 I/O 变为**顺序 I/O**(B+Tree 索引是有序的，也就将**相邻的数据都存储在一起**)



#### **<u>索引在什么场景下使用？</u>**

对于**中到大型的表**，索引就非常有效；在小表和大表上建立索引的ROI很低

1. **字段的数值有唯一性的限制**。业务上具有唯一特性的字段，即使是组合字段，也必须建成唯一索引。(来源:Alibaba) 说明:不要以为唯一索引影响了 insert 速度，这个速度损耗可以忽略，但提高查找速度是明显的。
2. **频繁作为 WHERE 查询条件的字段**。某个字段在SELECT语句的 WHERE 条件中经常被使用到，那么就需要给这个字段创建索引了。尤其是在数据量大的情况下，创建普通索引就可以大幅提升数据查询的效率。 比如student_info数据表(含100万条数据)，假设我们想要查询 student_id=123110 的用户信息。

3. **经常GROUPBY和ORDERBY的列**。索引就是让数据按照某种顺序进行存储或检索，因此当我们使用 GROUP BY 对数据进行分组查询，或者 使用 ORDER BY 对数据进行排序的时候，就需要 对分组或者排序的字段进行索引 。如果待排序的列有多 个，那么可以在这些列上建立 组合索引 。

4. **UPDATE、DELETE 的 WHERE 条件列**。对数据按照某个条件进行查询后再进行 UPDATE 或 DELETE 的操作，如果对 WHERE 字段创建了索引，就 能大幅提升效率。原理是因为我们需要先根据 WHERE 条件列检索出来这条记录，然后再对它进行更新或 删除。 **如果进行更新的时候，更新的字段是非索引字段，提升的效率会更明显，这是因为非索引字段更 新不需要对索引进行维护。**<u>也就是不推荐对修改的目标字段建索引</u>

5. **DISTINCT 字段需要创建索引**。有时候我们需要对某个字段进行去重，使用 DISTINCT，那么对这个字段创建索引，也会提升查询效率。比如，我们想要查询课程表中不同的 student_id 都有哪些，如果我们对student_id 建立了索引，那么直接在索引中就能找到，并且结果都排好序的。

6. **多表 JOIN 连接操作时，对where字段建立索引，因为这才是最终的筛选字段**。同时应该注意，join的表不要超过3张，多重嵌套循环很影响效率。

7. **长文本建立索引时最好使用文本的一段前缀而非对整个文本建立索引**。这很好理解，长文本的重复率很低，比如博客文章，对前几个字符建立索引照样能精确匹配到结果，索引的时间和空间成本还都小了。

   `create table shop(address varchar(120) not null);`
   `alter table shop add index(address(12));`

8.  **区分度高(散列性高)的列适合作为索引**

9. **使用最频繁的列放到联合索引的左侧**



#### <u>什么场景下不适合建立索引？</u>

1. **在where中使用不到的字段，不要设置索引**
2. **数据量小的表不要创建索引**
3. **有大量重复数据的列不创建索引**。例如要在 100 万行数据中查找其中的 50 万行(比如性别为男的数据)，一旦创建了索引，你需要先访问 50 万次索引，然后再访问 50 万次数据表，这样加起来的开销比不使用索引可能还要大
4. **避免对经常更改数据的表创建过多的索引**
5. **避免重复创建或者冗余创建索引**





#### <u>什么是联合索引？</u>

联合索引是一种**非聚簇索引**，在设计联合索引时要关注最左匹配原则，索引文件具有 B-Tree 的最左前缀匹配特性，如果左边的值未确定，那么无法使用此索引。

联合索引就是在一个表的某项业务查询中的where子句中会用到多个字段，这些字段共同构建的索引叫做联合索引。联合索引的构建需要遵循一定的规则以达到最大效用，随意建立索引会使得索引的开销远大于索引所带来的回报。

具体的规则：

```sql
# 在都是=匹配的情况下，索引遵循最左匹配原则，应该把区别度最大的字段放在联合索引的最前端index (a,b,c)
SELECT * FROM table WHERE a = 1 and b = 2 and c = 3; 

# 对(b,a)建立索引,如果你建立的是(a,b)索引，那么只有a字段能用得上索引，毕竟最左匹配原则遇到范围查询就停止匹配
SELECT * FROM table WHERE a > 1 and b = 2; 

# 将=匹配放在最左
SELECT * FROM `table` WHERE a > 1 and b = 2 and c > 3; 

# IN 可以等价为等值匹配
SELECT * FROM `table` WHERE a IN (1,2,3) and b > 1; 
```





<img src="/assets/images/mysql-theory/image-20220416225235468.png" alt="image-20220416225235468" style="zoom:33%;" />



#### **<u>使用索引的代价：</u>**

**空间上的代价**

每建立一个索引都要为它建立一棵B+树，每一棵B+树的每一个节点都是一个数据页，一个页默认会占用**16KB** 的存储空间，一棵很大的B+树由许多数据页组成，那就是很大的一片存储空间。

**时间上的代价**

每次对表中的数据进行 操作时，都需要去修改各个B+树索引。而且我们讲过，B+树每 层节点都是按照索引列的值 而组成了双向链表 。不论是叶子节点中的记录，还是内节点中的记录(也就是不论是用户记录还是目录项记录)都是按照索引列的值从小到大的顺序而形成了一个单向链表。而增、删、改操作可能会对节点和记录的排序造成破坏，所以存储引擎需要额外的时间进行一些记录移位 ，页面分裂 、页面回收等操作来维护好节点和记录的排序。如果 我们建了许多索引，每个索引对应的B+树都要进行相关的维护操作，会给性能拖后腿。



#### **<u>B+树结构：</u>**

B+树是平衡查找树的加强版，在叶子结点上，每个节点之间都由指针维护，加大了节点之间的访达性

之所以使用B+树而不使用红黑树等其他类型的查找树主要原因有：

1. **更少的查找次数，**平衡树查找操作的时间复杂度等于树高 h，而树高大致为 O(h)=O(logdN)，其中 d 为每个节点的出度。红黑树的出度为 2，而 B+ Tree 的出度一般都非常大，所以红黑树的树高 h 很明显比 B+ Tree 大非常多，检索的次数也就更多。就是说B+树根节点的直接子节点的数量比红黑树远大，可以细分出很多区间。
2. **每次预读取会读出更多的数据，**由于mysql通常将数据存放在磁盘中，读取数据就会产生磁盘IO消耗。而B+树的非叶子节点中不保存数据，B树中非叶子节点会保存数据，通常一个节点大小会设置为磁盘页大小，这样B+树每个节点可放更多的key，B树则更少。这样就造成了，B树的高度会比B+树更高，从而会产生更多的磁盘IO消耗。



#### <u>**B树与B+树比较：**</u>

- **B+树层级更少**，查找更快，在数据量相同的情况下
- B+树查询速度稳定：由于**B+树所有数据都存储在叶子节点**，所以查询任意数据的次数都是树的高度h
- B+树有利于范围查找
- B+树全节点遍历更快：所有叶子节点**构成链表**，全节点扫描，只需遍历这个链表即可
- B树优点：如果在B树中查找的数据**离根节点近**，由于B树节点中保存有数据，那么这时查询速度比B+树快。

B+树高广度代替了B树的高深度，使得区间划分更加分散，加快查找速度



#### **<u>查找：</u>**

进行查找操作时，首先在根节点进行二分查找，找到一个 key 所在的指针，然后递归地在指针所指向的节点进行查找。直到查找到叶子节点，然后在叶子节点上进行二分查找，找出 key 所对应的 data。

插入删除操作记录会破坏平衡树的平衡性，因此在插入删除操作之后，需要对树进行一个分裂、合并、旋转等操作来维护平衡性。插入和删除操作带来了**较大的开销**



#### <u>索引的种类：</u>

1. 普通索引

在创建表的语句中增加 `INDEX(year_publication)`

2.  创建唯一索引

```sql
UNIQUE INDEX uk_idx_id(id)
SHOW INDEX FROM test1 \G
```

3. 主键索引

在创建表的时候随着主键的指定被指定，隐式和显式两种

```sql
# 删除主键索引
ALTER TABLE student
drop PRIMARY KEY ;
```

4. 单列索引

```sql
CREATE TABLE test2(
id INT NOT NULL,
name CHAR(50) NULL,
INDEX single_idx_name(name(20))
);
```

5. 组合索引

根据字段由前到后进行排序，假设index（a,b,c）

那么先以a进行排序然后再以b、c进行排序

```sql
CREATE TABLE test3(
id INT(11) NOT NULL,
name CHAR(30) NOT NULL,
age INT(11) NOT NULL,
info VARCHAR(255),
INDEX multi_idx(id,name,age)
);
```

6. 全文索引

```sql
CREATE TABLE `papers` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `title` varchar(200) DEFAULT NULL,
  `content` text,
  PRIMARY KEY (`id`),
  FULLTEXT KEY `title` (`title`,`content`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;

# 检索方式较普通方式有区别
SELECT * FROM papers WHERE MATCH(title,content) AGAINST (‘查询字符串’);
```



#### <u>隐式索引：</u>

由于索引存在开销，有些小表或小数据不用索引反而效率更高，应该在实际业务中灵活使用索引。但是，索引一旦删除再重建可能带来一系列额外问题和麻烦，那么当不用的时候可以**手动隐藏索引**，之后再查询的时候不会走索引，当有需要的时候可以再开启索引。这样能使开发更加灵活。



## 4. 性能优化

## 5. 事务

先看看事务的隔离级别和对应解决的事务问题

| *<u>Isolation Level</u>* | Dirty Read | Non-Repeatable Read | Phantom Read | Locked Read |
| :----------------------: | :--------: | :-----------------: | :----------: | :---------: |
|     READ UNCOMMITTED     |    Yes     |         Yes         |     Yes      |             |
|      READ COMMITTED      |     No     |         Yes         |     Yes      |             |
|     REPEATABLE READ      |     No     |         No          |     Yes      |             |
|       SERIALIZABLE       |     No     |         No          |      No      |             |



#### <u>事务的ACID特性是什么</u>？

**原子性(**atomicity**):**

原子性是指事务是一个不可分割的工作单位，要么全部提交，要么全部失败回滚到最初状态。

**一致性(**consistency**)：**

“一致”是指数据库中的数据是正确的，不存在矛盾。事务的一致性是指事务执行前后，数据都是正确的，不存在矛盾。如果执行后数据是矛盾的，事务就会回滚到执行前的状态（执行前是一致的）。也就是经过事务操作之后，无论事务是正常提交了或者是回滚了，最终业务的数据都是符合预期的。

**隔离型(**isolation**):**

事务的隔离性是指一个事务的执行 ，即一个事务内部的操作及使用的数据对 并发 的 其他事务是隔离的，并发执行的各个事务之间不能互相干扰。

**持久性(**durability**):
** 持久性是指一个事务一旦被提交，它对数据库中数据的改变就是 永久性的 ，接下来的其他操作和数据库故障不应该对其有任何影响。

持久性是通过 **事务日志** 来保证的。日志包括了 重做日志 和 回滚日志 。当我们通过事务对数据进行修改 的时候，首先会将数据库的变化信息记录到重做日志中，然后再对数据库中对应的行进行修改。这样做 的好处是，即使数据库系统崩溃，数据库重启后也能找到没有更新到数据库系统中的重做日志，重新执 行，从而使事务具有持久性。



#### <u>事务都有哪些状态？</u>

<img src="/assets/images/mysql-theory/image-20220418182740136.png" alt="image-20220418182740136" style="zoom:33%;" />



#### <u>事务有哪些启动方式？</u>

1. 显示启动

```sql
mysql> BEGIN;
# 或者, START TRANSACTION启动方式能选择参数READ ONLY、WRITE ONLY、WITH CONSISTENT SNAPSHOT
mysql> START TRANSACTION;

# 提交事务。当提交事务后，对数据库的修改是永久性的。 
mysql> COMMIT;
# 回滚事务。即撤销正在进行的所有没有提交的修改 
mysql> ROLLBACK;
# 将事务回滚到某个保存点。
mysql> ROLLBACK TO [SAVEPOINT]

SAVEPOINT a;
```

2. 隐式启动

```sql
mysql> SHOW VARIABLES LIKE 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (0.01 sec)

SET autocommit = OFF; #或
SET autocommit = 0;
```



#### **<u>mysql存在的事务问题？</u>**

1. **脏写(Dirty Write)**

对于两个事务 Session A、Session B，如果事务Session A 修改了 另一个 未提交 事务Session B 修改过 的数 据，那就意味着发生了脏写。脏写由于问题过于严重，在mysql中默认的隔离级别都已经解决了这个问题。

2. **脏读(Dirty Read )**

对于两个事务 Session A、Session B，Session A 读取 了已经被 Session B 更新 但还 没有被提交 的字段。之后若 Session B 回滚 ，Session A 读取 的内容就是 临时且无效 的。

Session A和Session B各开启了一个事务，Session B中的事务先将studentno列为1的记录的name列更新 为'张三'，然后Session A中的事务再去查询这条studentno为1的记录，如果读到列name的值为'张三'，而 Session B中的事务稍后进行了回滚，那么Session A中的事务相当于读到了一个不存在的数据，这种现象 就称之为 脏读 。

3. **不可重复读(Non-Repeatable Read)**

对于两个事务Session A、Session B，Session A 读取 了一个字段，然后 Session B 更新 了该字段。 之后

Session A 再次读取 同一个字段， 值就不同 了。那就意味着发生了不可重复读。

我们在Session B中提交了几个 隐式事务 (注意是隐式事务，意味着语句结束事务就提交了)，这些事务 都修改了studentno列为1的记录的列name的值，每次事务提交之后，如果Session A中的事务都可以查看 到最新的值，这种现象也被称之为 不可重复读 。

4. **幻读( Phantom )**

对于两个事务Session A、Session B, Session A 从一个表中 读取 了一个字段, 然后 Session B 在该表中 插入了一些新的行。 之后, 如果 Session A 再次读取 同一个表, 就会多出几行。那就意味着发生了幻读。Session A中的事务先根据条件 studentno > 0这个条件查询表student，得到了name列值为'张三'的记录; 之后Session B中提交了一个 隐式事务 ，该事务向表student中插入了一条新记录;之后Session A中的事务 再根据相同的条件 studentno > 0查询表student，得到的结果集中包含Session B中的事务新插入的那条记 录，这种现象也被称之为 幻读 。我们把新插入的那些记录称之为 幻影记录 。



#### <u>mysql的隔离级别有哪些？</u>

**READ UNCOMMITTED :**

读未提交，在该隔离级别，所有事务都可以看到其他未提交事务的执行结 果。不能避免脏读、不可重复读、幻读。

**READ COMMITTED :**

读已提交，它满足了隔离的简单定义:一个事务只能看见已经提交事务所做 的改变。这是大多数数据库系统的默认隔离级别(但不是MySQL默认的)。可以避免脏读，但不可 重复读、幻读问题仍然存在。

**REPEATABLE READ :**

可重复读，事务A在读到一条数据之后，此时事务B对该数据进行了修改并提 交，那么事务A再读该数据，读到的还是原来的内容。可以避免脏读、不可重复读，但幻读问题仍 然存在。这是MySQL的默认隔离级别。

**SERIALIZABLE :**

可串行化，确保事务可以从一个表中读取相同的行。在这个事务持续期间，禁止 其他事务对该表执行插入、更新和删除操作。所有的并发问题都可以避免，但性能十分低下。能避 免脏读、不可重复读和幻读。



#### <u>mysql的事务机制是如何实现的？</u>

##### **<u>UNDO LOG 称为回滚日志 ，</u>**回滚行记录到某个特定版本，用来保证事务的原子性、一致性。undo log保证原子性和一致性的原理是在事务被判断为失败的时候根据log中的记录来将某事务过程中的全部操作进行一个undo操作，也就是“回滚”。Undo日志的作用有两个，**回滚数据**和实**现MVCC**（InnoDB的MVCC通过undo log来实现）。

**undo log的存储结构：**

InnoDB对undo log的管理采用段的方式，也就是每个**rollback segment 回滚段**记录了 1024个undo log segment，而在每个undo log segment段中进行undo页的申请。从1.1版本开始InnoDB支持最大 128个rollback segment ，故其支持同时在线的事务限制提高到 了 **128*1024** 。

<u>p.s InnoDB引擎存储结构</u>

![img](http://images2015.cnblogs.com/blog/524341/201604/524341-20160416104932379-1446683950.jpg)



**undo log页是重复使用的**，当我们开启一个事务需要写undo log的时候，先去undo log segment中找到一个空闲的位置，然后申请undo页，每个undo页是16kb。当一个事务执行完成并commit了undo log之后，系统会判断当前的undo页剩余大小是否大于1/4，如果是的话下一个事务仍会被安排到该页中，以节约内存空间。

undo log根据业务类型被划分成了insert和update两种。其中在undo log执行删除操作的时候，这两种undo log的逻辑略有不同，insert直接删除，因为这类操作只对本事务可见。而update型的undo log可能需要提供MVCC机制，因此不能在事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。



**<u>REDO LOG 称为重做日志 ，</u>**提供再写入操作，恢复提交事务修改的页操作，用来保证事务的持久性。在事务执行过程中发生的一些修改处于事务原因不能直接持久化，否则会出现诸如不可重复读等问题，那么痛redo log在事务完毕之后或者定期进行持久化以保证事务的持久性。

<img src="/assets/images/mysql-theory/image-20220419004221989.png" alt="image-20220419004221989" style="zoom:33%;" />

Redo log可以简单分为以下两个部分：重做日志的缓冲 (redo log buffer) ，保存在内存中，是易失的。重做日志文件 (redo log file) ，保存在硬盘中，是持久的。前者通过某种持久化策略定期将内容持久化到磁盘中。

redo log的工作流程.jpg

<img src="/assets/images/mysql-theory/image-20220419004556227.png" alt="image-20220419004556227" style="zoom: 25%;" />

**redo log的刷盘策略**：InnoDB给出 innodb_flush_log_at_trx_commit 参数，共有三种策略。

设置为0:表示每次事务提交时不进行刷盘操作。(系统默认masterthread每隔1s进行一次重做日 志的同步)

设置为1:表示每次事务提交时都将进行同步，刷盘操作（系统默认值）

设置为2:表示每次事务提交时都只把 redo log buffer 内容写入 page cache，不进行同步。由os自己决定什么时候同步到磁盘文件。

**redo log的存储结构：**

mini transaction概念，一个事务可以包含若干条语句，每一条语句其实是由若干个 mtr 组成，每一个 mtr 又可以包含若干条redo日志

![image-20220419112627979](/assets/images/mysql-theory/image-20220419112627979.png)



**redo log以文件组的形式进行存储：**

![image-20220419113026566](/assets/images/mysql-theory/image-20220419113026566.png)

ib_logfile0-n指的是innodb中的文件组的数量编号，按环形进行存储。为了避免先前写过的数据被新数据覆盖，innodb设计了checkpoint概念，当目前的write pos到达checkpoint的时候说明**日志文件组**满了。系统将清空部分文件，之后将checkpoint和write pos移到空白区和已写区的开始。



**<u>mysql锁实现了事务的隔离性，</u>**即线程之间对同一资源的操作是互不可见的





#### <u>MySQL锁相关问题</u>？

**<u>mysql哪些操作需要加锁，加什么锁？</u>**

**读-读**情况下由于不设计数据的修改，每条线程读操作都具有幂等性，不需要加锁

**写-写**操作是最需要并非控制的情况，需要严格通过悲观锁机制来实现

**写-读**操作相对较为灵活，其中读部分可以通过MVCC来以较低开销的方式实现，而写部分还是得通过加锁的方式来实现。一般情况下我们当然愿意采用 MVCC 来解决 读-写 操作并发执行的问题



**<u>InnoDB引擎都有哪些锁？</u>**

<img src="/assets/images/mysql-theory/image-20220419151114654.png" alt="image-20220419151114654" style="zoom: 50%;" />

**从数据操作的类型划分:读锁、写锁**

读锁 :也称为 共享锁 、英文用 S 表示。针对同一份数据，多个事务的读操作可以同时进行而不会 互相影响，相互不阻塞的。

写锁 :也称为 排他锁 、英文用 X 表示。当前写操作没有完成前，它会阻断其他写锁和读锁。这样 就能确保在给定的时间里，只有一个事务能执行写入，并防止其他用户读取正在写入的同一资源。

**需要注意的是对于** InnoDB **引擎来说，读锁和写锁可以加在表上，也可以加在行上**



**从数据操作的粒度划分:表级锁、页级锁、行锁**

**<u>表级锁：</u>**

对整个表上读锁或是写锁。在对某个表执行一些诸如 ALTER TABLE 、 DROP TABLE 这类的 DDL 语句时，其他事务对这个表并发执行诸如SELECT、INSERT、DELETE、UPDATE的语句会发生阻塞。一般情况下，不会使用InnoDB存储引擎提供的表级别的 S锁 和 X锁 。只会在一些特殊情况下，比方说 崩 溃恢复 过程中用到。同时InnoDB引擎可以以显示方式`LOCK TABLES t READ/WRITE`来开启表锁



<u>表级锁-意向锁Intention Lock：</u>

意向锁是由存储引擎 ，用户无法手动操作意向锁，在为数据行加共享 / 排他锁之前， InooDB 会先获取该数据行所对应的意向锁。

【显式锁案例】

```sql
# 意向共享锁IS，事务有意向对表中的某些行加**共享锁**(S锁)。
# 事务要获取某些行的 S 锁，必须先获得表的 IS 锁。 
SELECT column FROM table ... LOCK IN SHARE MODE;

# 意向排他锁(intention exclusive lock, IX):事务有意向对表中的某些行加排他锁(X锁)
# 事务要获取某些行的 X 锁，必须先获得表的 IX 锁。 
SELECT column FROM table ... FOR UPDATE;
```

！意向锁不会与行级的共享 / 排他锁互斥!正因为如此，意向锁并不会影响到多个事务对不同数据行加排 他锁时的并发性。(不然我们直接用普通的表锁就行了)。意向锁之间互不排斥，但除了 IS 与 S 兼容外， 其他类型的意向锁和表的XS锁互斥。

意向锁的意义：我对表上了写的意向锁，那么其他线程不能再对该表里面的行上X锁。相当于节约了时间，不需要去遍历全表看看哪些行被锁了。**意向锁是在存在行锁场景下的表锁快速失败机制**



<u>自增锁</u>

在使用MySQL过程中，我们可以为表的某个列添加auto-increment属性。由于该属性是唯一的，那么在并发情况下需要保证对该字段的生成是唯一的，这个时候就需要自增锁了。

自增字段的插入方式：“Simple inserts” (简单插入，事先知道要插入多少行)， “Bulk inserts” (批量插入，事先不知道要插入多少行)，“Mixed-mode inserts” (混合模式插入，手动置顶部分id值，要插入多少行仍然是未知的)

采用AUTO_INC锁来实现自增锁，该锁的innodb_autoinc_lock_mode字段有三种取值，分别对应与不同锁定模式: 

(1)innodb_autoinc_lock_mode = 0(“传统”锁定模式)

在此锁定模式下，所有类型的insert语句都会获得一个特殊的表级AUTO-INC锁，用于插入具有 AUTO_INCREMENT列的表。这种模式其实就如我们上面的例子，即每当执行insert的时候，都会得到一个 表级锁(AUTO-INC锁)，使得语句中生成的auto_increment为顺序，且在binlog中重放的时候，可以保证 master与slave中数据的auto_increment是相同的。因为是表级锁，当在同一时间多个事务中执行insert的 时候，对于AUTO-INC锁的争夺会 限制并发 能力。

(2)innodb_autoinc_lock_mode = 1(“连续”锁定模式) 在 MySQL 8.0 之前，连续锁定模式是 默认 的。

在这个模式下，“bulk inserts”仍然使用AUTO-INC表级锁，并保持到语句结束。这适用于所有INSERT ... SELECT，REPLACE ... SELECT和LOAD DATA语句。同一时刻只有一个语句可以持有AUTO-INC锁。

对于“Simple inserts”(要插入的行数事先已知)，则通过在 mutex(轻量锁) 的控制下获得所需数量的 自动递增值来避免表级AUTO-INC锁， 它只在分配过程的持续时间内保持，而不是直到语句完成。不使用 表级AUTO-INC锁，除非AUTO-INC锁由另一个事务保持。如果另一个事务保持AUTO-INC锁，则“Simple inserts”等待AUTO-INC锁，如同它是一个“bulk inserts”。

(3)innodb_autoinc_lock_mode = 2(“交错”锁定模式) 从 **MySQL 8.0** 开始，交错锁模式是 **默认** 设置。

在此锁定模式下，自动递增值 保证 在所有并发执行的所有类型的insert语句中是唯一 且单调递增的。但是，由于多个语句可以同时生成数字(即，跨语句交叉编号)， **为任何给定语句插入的行生成的值可能不是连续的。**该模式不采用AUTO_INC锁，是效率最高的方式，但是当采用bin log对表进行SQL恢复时是不安全的。



**<u>行级锁：</u>**

记录锁：就是锁定一条记录，别的事务不能进行写操作。

间隙锁（Gap Locks）：解决幻读问题，在某记录的前面加入该锁，意味着不允许别的事务在id值为8的记录前边的间隙插入新记录 。其实就是 id列的值(i, i+1 )这个区间的新记录是不允许立即插入的，其中i是该条记录的id，id并不保证连续，只保证唯一。

临键锁（Next-Key Locks）：有时候我们既想锁住某条记录 ，又想阻止其他事务在该记录前边的记录进行操作 ，所以InnoDB就提出Next-Key Locks。该锁就是**记录锁和间隙锁的合并版**。

插入意向锁(Insert Intention Locks)：我们说一个事务在 插入一条记录时需要判断一下插入位置是不是被别的事务加了 gap锁 ( next-key锁 也包含 gap锁 )，如果有的话，插入操作需要等待，直到拥有 gap锁 的那个事务提交。**相当于是预定了某条记录的下一个独占锁（X锁）。**


**<u>页锁：</u>**

一个数据表在内存中对应多个页，一页中保存很多数据。每个层级的锁数量是有限制的，因为锁会占用内存空间， 锁空间的大小是有限的 。当某个层级的锁数量 超过了这个层级的阈值时，就会进行**锁升级** 。锁升级就是用更大粒度的锁替代多个更小粒度的锁，比如 InnoDB 中行锁升级为表锁，这样做的好处是占用的锁空间降低了，但同时数据的并发度也下降了。页锁会存在死锁的情况。



**<u>悲观锁与乐观锁：</u>**





<u>**聊一聊MVCC的原理，它是如何实现并发控制的？**</u>

MVCC 的实现依赖于：隐藏字段、Undo Log、Read View。其中两个隐藏字段记录了每条记录的事务ID（trx_id）和指向该记录的上一个版本的回滚指针（能定位到这个数据上一次被修改之前的值），这两个字段对用户不可见。Undo Log则是多版本的保证，当多个事务对某条记录进行操作之后，在undo log里都有体现，能够根据版本做还原。最后，Read View是做版本的管理，它是根据隐藏字段的事务ID和当前事务的ID来进行版本管理。MVCC解决不可重复读问题的方案就是，从Read View中查出历史版本的数据（快照读）并返回给用户。

Read View和事务是一对一的关系，维护了当前活跃事务的ID，也就是还没有结束的事务。MVCC无法实现`READ UNCOMMITED`和`SERIALISABLE`，前者是无差别返回最新版本数据，而后者通过高开销的锁方式实现。

![image-20220419205638616](/assets/images/mysql-theory/image-20220419205638616.png)

READ VIEW中的重要参数：

`creator_trx_id` 创建这个 Read View 的事务 ID

`trx_ids` 表示在生成ReadView时当前系统中活跃的读写事务的事务id列表 

`up_limit_id` 活跃的事务中最小的事务 ID

`low_limit_id` 表示生成ReadView时系统中应该分配给下一个事务的 id 值。low_limit_id 是系 统最大的事务id值，这里要注意是系统中的事务id，需要区别于正在活跃的事务ID

MVCC如何解决不可重复读和幻读问题：

总结起来，就是只能返回已经提交了的事务的数据给到用户。

具体流程：

每当我们进行一个事务操作的时候都会生成一个事务trx_id（可以粗粒度这么理解），同时数据库中的每条记录同时也有一个事务trx_id，这个id是最后一次修改过该数据的事务的id。

假设我们事务要select  * from [table_name]，数据库第一步会真的select全部数据出来，接下来就是判断哪些数据能返回给当前用户，就是通过事务数据和用户的事务id来进行判断

1、数据库会维护一个当前活跃事务（即还未commit的事务）的id列表，如果select * 的数据中有事务id大于`low_limit_id`的话，这些数据是不会返回的，因为说明正在操作该数据的事务甚至在当前用户开始事务之后才被开始，

2、如果数据的事务id小于`up_limit_id`那么这些数据可以返回，因为这些操作这些数据的事务已经提交了

3、如果数据的事务id处在`up_limit_id`和`low_limit_id`之间，并且事务id还存在于`trx_ids`表中，那么这些数据是不能被返回的，因为正在操作它们的事务还没有commit

4、如果数据的事务id等于`creator_trx_id`，也就是当前事务的id，那么这些数据能返回，因为该事务永远能访问自己正在操作的数据

*事务id的生成是随着时间的增长而单调递增的







**<u>你了解mysql事务锁的结构吗，它的原理是什么样的？</u>**

![image-20220419214102431](/assets/images/mysql-theory/image-20220419214102431.png)



在type_mode字段中，通过lock_mode的不同值来表示X、S、IS、IX等锁类型

![image-20220419215435915](/assets/images/mysql-theory/image-20220419215435915.png)



## 5. 并发与大流量应对

#### **<u>如何看待MySQL的并发问题？</u>**

数据库的并发问题和其他服务的并发问题存在一些区别。数据库在高并发下是有“倾向”的，读的并发是远大于写的。所以在解决MySQL并发问题时应该将读写分开看待，给予读更多的资源。常用的架构就是基于**主从复制**的**读写分离**。主从复制指的是一台主机配合多台从服务器，并且主服务器负责的功能有两个，一方面是写bin log主导主从之间的复制和同步，另外一方面是接受写数据的流量。

<img src="/assets/images/mysql-theory/image-20220420211025207.png" alt="image-20220420211025207" style="zoom:50%;" />

从较为宽泛的概念上来说，MySQL的性能优化应该首先从SQL优化上来考虑，也就是设计好合理的表体系和结构以及索引。上文中提到，数据库系统的读请求通常都是高于修改请求的，所以通过合理的索引设计能够改善大部分的流量并且这也是成本最低的方式。下表是一些各种优化高并发trade off。

![image-20220420215214737](/assets/images/mysql-theory/image-20220420215214737.png)





#### <u>MySQL主从复制的原理是什么，如何实现的？</u>

![image-20220420211507664](/assets/images/mysql-theory/image-20220420211507664.png)

主从同步的原理是基于bin log进行数据同步的。在主从复制过程中，会基于 3 个线程（主机log dump线程、从机I/O线程、从机SQL线程）来操作，一个主库线程，两个从库线程。

全过程的概述：

1. 主机负责写（修改）操作，每次的修改操作都备案到bin log当中
2. 主机定时或或按照其他策略向其他从服务器做bin log同步，从服务器拿到bin log后写入本地的relay log
3. 最后SQL线程从relay log中拿取进行到记录之后将数据写入从库

![image-20220420212607338](/assets/images/mysql-theory/image-20220420212607338.png)



#### <u>基于MyCat的一主一从实践</u>



#### <u>binlog都有哪些格式</u>

**STATEMENT模式** (基于SQL语句的复制(statement-based replication, SBR))，是mysql中默认的binlog格式

pros：

1. 不需要记录每一行的变化，减少了binlog日志量
2. 文件较小 binlog中包含了所有数据库更改信息，可以据此来审核数据库的安全等情况 （出事了看看问题sql是谁写的，明确锅谁来背）
3. binlog可以用于实时的还原，而不仅仅用于复制

cons：

1. 不是所有的UPDATE语句都能被复制，尤其是包含不确定操作的时候，以及当数据发生了改变再update的情况
2. 使用以下函数的语句也无法被复制:LOAD_FILE()、UUID()、USER()、FOUND_ROWS()、SYSDATE()
3. 更多的**行级锁产生**，开销大
   1. INSERT ... SELECT 会产生比 RBR **更多的行级锁**，需要遍历表从而生成了锁
   2. 复制需要进行全表扫描(WHERE 语句中没有使用到索引)的 UPDATE 时，需要比 RBR 请求**更多的行级锁**

4. 执行复杂语句得到的结果为简单数据，这种情况凭白增大了系统的开销
5. 数据表必须严格主服务器保持一致才行，否则通常语句来进行还原很有可能会导致复制出错



**Row模式，基于行的复制(row-based replication, RBR)**

直接把数据记录下来，包括哪条数据被修改了，修改成什么样了。

pros：

1. 执行 INSERT，UPDATE，DELETE 语句时**锁更少**。因为通过行的精确匹配来做到免除遍历加锁这个过程。
2. 任何情况都可以被复制，这对复制来说是最 安全可靠的，不存在数据错误的情况。(比如:不会出现某些特定情况下 的存储过程、function、trigger的调用和触发无法被正确复制的问题)
3. 从服务器上采用**多线程来执行复制**成为可能，因为是直接插入数据，不需要执行语句走加锁这个过程

cons：

1. binlog 大了很多，因为直接记录的数据
2. 复杂的回滚时 binlog 中会包含大量的数据
3. 主服务器上执行 UPDATE 语句时，所有发生变化的记录都会写到 binlog 中，而 SBR 只会写一 次，这会导致<u>频繁发生 binlog 的并发写</u>问题
4. 无法从 binlog 中看到都复制了些什么语句



**MIXED模式，混合模式复制(mixed-based replication, MBR)**

在Mixed模式下，一般的语句修改使用statment格式保存binlog。如一些函数，statement无法完成主从复 制的操作，则采用row格式保存binlog。

MySQL会根据执行的每一条具体的sql语句来区分对待记录的日志形式，也就是在Statement和Row之间选 择一种。

在开销和性能中间采取一种平衡。





#### <u>主从数据同步的一致性问题是如何产生的又该如何解决？</u>

主从数据不一致的根本是对从机没能及时从relay log中恢复数据。诱因主要有下面两个：

1. 进行主从同步的内容是二进制日志，它是一个文件，在进行**网络传输**的过程中就一定会产生延迟，这样就可能造成用户在从库上读取的数据不是最新的数据，也就是主从同步中的不一致性问题
2. 从机在消费relay log的时候耗时太长，可能是**从机的性能**问题导致的。要么是从机本身配置不够，或者是大部分资源被用来处理用户请求了



**关于减少主从延迟**

1. 降低多线程大事务并发的概率，优化业务逻辑
2. 优化SQL，避免慢SQL， 减少批量操作 
3. 提高物理带宽、从机服务器的配置
4. 实时性要求的业务读强制走主库，从库只做灾备，备份



**读写分离的背景下解决数据一致性问题**

读写分离情况下，解决主从同步中数据不一致的问题， 就是解决主从之间**数据复制方式**的问题，如果按照数据一致性从弱到强来进行划分，有以下 3 种复制方式。

**异步复制，弱一致性**

![image-20220421132412220](/assets/images/mysql-theory/image-20220421132412220.png)

在主机写完bin log并且事务提交之前，从机读bin log到relay log后执行数据同步操作。很显然，当写完bin log后事务失败，或者是log的传输、消费耗时很久，都无法保证数据一致性。



**半同步复杂，勉强能接受的一致性**

![image-20220421132638484](/assets/images/mysql-theory/image-20220421132638484.png)

半同步复制和异步复制的核心区别在于，主机要收到从机完成relay log的ack后才提交事务，这样一来保证了事务过程中的一致。但是这种硬等待开销大，并且在实时性要求高的场景下难以满足要求。



**组复制，强一致性**

![image-20220421132846457](/assets/images/mysql-theory/image-20220421132846457.png)

首先我们将多个节点共同组成一个复制组，在 执行读写(RW)事务 的时候，需要通过一致性协议层 (Consensus 层)的同意，也就是读写事务想要进行提交，必须要经过组里“大多数人”(对应 Node 节 点)的同意，大多数指的是同意的节点数量需要大于 (N/2+1)，这样才可以进行提交，而不是原发起 方一个说了算。而针对 只读(RO)事务 则不需要经过组内同意，直接 COMMIT 即可。

在一个复制组内有多个节点组成，它们各自维护了自己的数据副本，并且在一致性协议层实现了原子消息和全局有序消息，从而保证组内数据的一致性。

MGR 将 MySQL 带入了数据强一致性的时代，是一个划时代的创新，其中一个重要的原因就是MGR 是基 于 Paxos 协议的。**Paxos** 算法是由 2013 年的图灵奖获得者 Leslie Lamport 于 1990 年提出的，有关这个算 法的决策机制可以搜一下。事实上，Paxos 算法提出来之后就作为 被广泛应用，比如 Apache 的 ZooKeeper 也是基于 Paxos 实现的。

组复制需要条件，在较大规模的场景下组复制更加能够发挥出其优势，保证集群当中又一半以上的机器持有最新的数据。这种机制使得从机对relay log的消费延迟得到容忍，也能保证最新的数据是冗余备份的，能够被访问到且安全。







## 6. 架构

 <img src="/assets/images/mysql-theory/image-20220417182655660.png" alt="image-20220417182655660" style="zoom:67%;" />

MySQL8以上已经把Caches&Buffers这块给砍掉了（鸡肋），其他部分依旧



## 7.MyCat

MyCat能提供到的功能包括

1. 读写分析
2. 分布式数据库（数据库/表拆分）
3. 整合多数据源（Not Only MySQL）



#### 原理图，核心概念——”拦截“

![image-20220424222523798](/assets/images/mysql-theory/image-20220424222523798.png)

https://github.com/MyCATApache/Mycat-Server

通过配置文件的方式来定义好分库分表的规则（当然你得自己先操作数据库），然后再代码层面做到数据和业务代码解耦。mycat还存在的问题其实很明显，通过xml配置文件的方式来做分库分表的拦截管理是不是用户友好的方式呢？

mycat的数据库逻辑配置、分库分表等通过schema.xml来确定逻辑库和物理库的对应关系



MyCat属于MySQL的一种server，所以登录mysql的方式就是用mysql登录命令，但是指定到mycat的端口。

mysql -umycat -p123456 -P8066 -h[your ip]      数据端口

mysql -umycat -p123456 -P9066 -h[your ip]      管理员端口

mycat的主从复制从连接点开始进行复制，而redis是从头开始把全部数据写入RDB传给从机进行复制



配置主从同步

配置读写分离

1. 在schema.xml中定义了逻辑库和物理库对应
2. 在schema.xml中定义datahost的balance，1为单主单从，3为双主双从



双主双从

1. 各自的主从配置
2. 两个主机之间配置主备复制
3. 将四台及其都配置进schema.xml并将balance值改为1（3也可以，参考下表中mycat配置常用参数）



``` bash
# balance属性负载均衡类型

balance=”0”, 不开启读写分离机制，所有读操作都发送到当前可用的 writeHost 上
balance=”1”，全部的 readHost 与 stand by writeHost 参与 select 语句的负载均衡
balance=”2”，所有读操作都随机的在 writeHost、 readhost 上分发。
balance=”3”， 所有读请求随机的分发到 wiriterHost 对应的 readhost 执行,writerHost 不负担读压力


# writeType属性负载均衡类型，目前的取值有3种：

1.writeType="0", 所有写操作发送到配置的第一个writeHost，第一个挂了切到还生存的第二个writeHost，重新启动后已切换后的为准，切换记录在配置文件中:dnindex.properties.

2.writeType="1"，所有写操作都随机的发送到配置的writeHost，1.5以后废弃不推荐。

3.writeType="2"，不执行写操作

#switchType属性

-1 表示不自动切换(普通的读写分离最好是不要自动切换，避免了将数据写进slave的可能性。除非是双主)

1 默认值，自动切换

2 于MySQL主从同步的状态决定是否切换,心跳语句为 show slave status

3 基于MySQLgalarycluster的切换机制（适合集群）（1.4.1）心跳语句为show status like‘wsrep%’

# dbType属性
指定后端连接的数据库类型，目前支持二进制的mysql协议，还有其他使用
JDBC连接的数据库。例如：mongodb、oracle、spark等
```

