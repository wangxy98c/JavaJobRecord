# 基础篇

## Mysql执行流程

![image-20240310上午114957138](./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240310%E4%B8%8A%E5%8D%88114957138.png)

+ Server层：负责建立连接、分析和执行SQL
  + 包括连接器、查询缓存、解析器、预处理器、优化器、执行器等。
  + 内置函数（如日期、时间、数学和加密函数等）
  + 跨存储引擎的功能（如存储过程、触发器、视图等）
+ 存储引擎层
  + 支持 InnoDB、MyISAM、Memory 等多个存储引擎。MySQL 5.5 版本开始， **InnoDB** 成为了 MySQL 的**默认存储引擎**
  + 索引数据结构，就是由存储引擎层实现的。==InnoDB 支持索引类型是 B+树==

### 连接器

+ 与客户端进行 TCP 三次握手**建立连接**；
+ **校验**客户端的用户名和密码，如果用户名或密码不对，则会报错；
+ 如果用户名和密码都对了，会读取该用户的**权限**，然后后面的权限逻辑判断都基于此时读取到的权限；

**==如果一个用户已经建立了连接，即使管理员中途修改了该用户的权限，也不会影响已经存在连接的权限==**。修改完成后，只有再新建的连接才会使用新的权限设置

想知道当前 MySQL 服务被多少个客户端连接了，你可以执行 `show processlist` 命令进行查看。

![image-20240310下午121657161](./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240310%E4%B8%8B%E5%8D%88121657161.png)

+ 空闲连接不会一直占用：由`wait_timeout` 参数控制，默认8小时
+ 也可以用kill connection + id 命令断开连接
+ MySQL 服务支持的==**最大连接数**==由 max_connections 参数控制，默认==151==。
+ MySQL的连接也有短连接和长连接。一般**推荐**使用**长连接**
  + 使用长连接后可能会占用内存增多，因为 MySQL 在执行查询过程中临时使用内存管理连接对象，<u>这些连接对象资源只有在连接断开时才会释放</u>。
    + 定期断开长连接
    + 客户端主动重置连接：调用函数`mysql_reset_connection()`

### 查询缓存

连接器得工作完成后，客户端就可以向 MySQL 服务发送 SQL 语句了，MySQL 服务收到 SQL 语句后，就会解析出 SQL 语句的第一个字段，看看是什么类型的语句。

如果 SQL 是查询语句（select 语句），MySQL 就会先去查询缓存（ Query Cache ，不是Innodb引擎中的buffer pool）里查找缓存数据，看看之前有没有执行过这一条命令，这个查询缓存是以 key-value 形式保存在内存中的，key 为 SQL 查询语句，value 为 SQL 语句查询的结果。命中就直接返回，没命中就继续执行并把结果存入缓存。但是其实**查询缓存挺鸡肋**的。MySql8.0之后不会再有此步骤（之前也可以关）

### 解析SQL

1. 词法分析：识别关键字，如select、from
2. 语法分析：构建语法树

+ **==表不存在或者字段不存在，并不是在解析器里做==**，解析器只负责检查语法和构建语法树

### 执行SQL

1. 预处理阶段

   ==检查**表和字段**==，将`select *` 的 **`*` 扩展**为表上的所有列。

2. 优化阶段

   **优化器主要负责将 SQL 查询语句的==执行方案确定==下来**。比如索引选择

3. ==执行==阶段

   执行器和存储引擎交互（交互以记录为单位）。`read_first_record()`到 `read_record()`。

   + 主键索引查询
     + 主键 id 是唯一，不会有 id 相同的记录，所以优化器决定选用访问类型为 const 进行查询。因为优化器选择的访问类型为 const，`read_record()`指针被指向为一个永远返回 - 1 的函数，执行器退出循环。
   + 全表扫描
     + 优化器选择的访问类型为 all，`read_record()`指针指向的还是 InnoDB 引擎全扫描的接口，所以接着向存储引擎层要求继续读刚才那条记录的下一条记录
   + 索引下推：节省很多回表操作(回表：第一条索引条件满足即返回到Server层，不判断第二个条件)。索引下推后，第一条满足后顺势判断第二个条件，都满足后才返回到Server层

## 记录是如何存储的

### 数据文件

假设这里有一个名为 my_test 的 database，该 database 里有一张名为 t_order 数据库表。进入 /var/lib/mysql/my_test 目录。有三个文件：

+ **db.opt**：存储当前数据库的默认字符集和字符校验规则
+ **t_order.frm**：t_order 的**表结构**会保存在这个文件，主要包含表结构定义。
+ **t_order.ibd**：t_order 的**表数据**会保存在这个文件，表数据既可以存在共享表空间文件（文件名：ibdata1）里，也可以存放在独占表空间文件（文件名：表名字.ibd）。由参数 `innodb_file_per_table` 控制

### 表空间文件的结构

**表空间由==段（segment）、区（extent）、页（page）、行（row）==组成**

<img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240310%E4%B8%8B%E5%8D%8821820386.png" alt="image-20240310下午21820386" style="zoom:50%;" />

+ **记录是按照行来存储**的，但是数据库的读取并不以「行」为单位（一次IO只处理一行效率低下），**InnoDB 的数据是==按「页」为单位来读写==的**，**默认每个页的大小为 ==16KB==**。
  + 页的类型有很多，常见的有数据页、undo 日志页、溢出页等等。**数据表中的行记录是用「数据页」来管理**的
+ InnoDB 存储引擎是用 B+ 树来组织数据的。B+ 树中每一层都是通过双向链表连接起来的，如果是以页为单位来分配存储空间，那么链表中相邻的两个页之间的物理位置并不是连续的，可能离得非常远，那么磁盘查询时就会有大量的随机I/O，随机 I/O 是非常慢的。解决这个问题也很简单，就是让**链表中相邻的页的物理位置也相邻**，这样就可以使用顺序 I/O 了，那么在范围查询（扫描叶子节点）的时候性能就会很高。
  + **在==表中数据量大的时候==，为某个索引分配空间的时候就不再按照页为单位分配了，而是==<u>按照区（extent）为单位分配==</u>。每个区的大小为 ==1MB==，对于 16KB 的页来说，连续的 64 个页会被划为一个区，这样就使得链表中相邻的页的物理位置也相邻，就能使用顺序 I/O 了**。
+ 段是由多个区（extent）组成的，段一般分为数据段、索引段和回滚段等
  + **索引段**：存放 B + 树的**非叶子节点**的区的集合；
  + **数据段**：存放 B + 树的**叶子节点**的区的集合；
  + **回滚段**：存放的是**回滚数据**的区的集合

### InnoDB==行格式==

1. Redundant ：不紧凑，基本没人用了
2. Compact ：紧凑（为了让一个数据页中可以存放更多的行记录）。基于此改进出下面两种
   1. ==Dynamic== ：紧凑，5.7之后的默认行格式
   2. Compressed ：紧凑

### COMPACT行格式

#### 额外信息

![image-20240310下午23143474](./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240310%E4%B8%8B%E5%8D%8823143474.png)

1. ==变长字段长度列表==

   同一行内的**变长字段**（如varchar）的真实数据**占用的字节数**会按照列的顺序==**逆序存放**==。**NULL 是不会存放在行格式中记录的真实数据部分里的**，所以「变长字段长度列表」里不需要保存值为 NULL 的变长字段的长度

> 逆放原因：使得位置靠前记录的真实数据和其对应的字段长度信息可以在一个CPU-cache-line中，提高cache命中率

+ 当数据表没有变长字段的时候，比如全部都是 int 类型的字段，这时候表里的行格式就不会有**「变长字段长度列表」（不必须）**

2. ==Null值列表==

   + 把值为NULL的列存储到null值列表中。

   + **允许为NULL的列，每个列对应一个bit**（按列顺序的**逆序排列**），1为null，0为非空
   + 但是InnoDB用整数字节的二进制表示Null值列表（8bit的倍数，不足的补齐，且从右往左）
   + Null值列表也**不是必须的**，比如所有字段都是not-NULL
   
   > 以第二行为例：列123都可以为null。逆序存放100=4。同理第三列：110=6

![image-20240312下午73008763](./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240312%E4%B8%8B%E5%8D%8873008763.png)

3. ==记录头信息==（几个重要的）

   + **delete_mask** ：标识此条数据是否被删除。从这里可以知道，我们执行 detele 删除记录的时候，并不会真正的删除记录，只是将这个记录的 delete_mask 标记为 1。

   + **next_record**：下一条记录的位置。从这里可以知道，记录与记录之间是通过链表组织的。在前面我也提到了，指向的是下一条记录的「记录头信息」和「真实数据」之间的位置，这样的好处是向左读就是记录头信息，向右读就是真实数据，比较方便。

   + **record_type**：表示当前记录的类型，==**0表示普通记录，1表示B+树非叶子节点记录，2表示最小记录，3表示最大记录**==

     > + **最小记录**：是指一个B+树非叶子节点的记录，该记录包含一个键值对，其键是整个子树中的最小键值。这个记录通常位于B+树的非叶子节点的起始位置，用于指示该节点所代表的子树的最小键值。同理最大纪录

#### 真实数据

