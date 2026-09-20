# 中级 SQL

整理自 `笔记/数据库系统原理笔记.pdf` 第 43–47 页和第 50–60 页（第 4 章的视图、视图更新、事务、连接，第 5 章的 5.1 数据类型和 5.2 完整性约束），`笔记/数据库期末复习题整理.pdf` 第 99–102 页、第 106–108 页（SQL 语句整理中的视图、事务、约束、断言、连接、类型与域）、第 116 页（类型判断题）、第 118 页（历年题第 7 题），以及 `老师资料/SQL复习 (1).docx` 2.17–2.20 节。笔记把类型和约束放在第 5 章，教材和课后习题的第 4 章也讲这两块，所以放在这里；第 4 章里的增删改（4.2.1–4.2.3）在 `03 SQL基础查询.md` 第 11 节。

## 本章要点

- 视图是虚关系，数据库只存它的定义，查询用到视图时把定义展开代入。物化视图把结果真正存下来，基表变了要做**视图维护**。
- 视图可更新的四个条件：from 里只有一个关系；select 里只有属性名，没有表达式、聚集、`distinct`；没出现在 select 里的属性都允许为空；没有 `group by` 和 `having`。
- 事务是一组要么全做、要么全不做的语句，`commit work` 提交，`rollback work` 回滚；提交之后不能再回滚。
- 连接 = 连接类型（`inner join`、`left outer join`、`right outer join`、`full outer join`）+ 连接条件（`natural`、`on`、`using`），两两可以组合。
- 外连接先做内连接，再把某一侧没配上的元组补进来，另一侧的属性填 null。
- 左外连接里，写在 `on` 的条件不会让左表元组丢失，写在 `where` 的条件会把不满足的行过滤掉；内连接两者结果相同。
- 单个关系上的约束：`not null`、`unique`、`primary key`、`check`；参照完整性用 `foreign key ... references`，可以加 `on delete cascade`、`on update cascade`；跨表的条件可以写成断言。
- `check` 的结果为 unknown 时算满足，只有结果为 false 才违反约束。
- `create type` 定义强类型，不能带约束；`create domain` 可以带 `not null`、`check`、默认值，但不是强类型。

## 1. 视图

### 1.1 定义与使用

有时不想让用户看到整个表（比如教师工资），或者想把常用的复杂查询存起来反复用，就定义视图：

```sql
create view v as <查询表达式>;
```

`<查询表达式>` 是任意合法的查询。视图只保存这个定义，不预先算结果；查询用到视图时才执行定义它的查询。不是逻辑模型的一部分、但作为虚关系给用户看的关系，都叫视图。视图可以出现在任何能写关系名的地方。

老师资料：不含工资的教师视图，以及用它查生物系教师。

```sql
create view faculty as
    select ID, name, dept_name
    from instructor;

select name
from faculty
where dept_name = 'Biology';
```

笔记：由银行名和它的顾客（存款的和贷款的都算）构成的视图。

```sql
create view all_customer as
    (select branch_name, customer_name        -- 存款顾客
     from depositor, account
     where depositor.account_number = account.account_number)
    union
    (select branch_name, customer_name        -- 贷款顾客
     from borrower, loan
     where borrower.loan_number = loan.loan_number);

-- 列出 Perryridge 银行的顾客
select customer_name
from all_customer
where branch_name = 'Perryridge';
```

视图的属性可以在视图名后面显式命名。由银行名和它的贷款总额构成的视图：

```sql
create view branch_total_loan (branch_name, total_loan) as
    select branch_name, sum(amount)
    from loan
    group by branch_name;
```

（更正：原文作 `sum(account)`，`loan` 表里表示金额的属性是 `amount`；题目原作"存款总额"，按查询的内容应为"贷款总额"。）

删除视图用 `drop view v`。创建、删除视图只影响视图本身，不动基表。

### 1.2 优点与不足

优点：

- 简化操作。常用查询定义成视图，使用者不用关心背后有哪些表、怎么连接。
- 保护数据。只把视图的权限给用户，就能把他限制在一部分行和列上，比如 `faculty` 看不到 `salary`。
- 应对需求变化。表结构调整后，可以只改视图定义，上层查询不用动。
- 分解复杂查询。先建几个视图分别求出中间结果，再把它们组合起来。
- 视图只存定义，不额外存数据（物化视图除外）。

不足：

- 基表结构变了，相关视图要跟着维护。视图套视图时尤其难读、难改。
- 项目里视图太多，维护成本会上去。
- 笔记建议视图主要用来查询，尽量不要通过视图改数据，原因见 1.5 节。

### 1.3 视图展开

设视图 v1 由表达式 e1 定义，e1 里可能还用到别的视图。展开的过程：

```text
repeat
    找出 e1 中的任一视图 vi
    用定义 vi 的表达式替换 vi
until e1 中不再有视图
```

只要视图不是递归定义的，这个循环一定会结束。

例：在 `all_customer` 上再定义一个视图，然后查询它。

