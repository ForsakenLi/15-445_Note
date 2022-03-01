# Query Planning & Optimization

## 1. 背景

SQL是声明式语言，当应用程序向数据库发送查询时，该查询会告诉DBMS想要什么结果而非如何计算。

DBMS拿到SQL语句，需要选定执行该语句的最佳方案（不同的方案会带来不同的性能）

## 2. Query Optimization

### 2.1. Heuristics 启发式的（基于规则的）

- 当查询中的某些部分满足了某种条件，会触发特定的规则，对query进行重写（修改），移除一些错误或低效的内容
- 需要取检查catalog（元数据）来理解表中有哪些东西，但是不需要去检查数据本身

### 2.2. Cost-based Search

- 枚举SQL所有可能的不同查询方案，使用一个**cost model**去预估各个方案的cost（预估查询计划中的工作量）
- 选取cost最低的方案

## 3. Architecture

![image](https://user-images.githubusercontent.com/29897667/125680407-a9a62926-c903-456c-8cbf-bc5f2a3d36a3.png)

- **SQL Rewriter (optional)**：通过某些转换规则以某种形式对SQL语句进行重写（如给一张表补充disk存储信息等）
- **Parser**：将SQL字符串转换为抽象语法树（AST, Abstract Syntax Tree）
- **binder**：负责将SQL query中所引用的命名对象转换为内部标识符（Name->Internal ID，通过询问system catalog来做到这点），它输出logic plan
- **Tree Rewriter (optional)**：静态规则、对AST进行重写（需要向system catalog问table的schema info），输出修改后的logic plan
- **Optimizer**：通过**cost model**来找出最佳方案（需要向system catalog问table的schema info），会生成**physical plan**，即数据库系统实际执行查询语句的方式

## 4. Logical vs Physical plans

Optimizer生成 **logical algebra expression** 到优化后的 **physical algebra expression** 的映射。

**logical plan**

- 高级层面来讲，这个查询想做的事情是什么（如想对某表进行扫描、对两张表进行join）
- logical plan中不会表明实际该怎么执行该查询

**physical plan**

- physical operator定义实际的执行策略（比如使用某个index，使用hash join）
- 可以取决于执行的数据的物理形式
- logical到physical plan的对应关系并不一定时一对一的

## 5. 等价关系代数（Relational Algebra Equivalences）

- 对query plan进行操作和转换以找到更好的备选方案

- 如果两个关系代数表达式可以产生相同的tuples集合（无序），那么它们时等价的（equivalent）
- 可以对关系代数或查询计划转换成相同效果的不同关系代数语句
- 被称为 **query rewriting**
- 例如predicate pushdown（尽早地进行条件过滤如下图所示，或是nested join时将size更小的table作为outer table）：

![image](https://user-images.githubusercontent.com/29897667/125835077-b069047e-111c-4d42-b9f6-0a4a820cd2a0.png)
![image](https://user-images.githubusercontent.com/29897667/125835110-16d57980-5ef6-4a58-b7df-30c37e01808d.png)
![image](https://user-images.githubusercontent.com/29897667/125835148-e891a551-2943-419a-96dc-06615923a7b5.png)

### 5.1. Selections

- 尽早进行过滤操作
- 对条件进行重排序，将更具选择性的过滤条件放在前面
- 将一个复杂的谓语(predicate)拆解，并下推

- ![image](https://user-images.githubusercontent.com/29897667/126024448-8c02fb0e-5b47-47d9-8c23-af8d8035a108.png)

- **例子**：可以将x和y直接和常量比较，而不需要先拿y的值，将其存在寄存器中再进行比较成本更低（这是一种对内存型数据库起作用的微优化）

![image](img/10-1.png)

### 5.2. Projections

- 尽早地进行projection去减少operator之间传递的数据（更少的属性以及去重），以及减少中间结果, 如下图中传递给join operator前，先进行projection，将除了sid(join连接需要)、name、cid以外的属性都去掉
- 对于行存数据库有用，对于列存数据库没用; 在分布式数据库中常用以降低网络传输数据量

![image](https://user-images.githubusercontent.com/29897667/126030311-21019629-bbe4-4274-832d-f23e4982221d.png)

## 6. Cost Estimation

- **CPU**：CPU使用率，比较小的cost，难以估计
- **Disk**：block转移的数量
- **Memory**：DRAM使用数量
- **Network**：对于分布式数据库，RPC消息的数量

对一个查询的所有有效计划进行详尽的枚举，对优化器来说太慢了。仅就连接而言，它是交换和关联的，每个n路连接有4^n种不同的排序。优化器必须限制其搜索空间，以便有效地工作。

为了估算查询的成本，DBMS在其内部目录中维护关于表、属性和索引的内部统计数据。不同的系统以不同的方式维护这些统计数据。大多数系统试图通过维护一个内部的统计表来避免即时计算。然后，这些内部表可以在后台更新。

- high level上看，将tuples被读写的数量作为cost中的参考值, 本质上来说就是operator间传递数据的大小

## 7. Statistics

### 7.1. Intro

每个DBMS都有查询优化器，使用cost-based search。

DBMS中，用来估算cost所使用的的基础部分就是DBMS内部的statistics catalog。

DBMS在internal catalog中存储关于table、attribute、index的internal statistics。

所有DBMS都会通过某个命令强制收集新的统计信息，如 MySQL的ANALYZE TABLE。作用是循序扫描表并更新统计信息。（或者定时更新、触发器更新，OLTP系统可以将其放在访问低峰期完成）

### 7.2. 前提假设

在计算predicate的选择基数时，使用了以下三个假设：
- 均匀数据: 值的分布是均匀的,所有数据的出现概率相同（对于heavy hitters(热键，出现频率远高于平均)，可使用单独hash table或直方图来跟踪）
- 独立的判断条件(predicate): 属性上的predicate是独立的
- Inclusion原则: 对于join计算，内表中的每个key都将也存在于外表中

但真实的数据往往不满足这些条件。例如，相关的属性打破了predicate的独立性假设


![image](https://user-images.githubusercontent.com/29897667/126061855-d8e2101a-ee0e-4f63-b1b9-cf6b9759806b.png)

### 7.3. 表示

- Nr: DBMS中存储的tuple的数量
- V(A,R): 对于属性A，经过去重后取值的种类数
- SC(A,R): 选择基数

![image](https://user-images.githubusercontent.com/29897667/126060384-42b544a4-c88f-490d-b3f8-62eecf597184.png)

### 7.4. Selection Statistics

![image](https://user-images.githubusercontent.com/29897667/126061905-fcfd7d05-34f2-4068-9b2b-fd01b8d39192.png)

估测的条件选择率 约等于 某个tuple复合给定条件的概率。

#### 7.4.1. Equality Predicate

![image](https://user-images.githubusercontent.com/29897667/126061946-7baf6a1e-866a-4642-ada3-fe2e8a3c2f12.png)

#### 7.4.2. Range Predicate

![image](https://user-images.githubusercontent.com/29897667/126061994-6c64a53f-eb4e-43d5-ac00-b36727ef05d1.png)

预测情况可能与真实情况不一致。

#### 7.4.3. Negation Query

![image](https://user-images.githubusercontent.com/29897667/126062018-b14dc81f-7e9c-4abf-b835-bde46c5acf38.png)

#### 7.4.4. Conjunction

#### 7.4.5. Disjunction

![image](https://user-images.githubusercontent.com/29897667/126062043-7c7ac323-cbf7-4c4d-9e5b-0d7413e0a6ee.png)

#### 7.4.6. Join Estimation

![image](https://user-images.githubusercontent.com/29897667/126067668-bf675487-19b4-41c4-862f-9251dceed012.png)

### 7.5. Statistics Storage

#### 7.5.1. Histograms

**直方图**为一个属性上值的分布，横坐标为属性值，纵坐标为值的record数。

![image](https://user-images.githubusercontent.com/29897667/126068117-4701123b-954c-4637-9892-9c531048fad6.png)

实际数据库中数据分布式不均匀的，因此维护一个存储数据库中每一种值的直方图开销很大。

可以将values放在buckets中来减少直方图的大小：

![image](https://user-images.githubusercontent.com/29897667/126068130-cc47fb62-4fbd-4e9f-a877-17821b4dac57.png)

但是这种情况下，频繁值会影响不频繁值的计数，可能导致不准确。

为解决这个问题，可以使用不等宽直方图，使用Quantiles让每个buckets的size差不多大。

![image](https://user-images.githubusercontent.com/29897667/126068146-9247728c-6905-4d6c-a5b5-306ebfcef4b2.png)
![image](https://user-images.githubusercontent.com/29897667/126068165-9936d75f-bb6f-4188-9f31-ac31dd05b1b7.png)

#### 7.5.2. Sampling

维护样本表，根据该样本表来衍生出统计信息。假设该样本中值的分布情况和表中真正的值分布情况是相同的，根据样本所预测的选择率就会反映出表的真实情况。

当表中很大一部分数据发生改变（如批量加载表中数据、或者批量删除表中数据），会触发样本表的更新。

如果查询很简单，直接使用直方图即可；如果查询涉及的工作量很大，则使用采样法。

## 8. Plan Enumeration

DBMS进行rule-based rewriting后，将列举不同的查询计划并估计其成本。然后，在有限的时间和查询计划枚举的数量范围内，它将为该查询选择最佳的Query Plan。

### 8.1. Single-Relation Query Plan

首先要选择最好的access method：

- Sequential Scan
- Binary Search (clustered indexes)
- Index Scan

其次需要选择评估条件的顺序，优先评估选择性更高的条件。

简单的启发式规则是适用的。

OLTP的query plan很简单因为它们是**sargeble(Search Argument Able)** 的：

- 通常选用最好的index
- join通常建立在外键上，基数很少
- 可以使用一些启发式规则来实现

### 8.2. Multi-Relation Query Plan

join的表越多，可选的plan数量也越多。需要对方案进行剪枝。

只考虑左深连接树（left-deep join tree）的情况（适用于**pipeline model**，DBMS不需要将连接运算符的输出具体化，如果DBMS的优化器只考虑左深树，那么它将减少搜索过程所使用的内存量，并有可能减少搜索时间。）

![image](https://user-images.githubusercontent.com/29897667/126079987-56565154-31b3-4125-820a-677d7fe315ab.png)

**如何枚举所有可能的query plan**：

![image](https://user-images.githubusercontent.com/29897667/126080020-06033d29-aa8d-4a03-a1c2-099dcca2d483.png)

可以使用**Dynamic Programming** 去减少query plan的可能数量：

每一步取最优(每一步都枚举所有可以使用的算法的cost，选择其中最小的那个)，然后对全局上所有保留的路径取一个最优。

下图忽略了数据的额外信息(如数据是否需要排序、数据有无经过压缩、使用的行存储还是列存储等)，这些信息对算法的cost同样存在影响，在实际进行query plan选择时需要考虑。

![image](https://user-images.githubusercontent.com/29897667/126080199-68511f0e-bb14-46fb-bac5-70a9970754c8.png)
![image](https://user-images.githubusercontent.com/29897667/126080206-1d68edf2-0b5a-49d8-8265-e9ada5060086.png)
![image](https://user-images.githubusercontent.com/29897667/126080219-5fd4a1c6-2658-4312-9021-176ae9cee038.png)

当查询的表格数少于12个时，Postgres使用这种传统的动态规划算法；而当查询的表格大于12个时，Postgres会改为使用GEQO遗传算法

## 9. Nested Sub-Queries

DBMS将WHERE子句中的 **嵌套子查询** 视为获取参数并返回单个值或一组值的函数。

两种优化方法：

1.重写以取消相关或展平查询。

2.分解嵌套查询并将结果存储在子表中

![image](https://user-images.githubusercontent.com/29897667/126081155-1e7fa026-0d0c-4f94-8c4f-25286fdabc7d.png)