1. row_id

   如果我们建表的时候**指定了主键或者唯一约束列**，那么**就没有 row_id** 隐藏字段了。**如果既没有指定主键，又没有唯一约束，那么 InnoDB 就会为记录添加 row_id 隐藏字段**。row_id**不是必需的**，占用 **6 个字节**。

2. trx_id

   **事务id**，表示这个<u>数据是由哪个事务生成的</u>。 trx_id是**必需的**，占用 6 个字节。

3. roll_pointer

   这条记录上一个版本的指针。roll_pointer 是必需的，占用 7 个字节

### varchar(n)中n的最大取值是多少

**MySQL一行数据（所有列）的最大字节数是 65535（不包含 TEXT、BLOBs 这种大对象类型），其中包含了 storage overhead**（变长字段长度列表、Null值列表）。最终若以ascii码为字符集，算下来n最大值为65535-2-1=65532

### 行溢出

MySQL 中磁盘和内存交互的基本单位是页，一个页的大小一般是 `16KB`，也就是 `16384字节`。这时一个页可能就存不了一条记录。这个时候就会==**发生行溢出，多的数据就会存到另外的「溢出页」中**==。当发生行溢出时，在记录的真实数据处只会保存该列的一部分数据，而把剩余的数据放在「溢出页」中，然后**在==真实数据处用 20 字节存储指向溢出页的地址==**（Compact行格式的处理方式）

<img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240312%E4%B8%8B%E5%8D%8880914677.png" alt="image-20240312下午80914677" style="zoom:50%;" />

+ Compressed、Dynamic这两种的行格式处理行溢出

  这两种格式采用==**完全的行溢出**==方式，记录的真实数据处不会存储该列的一部分数据，**只存储 20 个字节的指针来指向溢出页**。而==实际的**数据都存储在溢出页**==中，看起来就像下面这样

  <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240312%E4%B8%8B%E5%8D%8881134336.png" alt="image-20240312下午81134336" style="zoom:50%;" />

# 索引篇

## 常见面试题

+ 按「数据结构」分类：**B+tree索引、Hash索引、Full-text索引**。
+ 按「物理存储」分类：**聚簇索引（主键索引）、二级索引（辅助索引）**。
+ 按「字段特性」分类：**主键索引、唯一索引、普通索引、前缀索引**。
+ 按「字段个数」分类：**单列索引、联合索引**。

### 按数据结构分类

<img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240312%E4%B8%8B%E5%8D%8883413491.png" alt="image-20240312下午83413491" style="zoom:50%;" />

在创建表时，InnoDB 存储引擎会根据不同的场景选择不同的列作为索引：

+ 如果==**有主键**，**默认会使用主键**==作为聚簇索引的索引键（key）；
+ 如果**没有主键**，就选择==**第一个不包含 NULL 值的唯一列**==作为聚簇索引的索引键（key）；
+ 在上面两个**都没有**的情况下，InnoDB 将==**自动生成==一个隐式自增 id** 列作为聚簇索引的索引键（key）；

**创建的主键索引和二级索引默认使用的是 B+Tree 索引**。B+Tree 是一种多叉树，叶子节点才存放数据，非叶子节点只存放索引，而且每个节点里的数据是**按主键顺序存放**的。

> B+Tree 存储千万级的数据只需要 3-4 层高度就可以满足，这意味着从千万级的表查询目标数据最多需要 3-4 次磁盘 I/O，所以**B+Tree 相比于 B 树和二叉树来说，最大的优势在于查询效率很高，因为即使在数据量很大的情况，查询一个数据的磁盘 I/O 依然维持在 3-4次**

#### 通过二级索引查询商品数据的过程

主键索引的 B+Tree 和二级索引的 B+Tree 区别如下：

+ **==主键索引的 B+Tree 的叶子节点存放的是实际数据==**，所有完整的用户记录都存放在主键索引的 B+Tree 的叶子节点里；

  + > + InnoDB 存储引擎：B+ 树索引的叶子节点保存数据本身；
    > + MyISAM 存储引擎：B+ 树索引的叶子节点保存数据的物理地址；

+ **==二级索引的 B+Tree 的叶子节点存放的是主键值==**，而不是实际数据。（**再根据查询到的主键查询数据的过程叫做「回表」**。（当然，如果查询的信息就是主键则不需要回表）**这种在二级索引的 B+Tree 就能查询到结果的过程就叫作「覆盖索引」，也就是只需要查一个 B+Tree 就能找到数据**

#### 为什么选择B+树

+ B+树每一层间点之间都是双向链表

1. **B+Tree vs B Tree**

   + B+Tree 只在叶子节点存储数据，而 B 树 的非叶子节点也要存储数据，所以 ==**B+Tree 的单个节点的数据量更小**==，在**相同的磁盘 I/O 次数下，就能查询更多的节点**

   + 另外，B+Tree **叶子节点采用的是双链表连接**，**适合** MySQL 中常见的**基于范围的==顺序查找==**，而 B 树无法做到这一点。
   + B+树在插入删除时，==树的结构变化很小==（不太用调整）。效率更高
   + 

2. **B+Tree vs 二叉树**
   +  N 个叶子节点的 B+Tree，其搜索复杂度为$O(log_dN)$。其中 d 表示节点允许的最大子节点个数为 d 个。在实际的应用当中， d 值是大于100的，这样就保证了，即使数据达到千万级别时，B+Tree 的高度依然维持在 3~4 层左右，也就是说一次数据查询操作只需要做 3~4 次的磁盘 I/O 操作就能查询到目标数据。
   + 二叉树检索到目标数据所经历的磁盘 I/O 次数要更多。

3. **B+Tree vs Hash**
   +  ==Hash 表不适合做范围查询==，它更适合做等值的查询。

### 按字段特性分类

1. ==主键索引==