```sql
create view all_customer as
    (select branch_name, customer_name
     from depositor as R, account as S
     where R.account_number = S.account_number)
    union
    (select branch_name, customer_name
     from borrower as T, loan as P
     where T.loan_number = P.loan_number);

create view perryridge_customer as
    select customer_name
    from all_customer
    where branch_name = 'Perryridge';

select *
from perryridge_customer
where customer_name = 'John';
```

第一步，把 `perryridge_customer` 换成它的定义：

```sql
select *
from (select customer_name
      from all_customer
      where branch_name = 'Perryridge') as pc
where customer_name = 'John';
```

第二步，把 `all_customer` 换成它的定义，这时已经没有视图了：

```sql
select *
from (select customer_name
      from ((select branch_name, customer_name
             from depositor as R, account as S
             where R.account_number = S.account_number)
            union
            (select branch_name, customer_name
             from borrower as T, loan as P
             where T.loan_number = P.loan_number)) as ac
      where branch_name = 'Perryridge') as pc
where customer_name = 'John';
```

笔记里的展开图没给派生表起名字，这里加上 `pc`、`ac`，写成能直接执行的形式。

### 1.4 物化视图

物化视图把定义它的查询结果真正存下来，也叫快照（snapshot）。好处是查询快，尤其是复杂的聚集查询，不用每次都重新算。

代价是基表变了，存下来的结果会过时。让物化视图保持最新的过程叫**物化视图维护**，简称视图维护。可以设置成基表一变就立即更新，也可以定期或手动刷新。

### 1.5 通过视图更新数据

对视图做插入、删除、更新，最终要翻译成对基表的修改。有些修改翻译不过去，或者翻译后产生怪结果。

定义视图的查询满足下面全部条件时，SQL 视图才是**可更新的**：

1. from 子句里只有一个数据库关系；
2. select 子句里只有属性名，没有表达式、聚集函数，也没有 `distinct`；
3. 没出现在 select 里的属性都可以取空值，也就是这些属性上没有 `not null` 约束，也不是主码的一部分；
4. 查询里没有 `group by` 和 `having`。

（更正：笔记第 50 页把第 3 条写成"select 中属性不可以取空值"，意思反了。第 45 页和复习题整理第 100 页的写法是对的。复习题整理补了一句，可以换个说法记：select 里要包含基表所有不允许为空的属性。）

笔记举了四个出问题的例子。

例 1：通过 `loan_branch` 插入贷款号 L-37 和银行 Perryridge。

```sql
create view loan_branch as
    select loan_number, branch_name
    from loan;

insert into loan_branch
values ('L-37', 'Perryridge');
```

视图里没有 `amount`，插进 `loan` 的元组只能是 `('L-37', 'Perryridge', null)`。这个视图满足四个条件，插入能成功，但贷款额是空值。

例 2：通过 `loan_info` 插入 Johnson 贷款 1900。

```sql
create view loan_info as
    select customer_name, amount
    from borrower, loan
    where borrower.loan_number = loan.loan_number;

insert into loan_info
values ('Johnson', 1900);
```

（更正：原文定义里漏了 where 连接条件，那样视图是两表的笛卡儿积。）

要完成插入，只能往 `borrower` 插 `('Johnson', null)`，往 `loan` 插 `(null, null, 1900)`。两边的贷款号都是 null，`null = null` 是 unknown，这条数据在视图里根本查不出来。况且 `loan_number` 是 `loan` 的主码，不能为空，实际会被拒绝。from 里有两个关系，本来就不满足可更新条件。

例 3：通过 `loan_downtown` 插入 `('L-37', 'Perryridge', 800)`。

```sql
create view loan_downtown as
    select loan_number, branch_name, amount
    from loan
    where branch_name = 'Downtown';

insert into loan_downtown
values ('L-37', 'Perryridge', 800);
```

元组能插进 `loan`，但它的银行是 Perryridge，不满足视图的 where 条件，插完在视图里看不到。整理补充：在视图定义末尾加 `with check option`，系统会拒绝这种插进去却不满足视图条件的修改。

例 4：通过 `all_customer` 插入 `('Perryridge', 'John')`。视图由两个查询 union 而成，系统没法判断这个元组该插进存款那边还是贷款那边，而且两边都缺账号或贷款号。

### 1.6 视图与 with 子句

两者都能给一段查询起名字，区别见文末易混对比。笔记注明这一部分不是课程内容，是整理者自己查资料补的。

## 2. 事务

事务是由一系列查询和更新语句组成的逻辑工作单元。这些语句作为一个整体执行：要么全部执行，要么全部不执行，这是**原子性**。

| 语句 | 作用 |
| --- | --- |
| `commit work` | 提交当前事务，所做的修改永久写入数据库 |
| `rollback work` | 回滚当前事务，撤销它做的所有修改，数据库回到事务开始前的状态 |

- 事务一旦提交，它的影响就不能再用回滚撤销（复习题整理第 100 页）。
- 一个事务可以只有一条 SQL 语句，也可以是一组语句。标准 SQL 用 `begin atomic ... end` 把一组语句括成一个事务：

```sql
begin atomic
    ...
end
```

整理补充：很多数据库默认每条语句执行完自动提交。MySQL 里显式开启事务写 `start transaction`，再用 `commit` 或 `rollback` 结束。

转账是最典型的例子：一个账户扣钱、另一个账户加钱，两条 update 必须一起成功，写法见典型题 13。一条 update 只能改一个表，要同时改几张表，也是把几条语句放进同一个事务。事务的 ACID 性质、调度和隔离级别见 `10 事务.md`。

## 3. 连接表达式

### 3.1 连接类型与连接条件

| 连接类型 | 连接条件 |
| --- | --- |
| `inner join` | `natural` |
| `left outer join` | `on <谓词>` |
| `right outer join` | `using (A1, A2, ..., An)` |
| `full outer join` | |

两列可以任意搭配，比如 `natural left outer join`、`full outer join ... using (...)`。

（更正：老师资料的图里写成 `using (A1, A1, ..., An)`，第二个应为 `A2`。）

- 为了和外连接区分，普通连接叫内连接。只写 `join` 就是 `inner join`。
- `natural`：按两个关系的全部同名属性做等值连接，同名属性在结果里只留一份。
- `on <谓词>`：按给定条件连接，两边的属性都保留，同名属性会出现两次。
- `using (A1, ..., An)`：只按列出的同名属性做等值连接，这些属性只留一份。复习题整理把它比作"挑选属性的等值连接"。

### 3.2 示例数据与结果

老师资料用的两个关系：

`course`

| `course_id` | `title` | `dept_name` | `credits` |
| --- | --- | --- | --- |
| BIO-301 | Genetics | Biology | 4 |
| CS-190 | Game Design | Comp. Sci. | 4 |
| CS-315 | Robotics | Comp. Sci. | 3 |

`prereq`

| `course_id` | `prereq_id` |
| --- | --- |
| BIO-301 | BIO-101 |
| CS-190 | CS-101 |
| CS-347 | CS-101 |

注意 CS-315 没有先修课记录，CS-347 在 `course` 里没有。

`course natural join prereq`（内连接，整理补充）：

| `course_id` | `title` | `dept_name` | `credits` | `prereq_id` |
| --- | --- | --- | --- | --- |
| BIO-301 | Genetics | Biology | 4 | BIO-101 |
| CS-190 | Game Design | Comp. Sci. | 4 | CS-101 |

`course natural left outer join prereq`：保留 `course` 的全部元组。

| `course_id` | `title` | `dept_name` | `credits` | `prereq_id` |
| --- | --- | --- | --- | --- |
| BIO-301 | Genetics | Biology | 4 | BIO-101 |
| CS-190 | Game Design | Comp. Sci. | 4 | CS-101 |
| CS-315 | Robotics | Comp. Sci. | 3 | null |

`course natural right outer join prereq`：保留 `prereq` 的全部元组。

| `course_id` | `title` | `dept_name` | `credits` | `prereq_id` |
| --- | --- | --- | --- | --- |
| BIO-301 | Genetics | Biology | 4 | BIO-101 |
| CS-190 | Game Design | Comp. Sci. | 4 | CS-101 |
| CS-347 | null | null | null | CS-101 |

`course natural full outer join prereq`：两边都保留。

| `course_id` | `title` | `dept_name` | `credits` | `prereq_id` |
| --- | --- | --- | --- | --- |
| BIO-301 | Genetics | Biology | 4 | BIO-101 |
| CS-190 | Game Design | Comp. Sci. | 4 | CS-101 |
| CS-315 | Robotics | Comp. Sci. | 3 | null |
| CS-347 | null | null | null | CS-101 |

这里两个关系只有 `course_id` 同名，所以 `course full outer join prereq using (course_id)` 的结果和上表一样。

换成 `on`，同名属性保留两份（整理补充）：

```sql
select *
from course left outer join prereq
     on course.course_id = prereq.course_id;
```

| `course.course_id` | `title` | `dept_name` | `credits` | `prereq.course_id` | `prereq_id` |
| --- | --- | --- | --- | --- | --- |
| BIO-301 | Genetics | Biology | 4 | BIO-301 | BIO-101 |
| CS-190 | Game Design | Comp. Sci. | 4 | CS-190 | CS-101 |
| CS-315 | Robotics | Comp. Sci. | 3 | null | null |

### 3.3 笔记里银行模式的写法

| 表达式 | 结果 |
| --- | --- |
| `loan inner join borrower on loan.loan_number = borrower.loan_number` | 两表贷款号相同的元组配对 |
| `loan left outer join borrower on loan.loan_number = borrower.loan_number` | 上面的结果，再加上 `loan` 里没配上的元组，`borrower` 的属性填 null |
| `loan right outer join borrower on loan.loan_number = borrower.loan_number` | 再加上 `borrower` 里没配上的元组，`loan` 的属性填 null |
| `loan full outer join borrower on loan.loan_number = borrower.loan_number` | 两边没配上的都加上 |
| `loan natural right outer join borrower` | 按同名属性 `loan_number` 连接，保留 `borrower` 的全部元组 |
| `loan full outer join borrower using (loan_number)` | 只按 `loan_number` 连接，两边都保留，`loan_number` 只留一份 |