2. ==唯一索引==

   唯一索引建立在 **UNIQUE 字段上**的索引，一张表**可以有==多个==唯一索引**，索引列的值必须唯一，但是**允许有空值**

   ```sql
   # 创建唯一索引的表
   CREATE TABLE table_name  (
     ....
     UNIQUE KEY(index_column_1,index_column_2,...) 
   );
   # 建表后，创建唯一索引
   CREATE UNIQUE INDEX index_name
   ON table_name(index_column_1,index_column_2,...); 

3. ==普通索引==

   ```sql
   CREATE TABLE table_name  (
     ....
     INDEX(index_column_1,index_column_2,...) 
   );
   # 建表后，创建
   CREATE INDEX index_name
   ON table_name(index_column_1,index_column_2,...); 
   ```

4. ==前缀索引==

   前缀索引是指对字符类型字段的**前几个字符**建立的索引，而不是在整个字段上建立的索引，前缀索引**可以建立在字段类型为 char、 ==varchar、binary、varbinary==** 的列上。

   ```sql
   CREATE TABLE table_name(
       column_list,
       INDEX(column_name(length))
   ); 
   # 建表后，创建
   CREATE INDEX index_name
   ON table_name(column_name(length)); 
   ```

#### 按字段个数分类

1. ==单列索引==

2. ==联合索引（复合索引）==

   将多个字段组合成一个索引，该索引就被称为联合索引。

   ```sql
   # 把product_no和name 联合索引
   CREATE INDEX index_product_no_name ON product(product_no, name);
   ```

   使用联合索引时，存在**最左匹配原则**，也就是按照最左优先的方式进行索引的匹配。

   > 比如，如果创建了一个 `(a, b, c)` 联合索引，如果查询条件是以下这几种，就可以匹配上联合索引：
   >
   > + where a=1；
   > + where a=1 and b=2 and c=3；
   > + where a=1 and b=2；
   >
   > 需要注意的是，因为有**查询优化器**，所以 **a 字段在 where 子句的顺序并不重要**。

3. 索引下推：index condition

   **索引下推优化**（index condition pushdown)， **可以在联合索引遍历过程中，对「联合索引中包含的字段」先做判断，直接过滤掉不满足条件的记录，减少回表次数**。

#### 索引区分度

**建立联合索引时，要把区分度大的字段排在前面，这样区分度大的字段越有可能被更多的 SQL 使用到**

$区分度=\frac {该字段值去重后的数量}  {总行数}$。比如性别区分度就很小（因为去重后只有男女），而uuid这种就区分度很大

### 什么时候需要索引（优缺点）

+ 提高查询速度
+ 占用空间大、维护耗时、降低增删改效率

#### 适用场景

1. 字段有==唯一性==：比如Id
2. ==经常用`where` 查询==的字段
3. ==经常用`group by ` `order by`的字段==

#### 不适合用的

1. ==经常更新==的字段
2. 大量重复数据（如男女）
3. 数据量小的时候

### 索引优化

+ **索引最好设置为 NOT NUL**
  
  + 索引列存在 NULL 就会导致优化器在做索引选择的时候更加复杂，更加难以优化
  + 如果表中存在允许为 NULL 的字段，那么至少会用 1 字节空间存储 NULL 值列表
  
+ **前缀索引优化**；

  + ordey by无法使用前缀优化
  + 无法把前缀索引 用作 覆盖索引

+ **覆盖索引优化**；

  + 不需要查询出包含整行记录的所有信息，也就减少了大量的 I/O 操作

+ **主键索引最好是自增的**；

  > **如果我们使用非自增主键**，由于每次插入主键的索引值都是随机的，因此每次插入新的数据时，就可能会插入到现有数据页中间的某个位置，这将不得不移动其它数据来满足新数据的插入，甚至需要从一个页面复制数据到另外一个页面，我们通常将这种情况称为**页分裂**。**页分裂还有可能会造成大量的内存碎片，导致索引结构不紧凑，从而影响查询效率**。

  + **主键字段的长度不要太大**

  > 主键字段长度越小，意味着二级索引的叶子节点越小（二级索引的叶子节点存放的数据是主键值），这样二级索引占用的空间也就越小

+ **防止索引不生效**（有时查询语句不太对）

  + 当我们使用==**左或者左右模糊匹配**==的时候，也就是 `like %xx` 或者 `like %xx%`这两种方式都**可能**造成索引失效；（如果数据库表中的字段**只有主键+二级索引**，那么即使使用了左模糊匹配，也不会走全表扫描（type=all），而是走全扫描二级索引树(type=**index**)。）

  + 当我们**在查询条件中对「索引列」做了==计算、函数、类型转换==操作**，这些情况下都会造成索引失效；

    > 比如`id+1=10` 失效但`id=10-1`不失效

  + 联合索引要能正确使用需要遵循==最左匹配原则==，也就是按照最左优先的方式进行索引的匹配，否则就会导致索引失效。

  + 在 **WHERE 子句中，如果在 OR 前的条件列是索引列，而在 ==OR== 后的条件列不是索引列，那么索引会失效**。

    > `where id=1 or age=18`：只有一个条件列是索引列是没有意义的，索引条件是OR时，**只要有条件列不是索引列，就会进行全表扫描。**（id是主键，只要把age也设置为索引即可解决）

  + 如果**索引字段是字符串**类型，但是在条件查询中，输入的**参数是整型**的话会导致**索引失效**；但是如果索引字段是整型类型，查询条件中的输入参数即使字符串，是不会导致索引失效（反之不会）

    > 原因：和MySQL的数据类型转换规则有关。**MySQL 在遇到字符串和数字比较的时候，会==自动把字符串转为数字==，然后再进行比较**。（于是上面的索引字段被转化为数字导致失效）

#### 扫描方式Type

Type字段描述了扫描方式，**执行效率从「低到高」的顺序为**：

+ All（全表扫描）；
+ index（全索引扫描）；代表着是通过全扫描二级索引的 B+ 树的方式查询到数据的。
+ range（索引范围扫描）；从此级别开始，索引作用越来越明显
+ ref（非唯一索引扫描）；
+ eq_ref（唯一索引扫描）；
+ const（结果只有一条的主键或唯一索引扫描）。

> 另外，关注extra显示的结果，
>
> + **Using filesort** ：*当查询语句中包含 group by 操作，而且无法利用索引完成排序操作的时候*， 这时不得不选择相应的排序算法进行，甚至可能会通过文件排序，**效率是很低的**，所以要避免这种问题的出现。
> + **Using temporary**：使了用临时表保存中间结果，MySQL 在对查询结果排序时使用临时表，*常见于排序 order by 和分组查询 group by*。**效率低**，要避免这种问题的出现。
> + **Using index**：所需数据只需在索引即可全部获得，不须要再到表中取数据，也就是使用了覆盖索引，避免了回表操作，**效率不错**

### 数据页和B+树

InnoDB数据页的默认大小是16KB。数据页包括：

1. ==文件头==：表示页的信息，有**两个分别指向上一个数据页和下一个数据页的==指针==**（类似双向链表）

2. 页头：页的状态信息

3. ==最小和最大纪==录：两个**虚拟的伪纪录**，表示页中最小最大纪录

4. 用户记录：存储行纪录内容，**数据页中的记录按照「主键」顺序组成「单向链表」**。

   + 数组链表的存储方式。且目录分块（槽）

5. 空闲空间：页中还没被使用的空间

6. 页目录：存储用户记录的相对位置，对纪录起到索引作用。

   1. 把所有纪录划分成组，每个组的最后一条（组内最大）纪录该组有多少条纪录作为n_owned字段

   2. 页目录项存储**每一组最后一条纪录**的**地址偏移量**（称之为槽slot）。

   3. 查找纪录时，可通过二分查找查找槽，定位后再到组里查找纪录

   4. > 1. 第一个分组只能有一条纪录（**即最小记录独立成组**）
      > 2. 最后一个分组只能有1-8条纪录
      > 3. 剩下的分组纪录条数在4-8之间

7. 文件尾：校验页是否完整

### 索引页

在 MySQL 中索引的数据结构和刚刚描述的页几乎是一模一样的，而且大小也是 16K,。

但是在索引页中记录的是页 (数据页，索引页) 的最小主键 id 和页号，以及在索引页中增加了层级的信息，从 0 开始往上算，所以页与页之间就有了上下层级的概念。

当单表数据库到达某个量级的上限时，导致内存无法存储其索引，使得之后的 SQL 查询会产生磁盘 IO，从而导致性能下降，所以增加硬件配置（比如把内存当磁盘使），可能会带来立竿见影的性能提升。

## Count

性能：==`count(*) = count(1) > count(主键字段) > count(字段)`==

### 执行过程

#### count(主键字段)

```sql
//id 为主键值。以此为例
select count(id) from t_order;
```

+ 如果**表里**只有主键索引，**没有二级索引时**，那么，InnoDB 循环**遍历聚簇索引**。读到的纪录返回给server层并读取id值，如果id值不空则count+1
+ 但是，如果**表里有二级索引**时，InnoDB 循环**遍历的对象**就不是聚簇索引，而**是二级索引**。因为二级索引纪录比聚簇索引纪录占用更少的存储空间

  > 主键索引以及聚簇索引会在节点中存储所有的信息，导致一个物理节点中能存的行数比较少。需要很多IO才能读取到所有行。

#### count(1)

count(1)不需要读取纪录中的字段值，**效率高于count(主键字段)**。且如果表里有二级索引，也会使用二级索引

#### count(*) 等价于 count(0)

性能和count(1)没什么差异

#### count(字段)

性能最差，如果不是索引，则会使用ALL类型的查询

#### 建议

+ 如果要执行 count(1)、 count(*)、 count(主键字段) 时，尽量在数据表上建立二级索引（`create index...ON..`)。
+ 不要使用 count(字段) 来统计记录个数，如果必须要用建议建立一个二级索引

### 如何优化Count(*)

假如一张表有1200w数据，即便创建了二级索引执行一次也要花费5秒

#### 近似值

使用`explain`命令：`explain select count(*) from t_order;`

#### 额外的表单独计数

插入删除操作时，对此表操作

# 事务篇

**原子性（Atomicity）**、**一致性（Consistency）**、**隔离性（Isolation）**、**持久性（Durability）**

1. 持久性通过**重做日志redo log**保证
2. 原子性通过**回滚日志undo log**保证
3. 隔离型通过**多版本并发控制MVCC 或者 锁**保证
4. 一致性通过上面三种性质保证

## 事务隔离

+ 并行事务可能出现 **脏读、不可重复度、幻读**。严重程度 脏读 > 不可重复读 > 幻读

#### 脏读

**如果一个事务「读到」了另一个「==未提交==事务修改过的数据」，就意味着发生了「脏读」现象。**（未提交的数据随时可能回滚）

#### 不可重复读

**在一个事务内多次读取同一个数据，如果出现前后两次「==读到的数据不一样==」的情况，就意味着发生了「不可重复读」现象。**

#### 幻读

**在一个事务内多次查询某个符合查询条件的「记录数量」，如果出现前后两次查询到的==记录数量不一样==的情况，就意味着发生了「幻读」现象。**

+ **幻读**是针对查询**结果的数量**不一致，而**不可重复读**针对某行里的某个**数据**不一致的

#### 隔离级别

**串行化 > 可重复读 > 读已提交 > 读未提交**

1. **读未提交**：事务还没提交，它做的变更就能被其他事务看到。**脏读、不可重复读、幻读**都可能发生

2. **读已提交**：事务提交后，它的变更才能被其他事务看到。**不可重复读、幻读**可能发生

3. **可重复读**（InnoDB默认隔离级别）：一个事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的。**幻读**都可能发生。（**但是InnoDB在此级别上却很大程度上避免了幻读现象**）

   1. 针对==**快照**==读，通过==MVCC==方式解决幻读

   2. 针对==**当前**==读，通过next-key lock（==**纪录锁+间隙锁**==）解决（插入导致的幻读不能完美解决，但一定可以**防止删除操作导致的幻读**）。

      当执行 select ... for update 语句的时候，会加上 next-key lock，如果有其他事务在 next-key lock 锁范围内插入了一条记录，那么这个插入语句就会被阻塞，无法成功插入。

4. **串行化**：对纪录加上读写锁，多个事务对这条纪录读写时如果冲突，后访问的事务必须等前个事务执行完成。**都不会发生**

   <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240314%E4%B8%8B%E5%8D%8843156846.png" alt="image-20240314下午43156846" style="zoom:50%;" />

### 隔离级别的实现

读未提交和串行化比较简单

+ 读提交和可重复读，通过**ReadView**（类似于**数据快照**）来实现，它们的区别在于创建ReadView的时机不同。

+ 读已提交：在**每个语句执行前**
+ 可重复读：**启动事务时**（注意，开始事务不意味着启动了事务），Mysql有两种开启事务的命令
  + `begin/start transaction`：执行此命令后事务并没有启动，而是在执行第一条select语句后才是事务真正启动的时机
  + `start transaction with consistent snapshot`：执行此命令后，立马启动事务

#### ReadView在MVCC中如何工作

+ ReadView中有四个字段
+ 聚簇索引纪录中有两个跟事务有关的隐藏列

<img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240314%E4%B8%8B%E5%8D%8883330793.png" alt="image-20240314下午83330793" style="zoom:50%;" />

1. m_ids：创建ReadView时，当前数据库中「**活跃事务的id**」列表。（活跃事务指**启动但还没提交**）

2. min_trx_id：「**活跃事务中id最小的事务**」（m_ids中的最小值）

3. **max_trx_id**：创建 Read View 时当前数据库中**应该给下一个事务的 id 值**。（即全局事务中最大的事务id值+1）

4. creator_trx_id：**创建**该**ReadView**的事务的**事务ID**

   <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240314%E4%B8%8B%E5%8D%8884852797.png" alt="image-20240314下午84852797" style="zoom:50%;" />

5. + 聚簇索引**隐藏列**（纪录的，会随着修改更新）
     + trx_id：当一个事务对某条聚簇索引纪录进行改动，纪录该事务的**事务ID**
     + roll_pointer：对纪录进行改动时，把旧版本的纪录写入undo日志。它指向旧纪录

6. + `trx_id < min_trx_id`：表示创建ReadView前已经提交的事务，所以它对当前事务可见
   + `trx_id >= max_trx_id`: ReadView后才启动，不可见。（即建立ReadView时还没开始的事务）
   + `trx_id在min和max之间`：
     + 如果`trx_id在m_ids列表中`：不可见（已启动还未提交）
     + 如果不在（其实是小于min的那种）：可见（表示事务已被提交）

### 可重复读如何工作以及如何解决幻读

+ **启动事务时新建ReadView**

事务B第二次去读取纪录，发现这条记录的 trx_id 值为 51，在事务 B 的 Read View 的 min_trx_id 和 max_trx_id 之间，则需要判断 trx_id 值是否在 m_ids 范围内，判断的结果是在的，那么说明这条记录是被还未提交的事务修改的，这时事务 B 并不会读取这个版本的记录。而是沿着 undo log 链条往下找旧版本的记录，直到找到 trx_id 「小于」事务 B 的 Read View 中的 min_trx_id 值的第一条记录。

即便事务A提交事务后（提交并未修改trx_id还是51），由于隔离级别时「可重复读」，所以事务 B 再次读取记录时，还是**基于启动事务时创建的 Read View 来判断当前版本的记录是否可见**。所以，即使事物 A 将小林余额修改为 200 万并提交了事务， 事务 B 第三次读取记录时，读到的记录都是小林余额是 100 万的这条记录。

#### 读已提交如何工作

+ **在执行语句（读数据）前新建ReadView**

主要不同在于，事务A提交后。B重新去读会新建ReadView从而min变得更大（使得trx_id<min)。从而和上面的过程不一样了。

#### 如何解决幻读

1. 快照（ReadView）：但只有普通查询是快照读

   在执行第一个查询语句后，会创建一个 Read View，后续的查询语句利用这个 Read View，通过这个 Read View 就可以在 undo log 版本链找到事务开始时的数据，所以事务过程中每次查询的数据都是一样的，即使中途有其他事务插入了新纪录，是查询不出来这条数据的。从而避免了幻读

2. 当前读：

   **MySQL 里<u>除了普通查询是快照读，其他都是当前读</u>，比如 update、insert、delete，这些<u>语句执行前都会查询最新版本的数据</u>，然后再做进一步的操作。**

​	有了快照读，并不能完美解决幻读问题，比如下面的例子

<img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240314%E4%B8%8B%E5%8D%88101250554.png" alt="image-20240314下午101250554" style="zoom:50%;" />

在可重复读隔离级别下，事务 A 第一次执行普通的 select 语句时生成了一个 ReadView，之后事务 B 向表中新插入了一条 id = 5 的记录并提交（事务 A 的 ReadView 并不包含这条新插入的记录）。接着，事务 A 对 id = 5 这条记录进行了更新操作，在这个时刻，这条新记录的 trx_id 隐藏列的值就变成了事务 A 的事务 id，之后事务 A 再使用普通 select 语句去查询这条记录时就可以看到这条记录了，于是就发生了幻读（之前4之后5）。原本新加入的纪录的trx_id是B的（事务B在后面发生，不在A的m_ids里，判定不可读），但是现在成A的了，就直接查了。

> 1. 第三步：事务 A 对 id = 5 这条记录进行了更新操作，并将该记录的 trx_id 隐藏列的值设置为事务 A 的事务 id。
> 2. 第四步：现在如果事务 A 再次执行 SELECT 查询 id = 5 这条记录，根据 MVCC 的机制，它将会看到更新后的记录。虽然在事务 A 启动时该记录并不存在，但是由于在事务 A 第一次生成 ReadView 时，该记录也不存在，所以 ReadView 并没有包含事务 B 插入的记录，因此事务 A 能够看到由自己更新的记录，从而导致了幻读的发生。

又比如：

+ T1 时刻：事务 A 先执行「快照读语句」：select * from t_test where id > 100 得到了 3 条记录。

+ T2 时刻：事务 B 往插入一个 id= 200 的记录并提交；

+ T3 时刻：事务 A 再执行「当前读语句」 select * from t_test where id > 100 **for update** 就会得到 4 条记录，此时也发生了幻读现象。

  > 在第二个场景中，事务 A 第一次执行的是快照读语句，然后在第二次查询时执行了当前读语句并且使用了锁（`FOR UPDATE`，**导致变成了当前读而不再是快照**读）。当前读语句使用了锁，使得事务 A 在执行第二次查询时可以看到事务 B 提交的新插入记录

#### 如何避免

**尽量在==开启事务之后，马上执行 select ... for update 这类当前读的语句==**。因为它会对记录加 next-key lock，从而避免其他事务插入一条新记录。

# 锁篇

我们可以通过 `select * from performance_schema.data_locks\G;` 这条语句，查看事务执行 SQL 过程中加了什么锁。

## 有哪些锁

### **==全局锁==**：主要用于**全库的==逻辑备份==**

`flush tables with read lock` ：整个数据库处于只读状态，`unlocak tables`解锁

+ 问题显而易见，影响业务的正常运行。如果数据库的引擎支持的事务支持**可重复读的隔离级别**，则可以解决此问题
  + 备份数据库的工具是 mysqldump，在使用 mysqldump 时加上 `–single-transaction` 参数的时候，就会在备份数据库之前先开启事务。

### **==表级锁==**：

1. **==表锁==**

   ```sql
   # 表级别的共享锁，也就是读锁；
   lock tables t_student read;
   # 表级别的独占锁，也就是写锁；
   lock tables t_stuent write;
   # 释放锁
   unlock tables
   
   #需要注意的是，表锁除了会限制别的线程的读写外，也会限制本线程接下来的读写操作。
   ```

   尽量避免使用表锁，因为表锁的颗粒度太大。**InnoDB 牛逼的地方在于实现了颗粒度更细的行级锁**

2. **==元数据锁MDL==（Metadata Locks）**：

   不需要显示的使用 MDL，因为当我们对数据库表进行操作时，会**自动**给这个表加上 MDL。

   + 对一张表进行 **CRUD 操作**时，加的是 **MDL 读锁**；
   + 对一张表做**结构变更**操作的时候，加的是 **MDL 写锁**；

   > 当有线程在执行 select 语句（ 加 MDL 读锁）的期间，如果有其他线程要更改该表的结构（ 申请 MDL 写锁），那么将会被阻塞，直到执行完 select 语句（ 释放 MDL 读锁）。
   >
   > 反之，当有线程对表结构进行变更（ 加 MDL 写锁）的期间，如果有其他线程执行了 CRUD 操作（ 申请 MDL 读锁），那么就会被阻塞，直到表结构变更完成（ 释放 MDL 写锁）。

   + MDL 是在事务提交后才会释放，这意味着**事务执行期间，MDL 是一直持有的**。
   + 申请 MDL 锁的操作会形成一个队列，队列中**写锁获取优先级高于读锁**，一旦出现 MDL **写锁等待，会阻塞后续该表的所有 CRUD 操作**。

3. **==意向锁==**：

   1. **意向锁的目的是为了==快速判断表里是否有记录被加锁==**。
   2. 在使用 InnoDB 引擎的表里对某些记录加上「共享锁」（读锁）之前，需要先在**表级别加上一个「意向共享锁」**；在使用 InnoDB 引擎的表里对某些纪录加上「独占锁」（写锁）之前，需要先在表级别加上一个「意向独占锁」；

   > 普通的 select 是不会加行级锁的，普通的 select 语句是利用 MVCC 实现一致性读，是无锁的，但是也可以加。
   >
   > ```sql
   > # 先在表上加上意向共享锁，然后对读取的记录加共享锁
   > select ... lock in share mode;
   > 
   > # 先表上加上意向独占锁，然后对读取的记录加独占锁
   > select ... for update;
   > ```
   >
   > 意向共享锁和意向独占锁是表级锁，不会和行级的共享锁和独占锁发生冲突，而且**意向锁之间也不会发生冲突**，**只会和**共享**表锁**（lock tables ... read）和独占**表锁**（\*lock tables ... write\*）**发生冲突**。

4. **==AUTO-INC锁==**：

   常用于实现主键自增，AUTO-INC 锁是特殊的表锁机制，锁**不是再一个事务提交后才释放，而是在执行完插入语句后就会立即释放**。**在插入数据时，会加一个表级别的 AUTO-INC 锁**。

   MySQL 5.1.22 版本开始，InnoDB 存储引擎提供了一种**轻量级的锁**来实现自增（可配置）。一样也是在插入数据的时候，会为**被 `AUTO_INCREMENT` 修饰的「字段」加上轻量级锁**，然后给该字段赋值一个自增的值，就把这个轻量级锁释放了，而不需要等待整个插入语句执行完后才释放锁。

   + 当 innodb_autoinc_lock_mode = **0**，就采用 **AUTO-INC** 锁，语句**执行结束后才释放**锁；
   + 当 innodb_autoinc_lock_mode = **2**，就采用**轻量级锁**，**申请自增主键后就释放锁**，并不需要等语句执行后才释放。
   + 当 innodb_autoinc_lock_mode = **1**：
     + **普通 insert 语句，自增锁在申请之后就马上释放**；
     + 类似 insert … select 这样的批量插入数据的语句，自增锁还是要等语句结束后才被释放；

   > 当 innodb_autoinc_lock_mode = **2** 是**性能最高**的方式，但是当**搭配 binlog 的日志格式是 statement** 一起使用的时候，在「主从复制的场景」中**会发生数据不一致的问题**。（**binlog日志格式设置为row不会**）

### 行级锁

InnoDB 引擎是支持行级锁的，而 MyISAM 引擎并不支持行级锁。

**普通的 select 语句是不会对记录加锁的，因为它属于快照读**。如果要在查询时对记录加行锁，可以使用下面这两个方式，这种查询会加锁的语句称为**锁定读**。

```sql
# 对读取的记录加共享锁
select ... lock in share mode;
# 对读取的记录加独占锁
select ... for update;
# 上面这两条语句必须在一个事务中，因为当事务提交了，锁就会被释放，所以在使用这两条语句的时候，要加上 begin/start transaction 或者 set autocommit = 0。
```

1. **RecordLock**（==纪录锁==）：有S锁和X锁之分（类似读锁和写锁）

2. **GapLock**（==间隙锁==，只存在于可重复读隔离级别）：间隙锁止键互相兼容,`(3,5)`

   假如数据库中只有id=1、5、10...（即没有234），那么想要锁2，间隙锁的范围应该是(1,5)。而不是(1,3)。

3. **==Next-Key==**（前两者一起使用）：锁定一个范围，并且锁定记录本身。类似`(3,5]`

#### 唯一索引等值查询

当我们用**唯一索引进行等值查询**的时候，查询的记录存不存在，加锁的规则也会不同：

+ 当查询的记录是「存在」的，在索引树上定位到这一条记录后，将该记录的索引中的 next-key lock 会**退化成「记录锁」**。
+ 当查询的记录是「不存在」的，在索引树找到第一条大于该查询记录的记录后，将该记录的索引中的 next-key lock 会**退化成「间隙锁」**。（因为仅靠间隙锁就可以避免幻读）

#### 唯一索引范围查询

> 总之，间隙锁的区间是**左开右闭**或者**都是开区间**的（**最多右边闭上**），如果能覆盖则不需要另外的记录锁。

当唯一索引进行范围查询时，**会对每一个扫描到的索引加 next-key 锁**

> ```sql
> mysql> select * from user where id > 15 for update;
> ```
>
> <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8825414832.png" alt="image-20240315下午25414832" style="zoom:50%;" />

如果遇到下面这些情况，会退化成记录锁或者间隙锁

+ 情况一：针对「**大于等于**」的范围查询，因为存在等值查询的条件，那么如果**等值**查询的记录是**存在**于表中，那么该记录的索引中的 next-key 锁会**退化成记录锁**。

  > ```sql
  > mysql> select * from user where id >= 15 for update;
  > ```
  >
  > <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8825343260.png" alt="image-20240315下午25343260" style="zoom:50%;" />

+ 情况二：针对「**小于或者小于等于**」的范围查询，要看条件值的记录是否存在于表中：

  + 当条件值的记录不在表中，那么**不管是「小于」还是「小于等于**」条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁。

    ```sql
    mysql> select * from user where id < 6 for update;
    ```

    <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8825529710.png" alt="image-20240315下午25529710" style="zoom:50%;" />

  + 当条件值的记录在表中，如果是「小于」条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁；如果「小于等于」条件的范围查询，扫描到终止范围查询的记录时，该记录的索引 next-key 锁不会退化成间隙锁。其他扫描到的记录，都是在这些记录的索引上加 next-key 锁。

    ```sql
    mysql> select * from user where id <= 5 for update;
    ```

    <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8825605280.png" alt="image-20240315下午25605280" style="zoom:50%;" />

    ```sql
    select * from user where id < 5 for update;
    ```

    <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8825723996.png" alt="image-20240315下午25723996" style="zoom:50%;" />

#### 非唯一索引等值查询

当我们用非唯一索引进行等值查询的时候，因为存在两个索引，一个是主键索引，一个是非唯一索引（二级索引），所以在加锁时，同时**会对这两个索引都加锁**，但是**对主键索引加锁的时候，只有满足查询条件的记录才会对它们的主键索引加锁**。

+ 纪录不存在

```sql
select * from user where age = 25 for update;
```

![image-20240315下午33739614](./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8833739614.png)

+ 纪录存在

  ```sql
  select * from user where age = 22 for update;
  ```

  <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8834612671.png" alt="image-20240315下午34612671" style="zoom:50%;" />

#### 非唯一索引范围查询

```sql
select * from user where age >= 22  for update;
```

<img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8835909754.png" alt="image-20240315下午35909754" style="zoom:50%;" />

等值查询，但非唯一索引不会像唯一索引那样，二级索引上的 next-key 锁退化为记录锁。

#### 没有加索引的查询

如果锁定读<u>查询语句，没有使用索引列作为查询条件，或者查询语句没有走索引查询，或者update和delete语句查询条件不加索引</u>，导致扫描是全表扫描。那么，**每一条记录的索引上都会加 next-key 锁**，这样就相当于锁住的全表，这时如果其他事务对该表进行增、删、改操作的时候，都会被阻塞。

---

**在线上在执行 ==update、delete、select ... for update 等具有加锁性质的语句==，一定要检查语句是否走了索引，如果是==全表扫描==的话，会对每一个索引加 next-key 锁，相当于==把整个表锁住==了**

#### 如何避免

**将 MySQL 里的 `sql_safe_updates` 参数设置为 1，开启安全更新模式。**此时**update**需要满足如下条件**之一**才能执行

+ 使用 where，并且 where 条件中必须有索引列；
+ 使用 limit；
+ 同时使用 where 和 limit，此时 where 条件中可以没有索引列；

delete满足**同时使用 where 和 limit，此时 where 条件中可以没有索引列**才能执行。如果 where 条件带上了索引列，但是优化器最终扫描选择的是全表，而不是索引的话，我们可以使用 **`force index([index_name])`** 可以**告诉优化器使用哪个索引**，以此避免有几率锁全表带来的隐患。

---

### 插入意向锁

+ 注意它不是意向锁，而是一种特殊的间隙锁，属于行级锁。

一个事务在插入一条记录的时候，需要判断**插入位置是否已被其他事务加了间隙锁**（next-key lock 也包含间隙锁）。如果**有的话**，插入操作就会发生**阻塞**，直到拥有间隙锁的那个事务提交为止（释放间隙锁的时刻），在此期间会生成一个==**插入意向锁**，**表明有事务想在某个区间插入新记录，但是现在处于等待状态**==（等待不意味着获得了锁）。

> **MySQL 加锁时，是先生成锁结构，然后设置锁的状态，如果锁状态是等待状态，并不是意味着事务成功获取到了锁，只有当锁状态为正常状态时，才代表事务成功获取到了锁**

## Mysql怎么加锁（总结）

### 行级锁

#### 什么SQL会加行级锁

1. ==普通select不会对纪录加锁==（除了串行化隔离级别），因为它属于快照读，通过MVCC实现

   + 如果要加行级锁

     ```sql
     # 对读取的记录加共享锁(S型锁)
     select ... lock in share mode;
     # 对读取的记录加独占锁(X型锁)
     select ... for update;
     ```

   上面这两条语句必须在事务中，**因为当事务提交了，锁就会被释放**，所以在使用这两条语句的时候，**要加上 begin 或者 start transaction 开启事务的语句**。

2. ==update 、delete 都会加行级锁，且都是独占锁==（X型锁）

   ```sql
   # 对操作的记录加独占锁(X型锁)
   update table .... where id = 1;
   # 对操作的记录加独占锁(X型锁)
   delete from table where id = 1;
   ```

#### 行级锁有哪些种类

1. **已提交**隔离级别下：仅有纪录锁
2. **可重复读**隔离级别下：
   1. 纪录锁：把一条纪录锁上。有X/S锁之分
   2. 间隙锁：锁定一个范围（是开区间），为了解决幻读。有X/S锁之分但无所谓
   3. Next-Key Lock：锁定范围但前开后闭（也可全开）。
      + 如果 LOCK_MODE 为 `X`，说明是 next-key 锁；
      + 如果 LOCK_MODE 为 `X, REC_NOT_GAP`，说明是记录锁；
      + 如果 LOCK_MODE 为 `X, GAP`，说明是间隙锁；

+ **加锁的对象是索引，加锁的基本单位是 next-key lock**。

+ 唯一索引加锁流程图

  ![唯一索引加锁流程](./%E5%9B%BE%E8%A7%A3mysql.assets/%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E5%8A%A0%E9%94%81%E6%B5%81%E7%A8%8B.jpeg)

+ 非唯一索引流程图

  ![非唯一索引加锁流程](./%E5%9B%BE%E8%A7%A3mysql.assets/%E9%9D%9E%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95%E5%8A%A0%E9%94%81%E6%B5%81%E7%A8%8B.jpeg)

## Mysql死锁

![image-20240315下午83148933](./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8883148933.png)

+ **插入意向锁与间隙锁是冲突的**

案例中的事务 A 和事务 B 在执行完后 `select ... for update` 语句后**都持有**范围为`(1006,+∞]`（因为最后一个是1006）的**next-key 锁**，而接下来的插入操作为了获取到插入意向锁，**都在等待对方事务的间隙锁释放**，于是就造成了循环等待，导致死锁。

**间隙锁的意义只在于阻止区间被插入**，因此是可以共存的。**一个事务获取的间隙锁不会阻止另一个事务获取同一个间隙范围的间隙锁**，共享和排他的间隙锁是没有区别的，他们相互不冲突，且功能相同，即两个事务可以同时持有包含共同间隙的间隙锁。

但是**next-key lock 是包含间隙锁+记录锁的，如果一个事务获取了 X 型的 next-key lock，那么另外一个事务在获取相同范围的 X 型的 next-key lock 时，是会被阻塞的**。但是还要注意！对于这种范围为 **(1006, +∞] 的 next-key lock**，**两个事务是可以同时持有的**，不会冲突。**因为 +∞ 并不是一个真实的记录**，自然就不需要考虑 X 型与 S 型关系。

### 唯一键冲突

如果在插入新记录时，插入了一个与「已有的记录的**主键或者唯一二级索引列值相同**」的记录，此时插入就会失败，然后对于这条记录加上了 **S 型的锁**。

+ 如果主键索引重复，插入新记录的事务会给已存在的主键值重复的聚簇索引记录**添加 S 型记录锁**。
+ 如果**唯一二级索引重复**，插入新记录的事务都会给已存在的二级索引列值重复的二级索引记录**添加 S 型 next-key 锁**。

![image-20240315下午92416337](./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240315%E4%B8%8B%E5%8D%8892416337.png)

+ 事务 A 先插入 order_no 为 1006 的记录，可以插入成功，此时对应的唯一二级索引记录被「隐式锁」保护，此时还没有实际的锁结构（执行完这里的时候，你可以看查 performance_schema.data_locks 信息，可以看到这条记录是没有加任何锁的）；

+ 接着，事务 B 也插入 order_no 为 1006 的记录，由于事务 A 已经插入 order_no 值为 1006 的记录，所以事务 B 在插入二级索引记录时会遇到重复的唯一二级索引列值，此时事务 B 想获取一个 S 型 next-key 锁，但是事务 A 并未提交，事务 A 插入的 order_no 值为 1006 的记录上的**「隐式锁」会变「显示锁」且锁类型为 X 型的记录锁**（注意，这个是在执行事务 B 之后才产生的锁，没执行事务 B 之前，该记录还是隐式锁），所以事务 B 向获取 S 型 next-key 锁时会遇到锁冲突，事务 B 进入阻塞状态。

  > 综上，第一个事务插入的记录，并不会加锁，而是会用隐式锁保护唯一二级索引的记录。但是当第一个事务还未提交的时候，有其他事务插入了与第一个事务相同的记录，第二个事务就会**被阻塞**，**因为此时第一事务插入的记录中的隐式锁会变为显示锁且类型是 X 型的记录锁，而第二个事务是想对该记录加上 S 型的 next-key 锁，X 型与 S 型的锁是冲突的**，所以导致第二个事务会等待，直到第一个事务提交后，释放了锁。

### 如何避免死锁

破坏循环等待条件：

+ 设置**事务等待锁的==超时时间==**：超过该值后这个事务进行回滚（锁被释放，另一个事务被执行）
+ **主动开启==死锁检测==**：监测到死锁，主动回滚死锁链条中的第一个事务。

# 日志篇

更新语句会涉及到三种日志

+ **undo log（回滚日志）**：是 Innodb 存储引擎层生成的日志，实现了事务中的**原子性**，主要**用于事务回滚和 MVCC**。
+ **redo log（重做日志）**：是 Innodb 存储引擎层生成的日志，实现了事务中的**持久性**，主要**用于掉电等故障恢复**；
+ **binlog （归档日志）**：是 Server 层生成的日志，主要**用于数据备份和主从复制**；

### undoLog

我们在执行执行一条“增删改”语句的时候， MySQL 会**隐式开启事务**来执行“增删改”语句的，执行完就自动提交事务。执行一条语句是否自动提交事务，是由 `autocommit` 参数决定的，默认是开启。所以，执行一条 update 语句也是会使用事务的。考虑一个问题。一个事务在执行过程中，在还没有提交事务之前，如果 MySQL 发生了崩溃，要怎么回滚到事务之前的数据呢？

每当 InnoDB 引擎对一条记录进行操作（修改、删除、新增）时，要把回滚时需要的信息都记录到 undo log 里，比如：

+ 在**插入**一条记录时，要把这条记录的主键值记下来，这样之后回滚时只需要把这个主键值对应的记录**删掉**就好了；
+ 在**删除**一条记录时，要把这条记录中的内容都记下来，这样之后回滚时再把由这些内容组成的记录**插入**到表中就好了；
+ 在**更新**一条记录时，要把被更新的列的旧值记下来，这样之后回滚时再把这些列**更新为旧值**就好了。

一条记录的每一次更新操作产生的 **undo log 格式都有一个 roll_pointer 指针和一个 trx_id 事务id**：

+ 通过 trx_id 可以知道该记录是被哪个事务修改的；
+ 通过 roll_pointer 指针可以将这些 undo log 串成一个链表，这个链表就被称为**版本链**；

#### 两大作用

+ **实现事务回滚，保障事务的原子性**

+ **undo log 还有一个作用，通过 ReadView + undo log 实现 MVCC（多版本并发控制）**。undo log 为每条记录保存多份历史数据，MySQL 在执行快照读（普通 select 语句）的时候，会根据事务的 Read View 里的信息，顺着 undo log 的版本链找到满足其可见性的记录

> undolog也需要redolog来保证持久化

### BufferPool

修改过的缓存叫脏页。缓存一开始是空闲的，随着程序运行才有磁盘上的页被缓存到BufferPool。且查询一条纪录也是调一页

**BufferPool存储「索引页」「数据页」「Undo页」「插入缓存」「自适应哈希索引」「锁信息」等**

### Redolog：

+ 时刻记住它是==更新后的数据==，为了崩溃恢复设计。log

当有一条记录需要更新的时候，InnoDB 引擎就会先更新内存（同时标记为脏页），然后将本次对这个页的修改以 **redo log** 的形式记录下来，**这个时候更新就算完成了**。**WAL 技术指的是， MySQL 的写操作并不是立刻写到磁盘上，而是先写日志，然后在合适的时间再写到磁盘上**。且由于redolog是顺序写，而写数据是随机写，使得**MySQL 的写操作从磁盘的「随机写」变成了「顺序写」**，提升语句的执行性能（日志再找合适时间把数据更新到磁盘）

![image-20240316下午24059181](./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240316%E4%B8%8B%E5%8D%8824059181.png)

**事务提交时，不需要将BufferPool里的数据持久化到磁盘，只需要将redolog持久化到磁盘即可**

undo和redo：事务提交之前发生了崩溃，重启后会通过 undo log 回滚事务，事务提交之后发生了崩溃，重启后会通过 redo log 恢复事务。

#### redolog的缓存

 redo log 也不是直接写入磁盘，redo log 也有自己的缓存—— **redo log buffer**。刷盘时机：

+ MySQL 正常**关闭**时；

+ 当 redo log buffer 中记录的==**写入量大于** redo log buffer **内存空间的一半时**==，会触发落盘；

+ InnoDB 的后台线程==**每隔 1 秒**==，将 redo log buffer 持久化到磁盘。

+ **==每次事务提交时==**都将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘（这个策略可由 `innodb_flush_log_at_trx_commit` 参数控制，下面会说）。

  <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240316%E4%B8%8B%E5%8D%8825702063.png" alt="image-20240316下午25702063" style="zoom:50%;" />

  ​		数据安全性：参数 1 > 参数 2 > 参数 0

  ​		写入性能：参数 0 > 参数 2> 参数 1

  + 当设置该**参数为 0 时**，表示每次事务提交时 ，还是将 redo log 留在 redo log buffer 中 ，该模式下在**事务提交时不会主动触发写入磁盘的操作**。（mysql崩溃就丢失1s数据）
  + 当设置该**参数为 1 时**（默认），表示每次**事务提交时**，都将缓存在 redo log buffer 里的 redo log 直接**持久化到磁盘**，这样可以保证 MySQL 异常重启之后数据不会丢失。
  + 当设置该**参数为 2 时**，表示每次**事务提交时**，都只是缓存在 redo log buffer 里的 redo log **写到 redo log 文件**，<u>注意写入到「 redo log 文件」并不意味着写入到了磁盘</u>，因为操作系统的文件系统中有个 Page Cache。Page Cache 是专门用来缓存文件数据的，<u>所以写入「 redo log文件」意味着写入到了操作系统的文件缓存</u>。(即mysql崩溃不一定丢失数据，系统也崩溃才会)

#### redolog写满

默认情况下， InnoDB 存储引擎有 1 个重做日志文件组( redo log Group），「重做日志文件组」由有 2 个 redo log 文件组成，这两个 redo 日志的文件名叫 ：`ib_logfile0` 和 `ib_logfile1`。假设每个 redo log File 设置的上限是 1 GB，那么总共就可以记录 2GB 的操作。

重做日志文件组是以==**循环写**==的方式工作的，从头开始写，写到末尾就又回到开头，相当于一个环形。**redo log 文件满了，这时 MySQL 不能再执行新的更新操作，也就是说 MySQL 会被阻塞**（*因此所以针对并发量大的系统，适当设置 redo log 的文件大小非常重要*），redo log 是为了防止 Buffer Pool 中的脏页丢失而设计的，此时**会停下来将 Buffer Pool 中的脏页==刷新到磁盘==中，然后标记 redo log 哪些记录可以被擦除，接着对旧的 redo log 记录进行擦除，等擦除完旧记录腾出了空间，checkpoint 就会往后移动（图中顺时针）**

### binlog

MySQL 在完成一条更新操作后，**Server 层**还会生成一条 binlog，等之后事务提交的时候，会将该事物执行过程中产生的所有 binlog 统一写 入 binlog 文件。binlog 文件是**记录**了所有数据库==**表结构变更和表数据修改**==的日志，==**不会记录查询类的操作**==，比如 SELECT 和 SHOW 操作。

#### 有了redolog为什么还有binlog

跟一开始的设计有关。最开始 MySQL 里并没有 InnoDB 引擎，MySQL 自带的引擎是 MyISAM，但是 MyISAM 没有 crash-safe 的能力，binlog 日志只能用于归档。而 InnoDB 是另一个公司以插件形式引入 MySQL 的，既然只依靠 binlog 是没有 crash-safe 能力的，所以 InnoDB 使用 redo log 来实现 crash-safe 能力。

> + binlog 是 MySQL 的 **Server 层实现**的日志，所有存储引擎都可以使用；redo log 是 Innodb **存储引擎实现**的日志；
> + binlog 是追加写，写满一个文件，就创建一个新的文件继续写，**不会覆盖以前的日志**，保存的是全量的日志。redo log 是**循环写**，日志空间大小是固定，全部写满就从头开始，保存未被刷入磁盘的脏页日志。
> + **==binlog 用于备份恢复、主从复制；redo log 用于掉电等故障恢复==。**(比如数据库被删，redolog文件无法恢复，而binlog可以)
> + ==binlog 有 3 种格式类型==，分别是 STATEMENT（默认格式）、ROW、 MIXED。redo log 是物理日志（比如对XX表中YY数据页ZZ偏移量进行了AA修改）
>   + ==STATEMENT==：**每一条修改数据的 SQL 都会被记录到 binlog 中**（相当于记录了逻辑操作，所以针对这种格式， binlog 可以称为逻辑日志），主从复制中 slave 端再根据 SQL 语句重现。但 STATEMENT 有动态函数的问题，比如你用了 uuid 或者 now 这些函数，你在主库上执行的结果并不是你在从库执行的结果，这种随时在变的函数会导致复制的数据不一致；
>   + ==ROW==：**记录行数据最终被修改成什么样**了（这种格式的日志，就不能称为逻辑日志了），不会出现 STATEMENT 下动态函数的问题。但 ROW 的缺点是每行数据的变化结果都会被记录，比如执行批量 update 语句，更新多少行数据就会产生多少条记录，使 binlog 文件过大，而在 STATEMENT 格式下只会记录一个 update 语句而已；
>   + ==MIXED==：包含了 STATEMENT 和 ROW 模式，它会根据不同的情况**自动使用 ROW 模式和 STATEMENT 模式**；

### binlog主从复制

分为三个阶段：写入binlgo、同步binlog、回放binlog

**在完成主从复制之后，就可以在写数据时只写主库，在读数据时只读从库，这样即使写请求会锁表或者锁记录，也不会影响读请求的执行。**从库不是越多越好，一般一个主库跟2-3个从库。

+ 同步复制：MySQL 主库提交事务的线程要**等待所有从库**的复制成功响应，才返回客户端结果。这种方式在实际项目中，基本上没法用
+ 异步复制（默认）：不会等待binlog同步到各从库
+ 半同步复制：只要一部分复制成功响应回来，主库的事务线程就可以返回给客户端。即使出现主库宕机，至少还有一个从库有最新的数据，不存在数据丢失的风险

### binlog何时刷盘

1. 事务执行过程中，先把日志写到 binlog cache（Server 层的 cache），**事务提交时，再把 binlog cache 写到 binlog 文件中**。一个事务的 binlog 是不能被拆开的，因此无论这个事务有多大（比如有很多条语句），也要保证一次性写入。
2. 缓冲内存叫 binlog cache，参数 binlog_cache_size 用于控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。

+ 虽然每个线程有自己 binlog cache，但是最终都写到同一个 binlog 文件：

  MySQL提供一个 `sync_binlog` 参数来控制数据库的 binlog 刷到磁盘上的频率：

  + sync_binlog = **0**（默认） 的时候，表示每次提交事务都**只 write，不 fsync**，后续交由操作系统决定何时将数据持久化到磁盘；
  + sync_binlog = **1** 的时候，表示**每次提交事务都会 write，然后马上执行 fsync**；
  + sync_binlog =N(N>1) 的时候，表示**每次提交事务都 write，但累积 N 个事务后才 fsync**。

<img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240316%E4%B8%8B%E5%8D%8840701085.png" alt="image-20240316下午40701085" style="zoom:50%;" />

### update日志过程

具体更新一条记录 `UPDATE t_user SET name = 'xiaolin' WHERE id = 1;` 的流程如下:

1. 执行器负责具体执行，会调用存储引擎的接口，通过主键索引树搜索获取 id = 1 这一行记录：
   + 如果 id=1 这一行所在的数据页本来就在 buffer pool 中，就直接返回给执行器更新；
   + 如果记录不在 buffer pool，将数据页从磁盘读入到 buffer pool，返回记录给执行器。
2. 执行器得到聚簇索引记录后，会看一下更新前的记录和更新后的记录是否一样：
   + 如果一样的话就不进行后续更新流程；
   + 如果不一样的话就把更新前的记录和更新后的记录都当作参数传给 InnoDB 层，让 InnoDB 真正的执行更新记录的操作；
3. 开启事务， InnoDB 层**更新记录前，首先要记录相应的 undo log**，因为这是更新操作，需要把被更新的列的旧值记下来，也就是要生成一条 undo log，undo log 会**写入 Buffer Pool 中的 Undo 页面**，不过在内存**修改该 Undo 页面后，需要记录对应的 redo log**。
4. InnoDB 层开始**更新记录**，会先更新内存（同时标记为脏页），然后**将记录写到 redo log 里（redo log buffer）**，这个时候更新就算完成了。为了减少磁盘I/O，不会立即将脏页写入磁盘，后续由后台线程选择一个合适的时机将脏页写入到磁盘。这就是 **WAL 技术**，MySQL 的写操作并不是立刻写到磁盘上，而是先写 redo 日志，然后在合适的时间再将修改的行数据写到磁盘上。
5. 至此，一条记录更新完了。
6. 在一条**更新语句执行完成后**，然后开始记录该语句对应的 **binlog**，此时记录的 binlog 会被保存到 binlog cache，并没有刷新到硬盘上的 binlog 文件，在事务提交时才会统一将该事务运行过程中的所有 binlog 刷新到硬盘。
7. 事务提交，剩下的就是「两阶段提交」的事情了

#### 两段提交

它可以保证多个逻辑操作要不全部成功，要不全部失败，不会出现半成功的状态。**两阶段提交把单个事务的提交拆分成了 2 个阶段，分别是「准备（Prepare）阶段」和「提交（Commit）阶段**。

+ **prepare 阶段**：将 XID（**内部 XA 事务的 ID**） 写入到 redo log，同时将 **redo log** 对应的事务状态设置为 ==**prepare**==，然后将 redo log ==**持久化**==到磁盘（innodb_flush_log_at_trx_commit = 1 的作用）；
+ **commit 阶段**：把 XID 写入到 binlog，然后将 ==**binlog 持久化**==到磁盘（sync_binlog = 1 的作用），接着调用引擎的提交事务接口，将 ==**redo log 状态设置为 commit**==，<u>此时该状态并不需要持久化到磁盘，只需要 write 到文件系统的 page cache 中就够了</u>，因为只要 binlog 写磁盘成功，就算 redo log 的状态还是 prepare 也没有关系，一样会被认为事务已经执行成功；

>  需要两段提交的问题来源：事务提交后，redo log 和 binlog 都要持久化到磁盘，但是这两个是独立的逻辑，可能出现半成功的状态，两份日志之间的逻辑可能不一致
>
> + **redo log 刷入到磁盘之后， MySQL 突然宕机了，而 binlog 还没有来得及写入**MySQL 重启后，通过 **redo log 能**将 Buffer Pool 中 id = 1 这行数据的 name 字段**恢复**到新值 xiaolin，**但是 binlog 里面没有记录**这条更新语句，在主从架构中，<u>binlog 会被复制到从库，由于 binlog 丢失了这条更新语句</u>，从库的这一行 name 字段是旧值 jay，主从库的值不一致性；
> + redo log 刷入到磁盘之后， MySQL 突然宕机了，而 binlog 还没有来得及写入**（类似上述）

两段提交如何保证的？不论何时崩溃，redolog都处于prepare状态。在 MySQL 重启后会按顺序扫描 redo log 文件，碰到处于 prepare 状态的 redo log，就拿着 redo log 中的 XID 去 binlog 查看是否存在此 XID：

+ **如果 binlog 中没有当前内部 XA 事务的 XID，说明 redolog 完成刷盘，但是 binlog 还没有刷盘，则回滚事务**。
+ **如果 binlog 中有当前内部 XA 事务的 XID，说明 redolog 和 binlog 都已经完成了刷盘，则提交事务**。

#### 两段提交的问题

虽然能保证两个日志文件的数据一致性，但性能很差。

1. I/O次数高：每个事务都会进行两次刷盘（fsync）。

   > + 当 sync_binlog = 1 的时候，表示每次提交事务都会将 binlog cache 里的 binlog 直接持久到磁盘；
   > + 当 innodb_flush_log_at_trx_commit = 1 时，表示每次事务提交时，都将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘；

2. 锁竞争激烈：两阶段提交虽然能够保证「单事务」两个日志的内容一致，但在「多事务」的情况下，却不能保证两者的提交顺序一致。因此，还需要加一个锁来保证提交的原子性，从而保证多事务的情况下，两个日志的提交顺序一致。

#### binlog组提交

**MySQL 引入了 binlog 组提交（group commit）机制，当有多个事务提交的时候，会将多个 binlog 刷盘操作合并成一个，从而减少磁盘 I/O 的次数**（5.7后引入redolog组），引入后，prepare不变，commit拆分为三个过程（三个阶段都有一个队列，锁就只针对每个队列进行保护，不再锁住提交事务的整个过程，颗粒度减小提升效率）

+ **flush 阶段**：多个事务按进入的顺序将 binlog 从 cache 写入文件（不刷盘）；
+ **sync 阶段**：对 binlog 文件做 fsync 操作（多个事务的 binlog 合并一次刷盘）；
+ **commit 阶段**：各个事务按顺序做 InnoDB commit 操作；

#### redolog组提交

在 prepare 阶段不再让事务各自执行 redo log 刷盘操作，而是推迟到组提交的 flush 阶段，也就是说 prepare 阶段融合在了 flush 阶段。（prepare和commit合并成了一段flush、sync、commit）

1. flush阶段：**用于支撑 redo log 的组提交**

   1. 第一个事务会成为 flush 阶段的 Leader，此时后面到来的事务都是 Follower 
   2. 组的 Leader 对 redo log 做一次 write + fsync，即**一次将同组事务**的 redolog 刷盘：
   3. 将绿色这一组事务执行过程中产生的 binlog 写入 binlog 文件（write，不调用fnyc不刷盘，缓存在OS文件系统）

   在此之前崩溃，则会回滚

2. sync阶段：**用于支持 binlog 的组提交**

   绿色这一组事务的 binlog 写入到 binlog 文件后，并不会马上执行刷盘的操作，而是**会等待一段时间**，这个等待的时长由 `Binlog_group_commit_sync_delay` 参数控制，**目的是为了组合更多事务的 binlog，然后再一起刷盘**

3. commit阶段：最后进入 commit 阶段，调用引擎的提交事务接口，将 redo log 状态设置为 commit。

#### 磁盘 I/O 很高，有什么优化的方法？

将 binlog 和 redo log 持久化到磁盘，那么如果出现 MySQL 磁盘 I/O 很高的现象，我们可以通过控制以下参数，来 “延迟” binlog 和 redo log 刷盘的时机，从而降低磁盘 I/O 的频率：

+ **设置组提交的两个参数**： `binlog_group_commit_sync_delay` 和 `binlog_group_commit_sync_no_delay_count`参数，延迟 binlog 刷盘的时机，从而减少 binlog 的刷盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但即使 **MySQL 进程中途挂了，也没有丢失数据的风险**，因为 binlog 早被写入到 page cache 了，**只要系统没有宕机**，缓存在 page cache 里的 binlog 就会被持久化到磁盘。
+ **将 sync_binlog 设置为大于 1 的值**（比较常见是 100~1000），表示每次提交事务都 write，但累积 N 个事务后才 fsync，相当于延迟了 binlog 刷盘的时机。但是这样做的风险是，**主机掉电时会丢 N 个事务的 binlog 日志**。
+ **将 innodb_flush_log_at_trx_commit 设置为 2**。表示每次事务提交时，都只是缓存在 redo log buffer 里的 redo log 写到 redo log 文件，注意写入到「 redo log 文件」并不意味着写入到了磁盘，因为操作系统的文件系统中有个 Page Cache，专门用来缓存文件数据的，所以写入「 redo log文件」意味着写入到了操作系统的文件缓存，然后交由操作系统控制持久化到磁盘的时机。但是这样做的风险是，**主机掉电的时候会丢数据**。

# 内存篇

**Buffer Pool** 除了缓存「索引页」和「数据页」，还包括了 undo 页，插入缓存、自适应哈希索引、锁信息等等。InnoDB 为每一个缓存页都创建了一个**控制块**，控制块信息包括「缓存页的表空间、页号、缓存页地址、链表节点」等等。

#### 空闲页

为了能够快速找到空闲的缓存页，可以使用链表结构，将空闲缓存页的「控制块」作为链表的节点，这个链表称为 **Free 链表**（空闲链表）。

<img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240316%E4%B8%8B%E5%8D%8872136284.png" alt="image-20240316下午72136284" style="zoom:50%;" />

> **脏页**也是如此管理。不过叫 **Flush链表**

#### 提高缓存命中率

1. LRU：访问就把结点放到头部，淘汰就从尾部淘汰

   1. 预读失效：mysql会把相邻的数据页一并加载（预读）但可能失效。最好就是**让预读的页停留在 Buffer Pool 里的时间要尽可能的短，让真正被访问的页才移动到 LRU 链表的头部，从而保证真正被读取的热数据留在 Buffer Pool 里的时间尽可能长**。

      将 LRU 划分了 2 个区域：**old 区域 和 young 区域**。old区域占整个链表的比例可由`innodb_old_blocks_pct`参数来设置（默认37）。预读的页加入到old区头部，真正被访问的页插入young头部。以及一个**小优化**：**young 区域前面 1/4 被访问不会移动到链表头部，只有后面的 3/4被访问了才会**

      <img src="./%E5%9B%BE%E8%A7%A3mysql.assets/image-20240316%E4%B8%8B%E5%8D%8873408746.png" alt="image-20240316下午73408746" style="zoom:50%;" />

   2. BufferPool污染：

      当某一个 SQL 语句**扫描了大量的数据**时，在 Buffer Pool 空间比较有限的情况下，可能会将 **Buffer Pool 里的所有页都替换出去，导致大量热数据被淘汰了**。称之为BufferPool污染。提高进入young区域的门槛

      > Buffer Pool 污染并不只是查询语句查询出了大量的数据才出现的问题，即使查询出来的结果集很小，也会造成 Buffer Pool 污染。比如全表扫描`select * from t_user where name like "%xiaolin%";`

      进入到 young 区域条件增加了一个**停留在 old 区域的时间判断**。这个间隔时间是由 `innodb_old_blocks_time` 控制的，默认是 1000 ms。也就说，**只有同时满足「被访问」与「在 old 区域停留时间超过 1 秒」两个条件，才会被插入到 young 区域头部**（为什么是超过一秒，大概是为了保证不是同一语句/业务的连续使用）。

### 脏页刷盘

不用担心宕机导致的丢失（先写日志，redolog）下面几种情况会触发脏页的刷新：

+ Buffer Pool 空间不足时，需要将一部分数据页淘汰掉，如果淘汰的是脏页，需要先将脏页同步到磁盘；
+ 当 redo log 日志满了的情况下，会主动触发脏页刷新到磁盘（buffer刷盘，循环覆盖日志）
+ MySQL 认为空闲时，后台线程会定期将适量的脏页刷入到磁盘；
+ MySQL 正常关闭之前，会把所有的脏页刷入到磁盘；