用外连接写"有存款无贷款""只有存款或只有贷款"的例子在典型题 3、4。

### 3.4 左外连接里 on 和 where 的区别

复习题整理第 102–103 页的说法：

- `on` 条件是生成连接结果时用的。不管 on 条件是否为真，左表的元组都会出现在结果里，配不上的右边填 null。
- `where` 条件是连接结果生成之后再过滤用的，这时已经没有"左表必须保留"的含义，不满足的行全部去掉。
- 内连接时，条件放 on 和放 where 结果一样。

用上面的数据对比（整理补充）：

```sql
-- 写法一：条件放在 on 里，结果 3 行
select *
from course left outer join prereq
     on course.course_id = prereq.course_id
    and prereq.prereq_id = 'CS-101';

-- 写法二：条件放在 where 里，结果 1 行
select *
from course left outer join prereq
     on course.course_id = prereq.course_id
where prereq.prereq_id = 'CS-101';
```

写法一：BIO-301 的先修课不是 CS-101，配不上，右边填 null，但 BIO-301 仍保留；CS-190 配上 CS-101；CS-315 右边为 null。一共 3 行。

写法二：先得到 3.2 节最后那张 3 行的表，再按 where 过滤，只剩 CS-190 一行。null 和 `'CS-101'` 比较是 unknown，也被滤掉。

换成内连接，两种写法都只有 CS-190 一行。

### 3.5 自然连接的陷阱

题目：列出教师姓名和他教过的课程名。下面这样写是错的（整理补充，教材上的例子）：

```sql
select name, title
from instructor natural join teaches natural join course;
```

`instructor natural join teaches` 的结果里带着教师所在系 `dept_name`，`course` 里也有 `dept_name`，再做自然连接就会顺带要求"教师所在系 = 课程所属系"，教外系课程的记录全丢了。只按课程号连接：

```sql
select name, title
from (instructor natural join teaches) join course using (course_id);
```

### 3.6 MySQL 里的全外连接

MySQL 不支持 `full outer join`，可以把左外连接和右外连接的结果 union 起来。两边的列要一一对应，所以列名写全：

```sql
select course.course_id, title, dept_name, credits, prereq_id
from course left outer join prereq
     on course.course_id = prereq.course_id
union
select prereq.course_id, title, dept_name, credits, prereq_id
from course right outer join prereq
     on course.course_id = prereq.course_id;
```

结果和 3.2 节的全外连接表相同。

## 4. 数据类型与模式

### 4.1 日期和时间

| 类型 | 存什么 | 常量写法 |
| --- | --- | --- |
| `date` | 年月日 | `date '2024-05-05'` |
| `time` | 时分秒 | `time '10:37:47'` |
| `timestamp` | 日期加时间 | `timestamp '2024-05-05 10:37:47'` |

课程 `README.md` 回忆 22 级期末最后一道 SQL 题涉及视图的创建和对 date 类型的操作。常用写法（整理补充）：

| 需求 | 标准 SQL | MySQL |
| --- | --- | --- |
| 取年份、月份 | `extract(year from d)`、`extract(month from d)` | 同左，也可以用 `year(d)`、`month(d)` |
| 当前日期 | `current_date` | `current_date` 或 `curdate()` |
| 日期加 7 天 | `d + interval '7' day` | `date_add(d, interval 7 day)` |
| 两个日期相差几天 | 两个日期相减得到一个 interval 值 | `datediff(d1, d2)`，返回 d1 减 d2 的天数 |
| 比较 | `d >= date '2024-01-01'` | `d >= '2024-01-01'` |

### 4.2 类型转换：cast

```sql
cast(account.balance as numeric(12,2))
```

把 `account.balance` 转成总共 12 位、2 位小数的数值。（更正：笔记和复习题整理都写作 `cast(... to numeric(12,2))`，SQL 的写法是 `cast(表达式 as 类型)`。）

另一个用处：`ID` 是字符串类型时，`order by ID` 按字符顺序排，想按数值排写 `order by cast(ID as numeric(5))`（整理补充）。

### 4.3 默认值

整理补充（笔记第 58 页的约束列表里提到了 `DEFAULT`）：

```sql
create table student (
    ID        varchar(5),
    name      varchar(20) not null,
    dept_name varchar(20),
    tot_cred  numeric(3,0) default 0,
    primary key (ID)
);

insert into student (ID, name, dept_name)
values ('12789', 'Newman', 'Comp. Sci.');   -- tot_cred 自动取 0
```

### 4.4 大对象类型

| 类型 | 存什么 | 例子 |
| --- | --- | --- |
| `clob` | 字符数据大对象 | `book_review clob(10KB)`：最多 10KB 文本 |
| `blob` | 二进制数据大对象 | `image blob(10MB)`、`movie blob(2GB)` |

大对象一般不整个读进内存。查询返回的是一个"定位器"，程序再通过它一段一段地读（整理补充）。

### 4.5 用户定义类型：create type

```sql
create type Dollars as numeric(12,2) final;
create type Pounds  as numeric(12,2) final;

create table account (
    account_number char(10),
    branch_name    char(15),
    balance        Dollars
);
```

- `Dollars` 是一个新类型，底层是 `numeric(12,2)`。
- 用户定义类型是**强类型**：`Dollars` 和 `Pounds` 底层一样，也不能直接比较、赋值；`Dollars` 类型的属性和普通数值常量做运算也不行，要先 `cast`。
- 不能在类型上声明约束或默认值。
- `final` 是 SQL:1999 标准要求写的，在这里没有实际作用，有的系统允许省略。
- 删除和修改：`drop type`、`alter type`。

### 4.6 域：create domain

```sql
create domain DDollars as numeric(12,2);
create domain Dollars numeric(12,2) not null;       -- as 可以省略
create domain person_name char(20) not null;

create domain HourWage numeric(5,2)
    constraint wage_value_test check (value >= 8.00);   -- 小时工资不低于 8 美元

create domain AccountType char(10)
    constraint account_type_test check (value in ('Checking', 'Saving'));

create domain degree_level varchar(10)
    constraint degree_level_test check (value in ('Bachelors', 'Masters', 'Doctorate'));
```

- `constraint 名字` 给约束起名，出错信息里会带这个名字，以后也能按名字删除约束。`value` 指代这个域的值。
- 列的类型写成某个域，这个域上的 `not null`、`check` 就对这一列生效。

（更正：笔记第 56 页用一大段话说域上的约束"不是强制的"，插入不满足约束的值也能成功。这是把"域不是强类型"理解错了。域上的约束会检查，违反就拒绝插入；"不是强类型"指的是基本类型相容的两个域之间可以直接赋值、比较，不需要 `cast`。复习题整理第 108 页那句括号里的补充同样应删掉。）

### 4.7 类型和域的区别

| | `create type` | `create domain` |
| --- | --- | --- |
| 约束、默认值 | 不能声明 | 可以声明 `not null`、`check`、默认值 |
| 类型检查 | 强类型，不同类型之间要 `cast` | 不是强类型，基本类型相容就能互相赋值 |
| 用途 | 指定属性类型，也用在 SQL 过程扩展这类没法加约束的地方 | 给一类属性统一加上类型和约束 |

老师资料 2.20 的一句话总结：type 强类型检查、不能加约束；domain 弱类型检查、可以加约束。

MySQL 既不支持 `create type` 也不支持 `create domain`；PostgreSQL 支持 `create domain`。

## 5. 完整性约束

完整性约束防止合法用户的修改破坏数据的一致性，比如账户余额不能为负、学期只能取几个固定值。违反约束的插入、删除、更新会被拒绝。

### 5.1 单个关系上的约束

| 约束 | 含义 |
| --- | --- |
| `not null` | 属性不允许取空值 |
| `unique (A1, ..., An)` | 这组属性构成候选码，任意两个元组在上面取值不同；候选码属性**可以为空**（主码不行） |
| `primary key (A1, ..., An)` | 相当于 `unique` 加 `not null` |
| `check (P)` | 关系里每个元组都要满足谓词 P |
| `default` | 没给值时用的默认值 |
| `foreign key` | 参照完整性，见 5.2 |

`not null` 的例子：

```sql
create table account (
    account_number char(10),
    branch_name    char(15) not null,
    balance        numeric(12,2) not null,
    primary key (account_number)
);
```

`check` 的例子。建立 `branch` 表并保证资产非负：

```sql
create table branch (
    branch_name char(15),
    branch_city char(30),
    assets      numeric(16,2),
    primary key (branch_name),
    check (assets >= 0)
);
```

（更正：笔记第 57 页和复习题整理第 106 页原文在 `assets numeric(16,2)` 后面缺逗号，`primary key (branch name)` 缺下划线。）

老师资料：学期只能取四个值。

```sql
create table section (
    course_id    varchar(8),
    sec_id       varchar(8),
    semester     varchar(6),
    year         numeric(4,0),
    building     varchar(15),
    room_number  varchar(7),
    time_slot_id varchar(4),
    primary key (course_id, sec_id, semester, year),
    check (semester in ('Fall', 'Winter', 'Spring', 'Summer'))
);
```

（更正：原文 `time slot id` 缺下划线。）

几点要记：

- `check` 条件算出来是 unknown 时算满足，只有 false 才违反。所以 `check (assets >= 0)` 挡不住 `assets` 为 null 的元组，要禁止空值还得加 `not null`（整理补充）。
- `constraint`、`check`、`value` 都是保留字。
- 约束可以在 `create table` 时声明，也可以用 `alter table` 加上，可以作用于一列或多列。
- MySQL 8.0.16 之前会解析 `check` 但不检查。

### 5.2 参照完整性

一个关系在某些属性上的取值，必须在另一个关系的特定属性上出现过，这叫参照完整性。例：`account` 里每个元组的 `branch_name` 必须是 `branch` 里存在的银行。

笔记先写了一个用 check 表达的版本：

```sql
create table account (
    account_number char(10),
    branch_name    char(15),
    balance        numeric(12,2),
    primary key (account_number),
    check (branch_name in (select branch_name from branch))
);
```

（更正：原文 `primary key(account number)` 缺下划线，末尾还少一个右括号。）

标准 SQL 允许 check 里写子查询，但几乎没有数据库实现。而且这个 check 只在修改 `account` 时检查，删掉 `branch` 里的银行时不会检查，保证不了参照完整性。实际要用外码实现：

```sql
create table customer (
    customer_name   char(20),
    customer_street char(30),
    customer_city   char(30),
    primary key (customer_name)
);

create table depositor (
    customer_name  char(20),
    account_number char(10),
    primary key (customer_name, account_number),
    foreign key (customer_name) references customer,
    foreign key (account_number) references account
);
```

`depositor` 有两个外码：`customer_name` 参照 `customer`，`account_number` 参照 `account`。`depositor` 里出现的顾客名必须在 `customer` 里有，账号必须在 `account` 里有。

老师资料的写法，外码直接写在属性后面：

```sql
create table course (
    course_id char(5) primary key,
    title     varchar(20),
    dept_name varchar(20) references department
);
```

`primary key` 和 `references` 都可以直接跟在属性后面，只涉及一个属性时这样写更简洁。

规则：

- `references s` 默认参照 s 的主码；写成 `references s(A)` 时，A 必须是 s 的主码或声明为 `unique` 的候选码。
- 往参照表插入、修改元组，外码值在被参照表里找不到，就拒绝。
- 删除被参照表的元组、或修改它的码值，导致有元组"悬空"时，默认也拒绝。
- 外码属性允许为空；外码里有 null 时，这个约束自动算满足。

### 5.3 级联

被参照的元组删除或修改时，除了拒绝，还可以让参照它的元组跟着删、跟着改：

```sql
create table account (
    account_number char(10),
    branch_name    char(15),
    balance        numeric(12,2),
    primary key (account_number),
    foreign key (branch_name) references branch
        on delete cascade
        on update cascade,
    check (balance >= 0)
);
```

- `on delete cascade`：在 `branch` 里删掉一家银行，`account` 里这家银行的账户也一起删掉。
- `on update cascade`：把 `branch` 里某家银行改名，`account` 里对应账户的 `branch_name` 也跟着改。
- 整理补充：除了 `cascade`，还可以写 `on delete set null`（外码置空）、`on delete set default`（外码置默认值）。

### 5.4 断言

断言（assertion）是数据库必须一直满足的条件，可以跨多个表。

```sql
create assertion <断言名> check <谓词>;
```

SQL 没有"对所有 X"的写法，用双重否定表示：所有 X 都满足 P(X)，等价于不存在不满足 P(X) 的 X，写成 `not exists (... where not P ...)`。

例 1：每家银行的贷款总额必须小于它的存款余额总和。

```sql
create assertion sum_constraint check
    (not exists (select *
                 from branch
                 where (select sum(amount)
                        from loan
                        where loan.branch_name = branch.branch_name)
                       >= (select sum(balance)
                           from account
                           where account.branch_name = branch.branch_name)));
```

子查询里的 `loan.branch_name = branch.branch_name` 相当于按银行分组。存在一家银行贷款总额大于等于存款总额，断言就被违反。

（更正：笔记第 60 页和复习题整理第 102 页原文作 `sum(account) from loan`，应为 `sum(amount)`。）

整理补充：某家银行有贷款却一个存款账户都没有时，右边的 `sum(balance)` 是 null，比较结果为 unknown，这家银行不会被 `not exists` 抓到。要堵住这个漏洞，把右边写成 `coalesce((select sum(balance) ...), 0)`。

例 2：每笔贷款至少有一个贷款人的存款账户余额不少于 1000。

```sql
create assertion balance_constraint check
    (not exists (select *
                 from loan
                 where not exists (select *
                                   from borrower, depositor, account
                                   where loan.loan_number = borrower.loan_number
                                     and borrower.customer_name = depositor.customer_name
                                     and depositor.account_number = account.account_number
                                     and account.balance >= 1000)));
```

存在一笔贷款，它的贷款人里找不出余额不少于 1000 的账户，断言就被违反。

（更正：原文 from 里把 `depositor` 拼成了 `depositer`。）

断言每次相关的表修改都要检查，开销大，主流数据库基本都不支持 `create assertion`，实际多用触发器实现，触发器见 `08 应用开发、触发器与授权.md`。

## 易混对比

| 对比项 | 区别 |
| --- | --- |
| 内连接 / 外连接 | 内连接只要配上的元组；外连接还保留一侧或两侧没配上的元组，缺的属性填 null |
| 左 / 右 / 全外连接 | 保留左表 / 右表 / 两表中没配上的元组 |
| `natural join` / `join ... using (A)` / `join ... on P` | 按全部同名属性连接，同名属性留一份 / 只按列出的属性连接，留一份 / 按任意条件连接，两边属性都保留 |
| 外连接中条件放 `on` / 放 `where` | on：左表元组不会丢，配不上填 null；where：连接完再过滤，配不上的被滤掉。内连接两者相同 |
| 视图 / 基本表 | 视图只存定义，用时展开；基本表存数据 |
| 视图 / 物化视图 | 视图每次用时重新计算；物化视图存结果，查询快，但要维护 |
| 视图 / `with` 子句 | 视图是持久的数据库对象，建好后任何查询都能用，直到 `drop view`；with 只在定义它的那条查询里有效，查询结束就没了。可更新视图能直接修改，with 的结果不能修改 |
| 可更新视图 / 不可更新视图 | 单表、只有属性名、未选属性可空、无分组，四条全满足才可更新 |
| `commit` / `rollback` | 提交，修改永久生效 / 撤销当前事务的全部修改；提交之后不能再回滚 |
| `primary key` / `unique` | 主码不能为空，一个表只有一个；unique 允许为空，可以有多个 |
| `check` / 外码 | check 只在修改本表元组时检查；外码在参照表和被参照表修改时都检查 |
| `check` / 断言 | check 挂在一个关系或域上；断言独立定义，可以跨多个表 |
| 默认的外码处理 / `cascade` | 默认拒绝会导致悬空引用的删除和更新；cascade 让参照元组跟着删、跟着改 |
| `create type` / `create domain` | 强类型，不能加约束 / 不是强类型，可以加约束和默认值 |
| `clob` / `blob` | 字符大对象 / 二进制大对象 |
| `date` / `time` / `timestamp` | 日期 / 时间 / 日期加时间 |

## 典型题与解答

### 题 1：列出所有课程及其先修课，没有先修课的课程也要列出

思路：以 `course` 为主，保留全部课程，用左外连接。

```sql
select course_id, title, prereq_id
from course natural left outer join prereq;
```

用 3.2 节的数据，结果是 BIO-301 配 BIO-101、CS-190 配 CS-101、CS-315 配 null。

### 题 2：找出没有先修课的课程

思路：左外连接后，右边属性为 null 的就是没配上的。

```sql
select course_id, title
from course natural left outer join prereq
where prereq_id is null;
```

结果只有 CS-315。不用外连接也能写：`where course_id not in (select course_id from prereq)`。

### 题 3：列出在银行有存款而没有贷款的顾客

思路：`depositor` 左外连接 `borrower`，按顾客名配对，配不上贷款的顾客右边为 null。

```sql
select distinct d_CN
from (depositor left outer join borrower
      on depositor.customer_name = borrower.customer_name)
     as db1 (d_CN, account_number, b_CN, loan_number)
where b_CN is null;
```

用 `on` 连接后两个 `customer_name` 都保留，所以要用 `as db1 (...)` 给四列重新命名才分得清。`b_CN` 为 null 说明这个顾客在 `borrower` 里没有记录。一个顾客可能有多个存款账户，这里加了 `distinct`，原文没有加。

### 题 4：列出在银行只有存款或只有贷款的顾客

思路：两表按 `customer_name` 做自然全外连接，结果有 `customer_name`、`account_number`、`loan_number` 三列。只有存款的人 `loan_number` 为 null，只有贷款的人 `account_number` 为 null。

```sql
select customer_name
from (depositor natural full outer join borrower)
where account_number is null or loan_number is null;
```

MySQL 不支持全外连接，可以换成集合运算（MySQL 8.0.31 起支持 except）：

```sql
((select customer_name from depositor)
 except
 (select customer_name from borrower))
union
((select customer_name from borrower)
 except
 (select customer_name from depositor));
```

### 题 5：统计每个系的教师人数，没有教师的系显示 0

思路：从 `department` 出发左外连接 `instructor`，没有教师的系右边全是 null。计数要数 `I.ID`。

```sql
select D.dept_name, count(I.ID) as num_instructors
from department as D left outer join instructor as I
     on D.dept_name = I.dept_name
group by D.dept_name;
```

没有教师的系只有一行，`I.ID` 为 null，`count(I.ID)` 得 0；写 `count(*)` 会得 1。

### 题 6：左外连接中 on 和 where 的区别

题目：对 3.2 节的数据，下面两条查询各返回几行？

```sql
select * from course left outer join prereq
     on course.course_id = prereq.course_id and prereq_id = 'CS-101';

select * from course left outer join prereq
     on course.course_id = prereq.course_id
where prereq_id = 'CS-101';
```

思路：on 里的条件只决定能不能配上，左表元组一律保留；where 在连接完成后过滤。

解答：第一条 3 行（BIO-301 和 CS-315 的右边为 null，CS-190 配上 CS-101）；第二条 1 行（只有 CS-190）。

### 题 7：定义不含工资的教师视图，并通过它插入一名教师

思路：`faculty` 只选了 `ID`、`name`、`dept_name`，来自单个表，没有聚集；没选的 `salary` 允许为空，满足可更新的四个条件。

```sql
create view faculty as
    select ID, name, dept_name
    from instructor;

insert into faculty
values ('30765', 'Green', 'Music');
```

插入被翻译成往 `instructor` 插 `('30765', 'Green', 'Music', null)`。如果 `salary` 声明了 `not null`，这个视图就不可更新，插入会被拒绝。

### 题 8：判断下列视图能否更新

题目：1.1 和 1.5 节里的 `branch_total_loan`、`loan_branch`、`loan_info`、`all_customer`、`loan_downtown`。

思路：逐条对照四个条件。

| 视图 | 能否更新 | 原因 |
| --- | --- | --- |
| `branch_total_loan` | 不能 | 有聚集 `sum` 和 `group by` |
| `loan_branch` | 能 | 单表，只有属性名；未选的 `amount` 可空。插入后 `amount` 为 null |
| `loan_info` | 不能 | from 里有两个关系 |
| `all_customer` | 不能 | 由两个查询 union 而成，涉及多个关系 |
| `loan_downtown` | 能 | 单表、只有属性名、无分组。但插入不满足 where 的元组后在视图里看不到，要禁止就加 `with check option` |

### 题 9：定义视图 `v_customer`，求出 Austin 支行的所有客户姓名（历年题整理第 7 题）

思路：用 `03 SQL基础查询.md` 题 15 前面列出的那个带 `customer_id` 的银行模式。"支行的客户"包括在这家支行存款的和贷款的，两部分 union 起来。原稿无答案，以下为整理补充，`25 SQL与关系代数应用题.md` 里也有这一题。

```sql
create view v_customer as
    (select C.customer_name
     from customer as C, depositor as D, account as A
     where C.customer_id = D.customer_id
       and D.account_number = A.account_number
       and A.branch_name = 'Austin')
    union
    (select C.customer_name
     from customer as C, borrower as B, loan as L
     where C.customer_id = B.customer_id
       and B.loan_number = L.loan_number
       and L.branch_name = 'Austin');

select * from v_customer;
```

如果题意只指存款客户，保留 union 前半部分即可。

### 题 10：建表时声明约束

题目：建 `account` 表。余额不能为负；`branch_name` 必须是 `branch` 中存在的银行；银行被删除时，它的账户一起删除；银行改名时账户跟着改。

思路：`check` 管取值范围，外码管存在性，`on delete cascade`、`on update cascade` 管级联。

```sql
create table account (
    account_number char(10),
    branch_name    char(15) not null,
    balance        numeric(12,2) not null,
    primary key (account_number),
    foreign key (branch_name) references branch
        on delete cascade
        on update cascade,
    check (balance >= 0)
);
```

`balance` 加 `not null` 是因为 `check (balance >= 0)` 放得过 null。

### 题 11：删除一个系时会发生什么

题目：`course` 表里 `dept_name varchar(20) references department`，现在执行 `delete from department where dept_name = 'Biology'`，而 `course` 里还有生物系的课程。

思路：外码没写级联动作，默认拒绝会导致悬空引用的删除。

解答：删除被拒绝。想让删除成功，要先删掉或修改 `course` 里生物系的课程；或者把外码改成 `references department on delete cascade`，这时生物系的课程会一起被删；写成 `on delete set null` 则这些课程的 `dept_name` 被置空。

### 题 12：两个用户定义类型相加是否合法（判断题）

题目：

```sql
create type Dollars as numeric(12,2) final;
create type RMB as numeric(12,2) final;
```

`department.budget + added_budget.addbudget` 是合法的。

思路：题意是 `budget` 为 `Dollars` 类型、`addbudget` 为 `RMB` 类型。用户定义类型是强类型。

解答：错误（整理补充，原稿无答案）。`Dollars` 和 `RMB` 底层都是 `numeric(12,2)`，但属于不同类型，不能直接相加，要先转换：

```sql
cast(department.budget as numeric(12,2)) + cast(added_budget.addbudget as numeric(12,2))
```

如果把两个都定义成域（`create domain`），直接相加就是合法的。

### 题 13：转账事务

题目：从账户 A-101 转 100 到 A-215，要求不会出现只扣钱不加钱的情况。

思路：两条 update 放进同一个事务，全部成功才提交，中途出错就回滚。

```sql
start transaction;
update account set balance = balance - 100 where account_number = 'A-101';
update account set balance = balance + 100 where account_number = 'A-215';
commit;
```

如果第二条执行失败，执行 `rollback`，第一条的扣款也被撤销。标准 SQL 里也可以把两条语句写进 `begin atomic ... end`。

更多练习见期末复习题整理稿（`20` 号起的文件）和 `32 课后习题 第4章 中级SQL.md`。
