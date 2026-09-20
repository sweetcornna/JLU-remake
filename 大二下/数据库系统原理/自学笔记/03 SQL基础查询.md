# SQL 基础查询

整理自 `笔记/数据库系统原理笔记.pdf` 第 27–43 页（第 3 章 SQL）、第 47–50 页（4.2 数据修改里的删除、插入、更新），`笔记/数据库期末复习题整理.pdf` 第 7–9 页（SQL 部分）、第 104–106 页（SQL 语句整理里的查询和修改）、第 116–125 页（作业题和历年题整理里的 SQL），以及 `老师资料/SQL复习 (1).docx` 2.1–2.16 节。笔记的例子用银行模式，老师资料用大学模式，两边的例子按知识点放在一起。连接、视图、约束见 `04 中级SQL.md`。

## 本章要点

- SQL 由 DDL（建表、改表、删表、声明约束）和 DML（查询、插入、删除、更新）组成，另外还管完整性、视图、事务、嵌入式与动态 SQL、授权。
- `select ... from ... where ...` 的含义：from 里的表先做笛卡儿积，where 筛元组，select 取列。查询结果是**多重集**，默认不去重，去重写 `distinct`。
- `union`、`intersect`、`except` 默认去重，要保留重复写 `union all`、`intersect all`、`except all`。
- 5 个聚集函数：`avg`、`min`、`max`、`sum`、`count`。除 `count(*)` 外都忽略空值；输入为空时 `count` 得 0，其余得 null。
- `where` 在分组前筛元组，里面不能出现聚集函数；`having` 在分组后筛组。select 里没被聚集的属性必须写进 `group by`。
- 任何值和 null 比较都得 unknown，判空只能用 `is null`、`is not null`。
- 嵌套子查询做三件事：集合成员（`in`）、集合比较（`some`、`all`）、空集与重复测试（`exists`、`unique`）。`= some` 等价于 `in`，`<> all` 等价于 `not in`。
- "包含全部"类的查询（关系代数里的除法）用 `not exists (A except B)` 或两层 `not exists` 写。
- from 里的子查询要起别名；`with` 定义的临时关系只在这一条查询里有效；标量子查询只能返回一行一列。
- 删除、更新时带子查询，SQL 先把子查询算完、找齐要改的元组，再统一修改。

## 1. 两套示例模式

### 银行模式（笔记）

| 关系 | 属性 | 主码 |
| --- | --- | --- |
| `branch` | `branch_name`, `branch_city`, `assets` | `branch_name` |
| `customer` | `customer_name`, `customer_street`, `customer_city` | `customer_name` |
| `account` | `account_number`, `branch_name`, `balance` | `account_number` |
| `loan` | `loan_number`, `branch_name`, `amount` | `loan_number` |
| `depositor` | `customer_name`, `account_number` | 两个属性合起来 |
| `borrower` | `customer_name`, `loan_number` | 两个属性合起来 |

`account` 和 `loan` 的 `branch_name` 参照 `branch`；`depositor` 把顾客和存款账户连起来，`borrower` 把顾客和贷款连起来。

### 大学模式（老师资料，属性表按教材整理补充）

| 关系 | 属性 |
| --- | --- |
| `instructor` | `ID`, `name`, `dept_name`, `salary` |
| `department` | `dept_name`, `building`, `budget` |
| `course` | `course_id`, `title`, `dept_name`, `credits` |
| `section` | `course_id`, `sec_id`, `semester`, `year`, `building`, `room_number`, `time_slot_id` |
| `teaches` | `ID`, `course_id`, `sec_id`, `semester`, `year` |
| `student` | `ID`, `name`, `dept_name`, `tot_cred` |
| `takes` | `ID`, `course_id`, `sec_id`, `semester`, `year`, `grade` |
| `prereq` | `course_id`, `prereq_id` |

## 2. SQL 概览

SQL 的标准版本：SQL-86、SQL-89、SQL-92、SQL:1999、SQL:2003、SQL:2008、SQL:2011、SQL:2016。

| 组成部分 | 做什么 |
| --- | --- |
| 数据定义语言 DDL | 建库建表、修改表结构、删表、设置主码和唯一性等约束 |
| 数据操纵语言 DML | 查询、插入、删除、修改元组 |
| 完整性 | 在 DDL 里声明约束，违反约束的更新会被拒绝 |
| 视图定义 | 定义虚关系 |
| 事务控制 | 规定事务从哪开始、到哪结束，要么全做，要么全不做 |
| 嵌入式 SQL、动态 SQL | 在 C、Java 等宿主语言里用 SQL；动态 SQL 在运行时才拼出语句 |
| 授权 | 授予或收回用户对表、视图的访问权限 |

DDL 能描述的东西：

- 每个关系的模式，也就是有哪些属性；
- 每个属性的值域（类型）；
- 完整性约束，分三类：实体完整性（主码保证元组可区分）、参照完整性（外码描述表之间的联系）、用户定义的完整性（具体业务规则）；
- 索引，用来加快查询，相当于书的目录；
- 安全性和权限；
- 物理存储结构，比如堆文件、索引文件、散列文件。

## 3. 数据定义

### 3.1 基本类型

| 类型 | 含义 |
| --- | --- |
| `char(n)` | 定长字符串，长度为 n，如 `char(10)` |
| `varchar(n)` | 变长字符串，最大长度为 n，如 `varchar(255)` |
| `int` / `integer` | 整数，通常 4 字节，范围 -2147483648 到 2147483647 |
| `smallint` | 短整数，通常 2 字节，范围 -32768 到 32767 |
| `numeric(p, d)` | 定点数，共 p 位数字，其中 d 位在小数点后，精确存储。`numeric(5,2)` 能存 `44.50`，最大 `999.99` |
| `real`、`double precision` | 浮点数，近似存储，精度和机器有关 |
| `float(n)` | 精度至少为 n 位的浮点数 |

笔记里写 `char` 最多 8000 个字符、`varchar` 最多 65535 个字符，这是具体数据库的上限（前一个是 SQL Server 的规定，后一个是 MySQL 的规定，MySQL 实际还受单行大小限制），SQL 标准本身不规定。

### 3.2 建表

```sql
create table r (
    A1 D1,
    A2 D2,
    ...,
    An Dn,
    完整性约束1,
    ...,
    完整性约束k
);
```

`r` 是关系名，`Ai` 是属性名，`Di` 是属性的类型。老师资料的例子：

```sql
create table instructor (
    ID        char(5),
    name      varchar(20) not null,
    dept_name varchar(20),
    salary    numeric(8,2),
    primary key (ID),
    foreign key (dept_name) references department
);
```

笔记的例子，`branch` 的主码是 `branch_name`，`branch_city` 不能为空：

```sql
create table branch (
    branch_name char(15),
    branch_city char(30) not null,
    assets      integer,
    primary key (branch_name)
);
```

建表时常用的约束：

- `not null`：这一列不允许空值。
- `primary key (A1, ..., An)`：这些属性构成主码，主码属性自动非空，且任意两个元组在主码上不能相同。
- `foreign key (A) references s`：A 的取值必须在关系 s 的主码里出现过。

约束的细节（`unique`、`check`、级联、断言）在 `04 中级SQL.md`。

### 3.3 删表和改表

```sql
drop table r;          -- 删除表 r：数据和模式定义一起删掉
alter table r add A D; -- 给 r 增加属性 A，类型为 D，已有元组在 A 上取 null
alter table r drop A;  -- 删除属性 A（不少数据库不支持）
```

写建表脚本时，笔记的习惯是先 `drop table` 同名表再 `create`，免得表已存在时报错。

`drop table r` 和 `delete from r` 要分清：前者连表带模式都没了，后者只删元组，空表还在。

### 3.4 增、删、改的基本形式

老师资料 2.3 节的三条语句：

```sql
insert into instructor values ('10211', 'Smith', 'Biology', 66000);  -- 插入一个元组
delete from student;                   -- 删除 student 的全部元组，表还在
update instructor set salary = 70000;  -- 没写 where，所有教师的工资都改成 70000
```

带条件和子查询的写法在第 10 节。

## 4. 查询的基本结构

```sql
select A1, A2, ..., An
from r1, r2, ..., rm
where P;
```

`Ai` 是属性，`ri` 是关系，`P` 是谓词。它和关系代数表达式 $\Pi_{A_1,\dots,A_n}(\sigma_P(r_1 \times r_2 \times \dots \times r_m))$ 的多重集版本严格等价。

按下面的顺序理解执行过程：

1. from：把列出的关系做笛卡儿积。
2. where：用谓词 P 筛掉不满足的元组。
3. select：取出要的属性，算出表达式。

数据库实际执行时会优化，不会真的先生成完整的笛卡儿积，但结果和这个顺序一致。

### 4.1 select 子句

找出 `loan` 里出现的银行名：

```sql
select branch_name
from loan;
```

- 结果默认保留重复（相当于写了 `all`）。一家银行有几笔贷款，名字就出现几次。
- 去重写 `select distinct branch_name from loan`；老师资料的例子是 `select distinct dept_name from instructor`。
- `select *` 表示所有属性。

select 里可以写算术表达式，还可以用 `as` 给结果列起名：

```sql
select loan_number, branch_name, amount * 100
from loan;

select ID, name, salary / 12 as monthly_salary   -- 教师的工号、姓名、月薪
from instructor;
```

### 4.2 where 子句

where 里可以用 `and`、`or`、`not` 和比较运算，`between ... and ...` 包含两端的值。

找出 Perryridge 银行贷款额超过 1200 的贷款号：

```sql
select loan_number
from loan
where branch_name = 'Perryridge' and amount > 1200;
```

（更正：原文 where 写的是 `loan_number = 'Perryridge'`，贷款号不会等于银行名，应比较 `branch_name`。）

```sql
select loan_number
from loan
where amount between 90000 and 100000;   -- 等价于 amount >= 90000 and amount <= 100000

select name
from instructor
where dept_name = 'Comp. Sci.' and salary > 80000;
```

### 4.3 from 子句里放多个关系

把关系代数 $\Pi(borrower \bowtie loan)$（投影出 `customer_name`、`loan_number`、`amount`）写成 SQL：

```sql
select customer_name, borrower.loan_number, amount
from borrower, loan
where borrower.loan_number = loan.loan_number;
```

两个表都有 `loan_number`，引用时要加表名前缀，否则有歧义。

老师资料 2.4 的例子：找出 Art 系教过课的教师姓名和课程号。

```sql
select name, course_id
from instructor, teaches
where instructor.ID = teaches.ID
  and instructor.dept_name = 'Art';
```

漏写 `instructor.ID = teaches.ID` 的话，每个 Art 系教师会和 `teaches` 的每一行配对，结果里多出大量没有意义的配对。

## 5. 附加的基本运算

### 5.1 更名：as

写法是 `旧名 as 新名`，属性和关系都能改名。

例：列出贷款号、顾客名和贷款金额，并把贷款号列改名为 `loan_id`。

```sql
select customer_name, borrower.loan_number as loan_id, amount
from borrower, loan
where borrower.loan_number = loan.loan_number;
```

笔记用的数据：

| `loan_number` | `branch_name` | `amount` |
| --- | --- | --- |
| L-11 | Round Hill | 900 |
| L-14 | Downtown | 1500 |
| L-15 | Perryridge | 1500 |
| L-16 | Perryridge | 1300 |
| L-17 | Downtown | 1000 |
| L-23 | Redwood | 2000 |
| L-93 | Mianus | 500 |

| `customer_name` | `loan_number` |
| --- | --- |
| Adams | L-16 |
| Curry | L-93 |
| Hayes | L-15 |
| Jackson | L-14 |
| Jones | L-17 |
| Smith | L-11 |
| Smith | L-23 |
| Williams | L-17 |

查询结果（不写 `as` 时第二列叫 `loan_number`，写了就叫 `loan_id`）：

| `customer_name` | `loan_id` | `amount` |
| --- | --- | --- |
| Adams | L-16 | 1300 |
| Curry | L-93 | 500 |
| Hayes | L-15 | 1500 |
| Jackson | L-14 | 1500 |
| Jones | L-17 | 1000 |
| Smith | L-11 | 900 |
| Smith | L-23 | 2000 |
| Williams | L-17 | 1000 |

### 5.2 元组变量（表别名）

在 from 里给关系起的别名叫元组变量，可以理解成"指向这个关系某一行的变量"。复习题整理里的说法：更名运算作用在属性上（写在 select 里），元组变量作用在表上（写在 from 里），两者用的都是 `as`。

```sql
select customer_name, T.loan_number, amount
from borrower as T, loan as S
where T.loan_number = S.loan_number;
```

同一个表要和自己比较时，必须用别名把两份区分开。找出资产比 Brooklyn 某家银行高的银行：

```sql
select distinct T.branch_name
from branch as T, branch as S
where T.assets > S.assets and S.branch_city = 'Brooklyn';
```

老师资料的同类题：找出工资比 Comp. Sci. 系某位教师高的教师。

```sql
select distinct T.name
from instructor as T, instructor as S
where T.salary > S.salary and S.dept_name = 'Comp. Sci.';
```

`as` 可以省略，写成 `from branch T, branch S`。Oracle 里表别名前面不能写 `as`。

### 5.3 字符串运算

字符串常量用单引号括起来；串里本身有单引号时写两个单引号，如 `'it''s'`。

`like` 做模式匹配：

| 通配符 | 匹配 |
| --- | --- |
| `%` | 任意子串，包括空串 |
| `_` | 任意一个字符 |

```sql
select customer_name
from customer
where customer_street like '%Main%';   -- 街道名里含 Main

select name
from instructor
where name like '%dar%';               -- 名字里含 dar
```

- `like '_a%'`：第二个字符是 `a`，后面有没有字符都行。
- 不匹配用 `not like`。
- 标准 SQL 里模式匹配区分大小写；MySQL 在默认排序规则下不区分。

要匹配 `%` 或 `_` 本身，用 `escape` 指定转义字符：

```sql
like 'ab\%cd%' escape '\'   -- 匹配以 ab%cd 开头的字符串
```

`\` 后面的 `%` 当普通字符，最后一个 `%` 仍是通配符。（更正：原文作"匹配 ab%cd 的子串"。模式末尾只有一个 `%`，意思是以 `ab%cd` 开头。）

其他字符串运算：

| 运算 | 写法 |
| --- | --- |
| 拼接 | 标准 SQL 用 `\|\|`；MySQL 里 `\|\|` 默认是逻辑或，拼接用 `concat(A, B)`（老师资料用的就是它） |
| 转大写、小写 | `upper(s)`、`lower(s)` |
| 长度 | `length(s)` |
| 截取 | `left(s, n)`、`right(s, n)`、`substring(s, n, len)` |

笔记里用 `||` 把名字和姓拼成一列，换到银行模式上是这样：

```sql
select customer_name || ', ' || customer_city as name_city
from customer;
```

### 5.4 排序：order by

- 默认或写 `asc` 是升序，`desc` 是降序。
- 按多个属性排序时，先按左边的排，左边相同再按右边的排。

按字母序列出 Perryridge 银行的贷款顾客：

```sql
select customer_name
from borrower as T, loan as S
where T.loan_number = S.loan_number and branch_name = 'Perryridge'
order by customer_name;
```

按银行名升序、贷款额降序列出顾客、银行和贷款额：

```sql
select customer_name, branch_name, amount
from borrower as T, loan as S
where T.loan_number = S.loan_number
order by branch_name asc, amount desc;
```

（更正：原文 select 和 order by 里写的是 `account`，`loan` 表里表示贷款额的属性是 `amount`。）

老师资料：按字母序列出所有教师的名字。

```sql
select distinct name
from instructor
order by name;
```

### 5.5 重复：多重集

SQL 的关系是多重集，允许有重复元组。设 r1、r2 是多重集关系：

- r1 中元组 t1 有 c1 个副本，t1 满足选择条件 θ，则 $\sigma_\theta(r_1)$ 中有 c1 个 t1。
- t1 的每个副本在 $\Pi_A(r_1)$ 里都对应一个投影副本。
- r1 中 t1 有 c1 个副本、r2 中 t2 有 c2 个副本，则 $r_1 \times r_2$ 中有 c1 × c2 个 t1t2。

例：r1(A, B) = `{(1,a), (2,a)}`，r2(C) = `{(2), (3), (3)}`。

- $\Pi_B(r_1)$ = `{(a), (a)}`
- $\Pi_B(r_1) \times r_2$ = `{(a,2), (a,2), (a,3), (a,3), (a,3), (a,3)}`

## 6. 集合运算

| 运算 | 去重版 | 保留重复版：元组在 r 中出现 m 次、在 s 中出现 n 次时 |
| --- | --- | --- |
| 并 | `union` | `union all`：出现 m + n 次 |
| 交 | `intersect` | `intersect all`：出现 min(m, n) 次 |
| 差 | `except` | `except all`：出现 max(0, m − n) 次 |

银行模式的例子：

```sql
-- 有存款、或有贷款、或两者都有的顾客
(select customer_name from depositor)
union
(select customer_name from borrower);

-- 同上，不去重
(select customer_name from depositor)
union all
(select customer_name from borrower);

-- 既有存款又有贷款的顾客
(select customer_name from depositor)
intersect
(select customer_name from borrower);

-- 有存款但没有贷款的顾客
(select customer_name from depositor)
except
(select customer_name from borrower);
```

老师资料 2.7 的例子，换成大学模式：

```sql
-- 2009 年秋季或 2010 年春季开过的课
(select course_id from section where semester = 'Fall' and year = 2009)
union
(select course_id from section where semester = 'Spring' and year = 2010);

-- 两个学期都开过的课：把 union 换成 intersect
-- 2009 年秋季开过、2010 年春季没开的课：把 union 换成 except
```

（更正：原图里学期属性写作 `sem`，`section` 表的属性名是 `semester`。）

MySQL 从 8.0.31 才支持 `intersect` 和 `except`，更老的版本要用 `in`、`not in` 或 `exists` 改写。

## 7. 聚集函数与分组

### 7.1 五个聚集函数

| 函数 | 作用 |
| --- | --- |
| `avg` | 平均值 |
| `min` | 最小值 |
| `max` | 最大值 |
| `sum` | 总和 |
| `count` | 计数 |

`avg` 和 `sum` 的输入必须是数值；其余三个也能用于字符串等非数值类型。

```sql
-- Perryridge 银行存款账户的平均余额
select avg(balance)
from account
where branch_name = 'Perryridge';

-- customer 表有多少个元组
select count(*)
from customer;

-- 2010 年春季学期上过课的教师人数
select count(distinct ID)
from teaches
where semester = 'Spring' and year = 2010;

-- course 表有多少个元组
select count(*)
from course;
```

一个教师同一学期可能教好几个课段，所以要 `count(distinct ID)`。`count(*)` 不能和 `distinct` 连用。

### 7.2 group by

`group by` 把在分组属性上取值相同的元组分成一组，聚集函数对每组各算一次。

```sql
-- 每个银行存款账户的平均余额
select branch_name, avg(balance)
from account
group by branch_name;

-- 每个银行的储户人数
select branch_name, count(distinct customer_name)
from depositor as R, account as S
where R.account_number = S.account_number
group by branch_name;

-- 每个系教师的平均工资
select dept_name, avg(salary) as avg_salary
from instructor
group by dept_name;
```

同一顾客在一家银行可能有多个账户，统计人数要 `distinct`。

规则：select 里出现、又没被聚集函数包住的属性，必须出现在 `group by` 里。下面这条是错的：

```sql
select dept_name, ID, avg(salary)
from instructor
group by dept_name;     -- 错误：一个系有很多 ID，不知道输出哪个
```

having 也一样：having 里没被聚集的属性必须出现在 `group by` 里。

### 7.3 having

`having` 是分组之后对组的筛选条件，里面可以用聚集函数。

```sql
-- 平均余额超过 1200 的银行及其平均余额
select branch_name, avg(balance)
from account
group by branch_name
having avg(balance) > 1200;
```

老师资料 2.10：列出教师平均工资超过 42000 的系和它们的平均工资。

```sql
select dept_name, avg(salary)
from instructor
group by dept_name
having avg(salary) > 42000;
```

（更正：原文题干写"平均年薪大于 40000"，SQL 写的是 `> 42000`，两处不一致。教材原例是 42000，这里题干按 42000 理解。）

where 和 having 同时出现的例子：列出住在 Harrison、至少有三个存款账户的顾客的平均余额。

```sql
select R.customer_name, avg(balance)
from depositor as R, account as S, customer as T
where R.account_number = S.account_number
  and R.customer_name = T.customer_name
  and T.customer_city = 'Harrison'              -- 分组前：只留住在 Harrison 的
group by R.customer_name
having count(distinct R.account_number) >= 3;   -- 分组后：只留账户数不少于 3 的组
```

三个表连接后，每个元组仍带着 `account_number`，所以分组后能数每个顾客的账户数。

（更正：笔记第 37 页和复习题整理第 121 页的这两例里，`select branch name` 缺下划线，`'Harrison’` 前后引号不配对。）

一条查询里各子句的作用顺序：from → where → group by → having → select → order by。where 在分组前起作用，不能写聚集函数。

## 8. 空值

空值表示"不存在"或"不知道"。

### 8.1 运算规则

- 算术运算（加减乘除）只要有一个操作数是 null，结果就是 null。
- 比较运算只要涉及 null，结果就是 unknown，包括 `null = null`。
- 逻辑运算按三值逻辑算：

| 表达式 | 结果 |
| --- | --- |
| `true and unknown` | unknown |
| `false and unknown` | false |
| `unknown and unknown` | unknown |
| `true or unknown` | true |
| `false or unknown` | unknown |
| `unknown or unknown` | unknown |
| `not unknown` | unknown |

- where 条件的结果是 unknown 时按 false 处理，元组不进结果。
- `distinct`、`group by` 和集合运算判断重复时，两个 null 算相同，`group by` 会把 null 分成一组。

### 8.2 判断

```sql
select loan_number
from loan
where amount is null;
```

（更正：原文作 `account is null`，`loan` 表没有 `account` 属性，应为 `amount`。）

- 判空只能用 `is null`、`is not null`。写 `amount = null` 永远选不出东西。
- 判断比较结果是否为 unknown 用 `is unknown`、`is not unknown`。
- `select name from instructor where salary <> 70000` 选不出工资为 null 的教师，因为 `null <> 70000` 是 unknown。

### 8.3 聚集函数和空值

老师资料 2.11 用 `instructor` 的 `salary` 说明：

| 写法 | 处理 null 的方式 | 全是 null 或没有元组时 |
| --- | --- | --- |
| `sum(salary)` | 忽略 null，只加非空值 | null |
| `avg(salary)`、`max(salary)`、`min(salary)` | 忽略 null | null |
| `count(salary)` | 只数非空值 | 0 |
| `count(*)` | 数元组个数，null 也算 | 0（没有元组时） |

## 9. 嵌套子查询

老师资料 2.12 把能嵌套的位置归成三处：

- select 里的属性可以换成只返回一个值的子查询（标量子查询）；
- from 里的关系可以换成任意合法的子查询；
- where 里的条件可以写成 `B <运算> (子查询)` 的形式，B 是属性。

老师资料 2.13 说 where 里的子查询处理三类问题：集合成员判断、集合比较、集合基数测试（是否为空、有无重复）。复习题整理按这个思路列了四种：`in`/`not in`、`> all`/`> some` 这类集合比较、`exists`、`unique`/`not unique`。

### 9.1 不相关子查询和相关子查询

复习题整理用一张 `Books` 表讲了两种子查询的执行过程：

| 类编号 | 图书名 | 出版社 | 价格 |
| --- | --- | --- | --- |
| 2 | c#高级应用 | 圣通出版 | 23.00 |
| 2 | Jsp开发应用 | 机械出版社 | 45.00 |
| 3 | 高等数学 | 济南出版社 | 25.00 |
| 3 | 疯狂英语 | 清华大学出版社 | 32.00 |

不相关子查询（原稿叫嵌套子查询）不依赖外层查询：先把子查询执行一次，结果不显示，交给外层当条件用；再执行外层查询。按子查询返回什么，又分返回单个值和返回一列值两种。

```sql
-- 返回单个值：价格高于全部图书平均价格的图书
select 图书名, 出版社, 价格
from Books
where 价格 > (select avg(价格)
              from Books);

-- 返回一列值：借过书的读者（Readers、借阅记录表只是示意）
select *
from Readers
where 读者编号 in (select 读者编号
                   from 借阅记录);
```

（更正：原稿第一条的 select 里还有"作者"，`Books` 表没有这一列，这里去掉。原稿里 `USE tempdb`、`GO` 和方括号表名是 SQL Server 的写法，与题意无关，也一并去掉。）

相关子查询的 where 里引用了外层查询的表，没法单独执行。执行过程：

1. 外层取出一个元组，把相关列的值传给子查询；
2. 执行子查询，得到结果；
3. 外层根据子查询结果判断这个元组要不要；
4. 外层取下一个元组，重复 1 到 3，直到外层元组处理完。

例：找出价格高于同类图书平均价格的图书。

```sql
select 图书名, 出版社, 类编号, 价格
from Books as a
where 价格 > (select avg(价格)
              from Books as b
              where a.类编号 = b.类编号);
```

（更正：原稿写成 `SElECT 图书名,出版社,类编号,价格 SELECT FROM Books As a`，多了一个 `SELECT`。）

外层取第一行时类编号是 2，子查询变成求 2 类图书的平均价格 (23 + 45) / 2 = 34，外层条件变成 `价格 > 34`，第一行 23 不满足。逐行做下去，3 类的平均价格是 28.5，最后结果是"Jsp开发应用"（45）和"疯狂英语"（32）。

### 9.2 集合成员：in、not in

```sql
-- 既有存款又有贷款的顾客
select distinct customer_name
from borrower
where customer_name in (select customer_name
                        from depositor);

-- 有贷款但没有存款的顾客
select distinct customer_name
from borrower
where customer_name not in (select customer_name
                            from depositor);

-- 有贷款、名字既不是 Smith 也不是 Jones 的顾客：in 后面也可以是常量集合
select distinct customer_name
from borrower
where customer_name not in ('Smith', 'Jones');
```

子查询里不用写 `distinct`，in 只关心在不在，加了反而多一次去重。

in 左边可以是元组。列出在 Perryridge 银行既有存款又有贷款的顾客：

```sql
select distinct customer_name
from borrower, loan
where borrower.loan_number = loan.loan_number
  and branch_name = 'Perryridge'
  and (branch_name, customer_name) in
      (select branch_name, customer_name
       from depositor, account
       where depositor.account_number = account.account_number);
```

子查询先求出"在哪家银行有存款的哪位顾客"这些二元组，外层再挑出在 Perryridge 有贷款、且二元组落在其中的顾客。

老师资料的例子：2009 年秋季和 2010 年春季都开过的课。

```sql
select distinct course_id
from section
where semester = 'Fall' and year = 2009
  and course_id in (select course_id
                    from section
                    where semester = 'Spring' and year = 2010);
```

`not in` 有个坑：子查询结果里只要有一个 null，`x not in (...)` 就不可能为 true，整个查询一条也选不出。

### 9.3 集合比较：some、all

| 形式 | 含义 |
| --- | --- |
| `<some`、`<=some`、`>some`、`>=some`、`=some`、`<>some` | 子查询结果里**某个**值满足比较就为真 |
| `<all`、`<=all`、`>all`、`>=all`、`=all`、`<>all` | 子查询结果里**所有**值都满足比较才为真 |

找出资产比 Brooklyn 某家银行高的银行，除了 5.2 节的自连接写法，还能用 some：

```sql
select branch_name
from branch
where assets > some (select assets
                     from branch
                     where branch_city = 'Brooklyn');
```

找出资产比 Brooklyn 每一家银行都高的银行：

```sql
select branch_name
from branch
where assets > all (select assets
                    from branch
                    where branch_city = 'Brooklyn');
```

老师资料：找出工资比生物系所有教师都高的教师。

```sql
select name
from instructor
where salary > all (select salary
                    from instructor
                    where dept_name = 'Biology');
```

找出平均存款额最高的银行。SQL 不允许 `max(avg(balance))` 这种聚集套聚集，用 `>= all` 绕开：

```sql
select branch_name
from account
group by branch_name
having avg(balance) >= all (select avg(balance)
                            from account
                            group by branch_name);
```

几条等价关系：

- `= some` 等价于 `in`；
- `<> all` 等价于 `not in`；
- `<> some` 和 `not in` 不等价：只要集合里有一个值和它不同就为真；
- `= all` 和 `in` 不等价：要和集合里每个值都相等。
- 子查询结果为空时，`all` 比较为真，`some` 比较为假。
- `any` 是 `some` 的同义词。

### 9.4 空集测试：exists、not exists

- `exists r` 为真，当且仅当 r 非空。
- `not exists r` 为真，当且仅当 r 为空。

```sql
-- 既有存款又有贷款的顾客
select distinct customer_name
from borrower as R
where exists (select *
              from depositor as S
              where S.customer_name = R.customer_name);
```

子查询用到了外层的 `R`，是**相关子查询**（执行过程见 9.1）：外层每取一个元组，子查询就按这个元组重新算一次，结果非空就把外层元组放进结果。

除法查询的思路：集合 X 包含集合 Y，等价于 Y − X 为空。

找出在 Brooklyn 每家银行都有存款的顾客：

```sql
select distinct S.customer_name
from depositor as S
where not exists (
    (select branch_name                  -- Brooklyn 的所有银行
     from branch
     where branch_city = 'Brooklyn')
    except
    (select R.branch_name                -- 这位顾客有存款的银行
     from depositor as T, account as R
     where T.account_number = R.account_number
       and S.customer_name = T.customer_name));
```

"Brooklyn 的银行"减去"这位顾客存过款的银行"为空，就说明 Brooklyn 每家银行他都存过。

老师资料的同类题：找出选了生物系开设的全部课程的学生。

```sql
select distinct S.ID, S.name
from student as S
where not exists (
    (select course_id
     from course
     where dept_name = 'Biology')
    except
    (select T.course_id
     from takes as T
     where S.ID = T.ID));
```

### 9.5 重复测试：unique

子查询结果里没有重复元组时，`unique` 返回 true；`not unique` 相反。

```sql
-- 在 Perryridge 银行至多有一个存款账户的顾客
select T.customer_name
from depositor as T
where unique (select R.customer_name
              from account as S, depositor as R
              where T.customer_name = R.customer_name
                and R.account_number = S.account_number
                and S.branch_name = 'Perryridge');
```

（更正：原文题目作"只有一个存款账户"。子查询为空时 `unique` 也返回 true，在 Perryridge 没有账户的顾客同样会被选出，所以含义是"至多一个"。要"恰好一个"，再加一个 `exists` 条件。）

```sql
-- 在 Perryridge 银行有多于一个存款账户的顾客
select T.customer_name
from depositor as T
where not unique (select R.customer_name
                  from account as S, depositor as R
                  where T.customer_name = R.customer_name
                    and R.account_number = S.account_number
                    and S.branch_name = 'Perryridge');
```

老师资料：找出 2009 年至多开过一次的课程。

```sql
select T.course_id
from course as T
where unique (select R.course_id
              from section as R
              where T.course_id = R.course_id
                and R.year = 2009);
```

MySQL、PostgreSQL、Oracle、SQL Server 都不支持 `unique (子查询)` 这个谓词，上机时改用计数：

```sql
select T.course_id
from course as T
where 1 >= (select count(R.course_id)
            from section as R
            where T.course_id = R.course_id
              and R.year = 2009);
```

## 10. 复杂查询

### 10.1 from 子句中的子查询（派生关系）

在各银行的存款总额里找出最大的那个：

```sql
select max(tot_balance)
from (select branch_name, sum(balance)
      from account
      group by branch_name) as branch_total (branch_name, tot_balance);
```

执行过程：

1. 子查询按 `branch_name` 分组，算出每家银行的存款总额。
2. `as branch_total (branch_name, tot_balance)` 把这个结果临时叫作 `branch_total`，两列分别叫 `branch_name`、`tot_balance`。
3. 外层对 `tot_balance` 取最大值。

找出平均存款额超过 1200 的银行及其平均存款额，这回不用 having：

```sql
select branch_name, avg_balance
from (select branch_name, avg(balance)
      from account
      group by branch_name) as branch_avg (branch_name, avg_balance)
where avg_balance > 1200;
```

`branch_avg` 只在这一条语句里有效。复习题整理里拿它和 having 对比：from 里的子查询已经完成了分组和求平均，外层直接用 where 筛选，就不用再写 having。

老师资料 2.14，同一道"平均工资超过 42000 的系"：

```sql
select dept_name, avg_salary
from (select dept_name, avg(salary) as avg_salary
      from instructor
      group by dept_name) as dept_avg
where avg_salary > 42000;
```

（更正：原文 from 里的子查询没有别名。标准 SQL 要求派生表有名字，MySQL 缺别名会报错，这里补上 `as dept_avg`。）

笔记里还提到两点：

- 很多数据库支持 from 里嵌子查询，有些实现要求每个子查询结果都起名字，即使后面用不到这个名字。
- from 里的子查询不能直接用同一个 from 里其他关系的属性。SQL:2003 允许在子查询前加 `lateral`，这样子查询就能引用 from 里排在它前面的关系。MySQL 8.0.14 起支持。

```sql
-- 每位教师的姓名、工资和所在系的平均工资
select name, salary, avg_salary
from instructor as I1,
     lateral (select avg(salary) as avg_salary
              from instructor as I2
              where I2.dept_name = I1.dept_name) as dept_avg;
```

去掉 `lateral`，子查询里的 `I1` 就无法识别。

### 10.2 with 子句

`with` 定义一个临时关系（也叫公共表表达式 CTE），只在紧跟着的这条查询里有效，查询结束就没了。能用 from 或 where 里的子查询写的查询，改用 with 通常更好读，而且一个临时关系可以在同一条查询里多处使用。

找出存款余额最大的账户：

```sql
with max_balance (value) as (
    select max(balance)
    from account
)
select account_number
from account, max_balance
where account.balance = max_balance.value;
```

（更正：原文 `as` 后面的子查询没加括号，where 里的 `account` 拼成了 `accout`。）

找出存款总额大于"各银行存款总额平均值"的银行：

```sql
with branch_total (branch_name, value) as (
    select branch_name, sum(balance)
    from account
    group by branch_name
),
branch_total_avg (value) as (
    select avg(value)
    from branch_total
)
select branch_name
from branch_total, branch_total_avg
where branch_total.value > branch_total_avg.value;
```

（更正：原文写了两个 `with`。一条查询定义多个临时关系时只写一次 `with`，各定义之间用逗号隔开，每个子查询都要加括号。）

老师资料 2.15：找出最大的系预算。

```sql
with max_budget (value) as (
    select max(budget)
    from department
)
select budget
from department, max_budget
where department.budget = max_budget.value;
```

（更正：原文作 `select department.name`，`department` 表的属性只有 `dept_name`、`building`、`budget`，没有 `name`。题目要最大预算，选 `budget`；想同时知道是哪个系，写 `select dept_name, budget`。）

with 和视图的区别见 `04 中级SQL.md` 的易混对比。

### 10.3 标量子查询

只返回一行一列的子查询叫标量子查询，可以出现在 select、where、having 等任何需要一个值的地方。老师资料 2.16：统计每个系的教师人数。

```sql
select dept_name,
       (select count(*)
        from instructor
        where department.dept_name = instructor.dept_name) as num_instructors
from department;
```

- 子查询对 `department` 的每个元组算一次；没有教师的系得 0。
- 子查询返回多于一行时，运行出错；返回空时，值为 null。
- where 里也常用：`where balance > (select avg(balance) from account)`。

## 11. 数据库修改

每条 `delete`、`insert`、`update` 语句只修改一个关系。

### 11.1 删除

```sql
delete from r
where P;
```

只能删整个元组，不能只删某几个属性的值。不写 where 就删掉全部元组。

```sql
-- 删除 Perryridge 银行的全部存款账户
delete from account
where branch_name = 'Perryridge';

-- 删除贷款额在 1300 到 1500 之间的贷款
delete from loan
where amount between 1300 and 1500;

-- 删除位于 Brooklyn 的所有银行的存款账户
delete from account
where branch_name in (select branch_name
                      from branch
                      where branch_city = 'Brooklyn');

-- 删除余额低于平均余额的存款账户
delete from account
where balance < (select avg(balance)
                 from account);
```

（更正：第二条原文作 `where account between 1300 and 1500`，`loan` 表没有 `account` 属性，应为 `amount`；复习题整理第 104 页的第三条还把 `branch_name` 写成了 `branch name`。）

最后一条会不会"边删边变平均值"？不会。SQL 先算出平均值，把所有要删的元组都找出来，然后统一删除，删除过程中不再重算平均值。否则删除结果会随元组的处理顺序变化。

上机时注意：MySQL 不允许在 delete、update 的子查询里直接读正在修改的表（报错 1093），要把子查询再包一层派生表：

```sql
delete from account
where balance < (select avg_bal
                 from (select avg(balance) as avg_bal from account) as t);
```

### 11.2 插入

插入一个元组。把 Perryridge 银行账户 A-9732、余额 1200 存入数据库：

```sql
insert into account
values ('A-9732', 'Perryridge', 1200);
```

这种写法的值要按建表时属性的顺序给。也可以先列出属性名，值按列出的顺序给：

```sql
insert into account (account_number, branch_name, balance)
values ('A-9732', 'Perryridge', 1200);
```

列属性名的写法可以只插部分属性，没给值的属性取默认值，没有默认值就取 null。

插入一批元组。Perryridge 银行送每位贷款客户 200 美元，存进一个账号和贷款号相同的存款账户：

```sql
insert into account
    select loan_number, branch_name, 200
    from loan
    where branch_name = 'Perryridge';

insert into depositor
    select customer_name, R.loan_number
    from borrower as R, loan as S
    where R.loan_number = S.loan_number
      and branch_name = 'Perryridge';
```

新开存款账户要同时改 `account` 和 `depositor` 两个表。第一条给 Perryridge 的每笔贷款建一个余额 200 的账户；第二条让对应的贷款客户成为这个账户的存款人。

（更正：原文第二条 select 写的是 `loan_number`，`borrower` 和 `loan` 都有这个属性，会有歧义，这里加上前缀 `R.`。）

`insert ... select` 同样是先把 select 全部算完再插入。所以 `insert into r select * from r` 只会把现有元组复制一遍，不会无限插下去。

### 11.3 更新

```sql
-- 所有账户按 5% 计息
update account
set balance = balance * 1.05;

-- 余额超过 10000 的账户按 5% 计息
update account
set balance = balance * 1.05
where balance > 10000;

-- 余额高于平均余额的账户按 5% 计息
update account
set balance = balance * 1.05
where balance > (select avg(balance)
                 from account);
```

余额超过 10000 的按 6% 计息，其余按 5% 计息。用两条语句时要注意顺序：

```sql
update account
set balance = balance * 1.06
where balance > 10000;

update account
set balance = balance * 1.05
where balance <= 10000;
```

两条反过来写就错了：余额 9800 的账户先涨 5% 变成 10290，第二条又给它加一次 6%。

用 `case` 一条语句搞定，每个账户只判断一次，没有顺序问题：

```sql
update account
set balance = case
                  when balance <= 10000 then balance * 1.05
                  else balance * 1.06
              end;
```

`case` 的一般形式：

```sql
case
    when pred1 then result1
    when pred2 then result2
    ...
    when predn then resultn
    else result0
end
```

（更正：原文最后一行作 `else pred0`，else 后面跟的是结果值，应为 `result0`。）

课程 `README.md` 回忆的往年简答题里有"如何实现多表更新"。按本节的规则，一条 `update` 只改一个表，要同时改几张表，就把几条语句放进同一个事务（见 `04 中级SQL.md` 的事务一节），保证要么都改、要么都不改。MySQL 另外提供多表 `update` 语法，如 `update t1, t2 set t1.a = t2.a where t1.id = t2.id`，属于 MySQL 扩展。

## 易混对比

| 对比项 | 区别 |
| --- | --- |
| `where` / `having` | where 在分组前筛元组，不能用聚集函数；having 在分组后筛组，可以用聚集函数 |
| `count(*)` / `count(列)` / `count(distinct 列)` | 数元组个数，null 也算 / 数非空值 / 数不同的非空值 |
| `union` / `union all` | 前者去重；后者保留重复，出现次数相加 |
| `in` / `exists` | in 看左边的值在不在子查询结果里；exists 只看子查询结果是否为空，常配合相关子查询 |
| `not in` / `not exists` | 子查询结果含 null 时 `not in` 永远不为真；`not exists` 不受影响 |
| `= some` / `in` | 等价 |
| `<> all` / `not in` | 等价 |
| `<> some` / `not in` | 不等价：`<> some` 只要有一个值不同就为真 |
| `some` / `all` 遇到空集 | some 为假，all 为真 |
| `unique` / `distinct` | unique 是谓词，测试子查询结果有没有重复；distinct 用在 select 里，去掉结果中的重复 |
| `delete from r` / `drop table r` | 前者只删元组，表还在；后者连表的定义一起删 |
| `char(n)` / `varchar(n)` | 定长 / 变长 |
| `= null` / `is null` | 前者结果总是 unknown，选不出东西；判空只能用后者 |
| from 子查询 / `with` | 效果相同；with 把子查询提到前面起名字，可读性好，同一条查询里能多处引用 |
| `%` / `_` | 任意子串（含空串）/ 恰好一个字符 |

## 典型题与解答

### 题 1：工资比 Comp. Sci. 系某位教师高的教师

思路：拿每位教师和计算机系的教师逐个比，有一个比得过就算。可以自连接，也可以用 `> some`。

```sql
select distinct T.name
from instructor as T, instructor as S
where T.salary > S.salary and S.dept_name = 'Comp. Sci.';

select name
from instructor
where salary > some (select salary
                     from instructor
                     where dept_name = 'Comp. Sci.');
```

自连接写法里一位教师可能比好几位计算机系教师工资高，会重复出现，要 `distinct`。

### 题 2：不用聚集函数求最高工资

思路：最高工资就是"没有比它更高的工资"。老师资料用差运算，先求出所有"存在更高工资"的值，再从全部工资里减掉。

```sql
(select distinct salary from instructor)
except
(select distinct T.salary
 from instructor as T, instructor as S
 where T.salary < S.salary);
```

另外两种写法（整理补充）：

```sql
select distinct salary
from instructor as T
where not exists (select *
                  from instructor as S
                  where S.salary > T.salary);

select distinct salary
from instructor
where salary >= all (select salary
                     from instructor
                     where salary is not null);
```

`salary` 有空值时要小心：差运算和 `not exists` 两种写法的结果里会多出一个 null（null 和别人比较都是 unknown，减不掉，也找不到比它大的）；`>= all` 写法的子查询如果不排除 null，比较结果全是 unknown，一行也选不出来。

### 题 3：平均工资超过 42000 的系

思路：先按系分组算平均值，再筛组。筛组条件里有聚集函数，只能放 having，或者先在 from 里算好再用 where。

```sql
select dept_name, avg(salary) as avg_salary
from instructor
group by dept_name
having avg(salary) > 42000;

select dept_name, avg_salary
from (select dept_name, avg(salary) as avg_salary
      from instructor
      group by dept_name) as dept_avg
where avg_salary > 42000;
```

常见错误是写成 `where avg(salary) > 42000`，where 里不允许聚集函数。

### 题 4：平均存款额最高的银行

思路：聚集函数不能嵌套，`max(avg(balance))` 不合法。要么用 `>= all` 和每家银行的平均值比，要么先用 with 算出各银行平均值。

```sql
select branch_name
from account
group by branch_name
having avg(balance) >= all (select avg(balance)
                            from account
                            group by branch_name);

with branch_avg (branch_name, avg_balance) as (
    select branch_name, avg(balance)
    from account
    group by branch_name
)
select branch_name
from branch_avg
where avg_balance = (select max(avg_balance) from branch_avg);
```

### 题 5：2009 年秋季和 2010 年春季都开过的课

思路：两个学期的课程号取交集。三种写法都要会。

```sql
-- 写法一：intersect
(select course_id from section where semester = 'Fall' and year = 2009)
intersect
(select course_id from section where semester = 'Spring' and year = 2010);

-- 写法二：in
select distinct course_id
from section
where semester = 'Fall' and year = 2009
  and course_id in (select course_id
                    from section
                    where semester = 'Spring' and year = 2010);

-- 写法三：exists（相关子查询）
select distinct S.course_id
from section as S
where S.semester = 'Fall' and S.year = 2009
  and exists (select *
              from section as T
              where T.semester = 'Spring' and T.year = 2010
                and S.course_id = T.course_id);
```

### 题 6：选了生物系全部课程的学生

思路：除法。"生物系的课"减去"这个学生选过的课"为空，他就全选了。也可以写成两层 `not exists`：不存在一门生物系的课，这个学生没选过。

```sql
-- 写法一：not exists + except（老师资料）
select distinct S.ID, S.name
from student as S
where not exists (
    (select course_id
     from course
     where dept_name = 'Biology')
    except
    (select T.course_id
     from takes as T
     where S.ID = T.ID));

-- 写法二：两层 not exists（整理补充，不依赖 except）
select S.ID, S.name
from student as S
where not exists (
    select *
    from course as C
    where C.dept_name = 'Biology'
      and not exists (
          select *
          from takes as T
          where T.ID = S.ID
            and T.course_id = C.course_id));
```

生物系一门课都没开时，两种写法都会选出全部学生（空集被任何集合包含）。

### 题 7：2009 年至多开过一次的课程

思路：对每门课，看 2009 年它的课段有几个。`unique` 测的是"没有重复"，空集也算，所以含义是"至多一次"。

```sql
select T.course_id
from course as T
where unique (select R.course_id
              from section as R
              where T.course_id = R.course_id
                and R.year = 2009);

-- 数据库不支持 unique 谓词时
select T.course_id
from course as T
where 1 >= (select count(R.course_id)
            from section as R
            where T.course_id = R.course_id
              and R.year = 2009);
```

要"恰好开过一次"，把 `1 >=` 改成 `1 =`。

复习题整理的作业答案给的是分组写法：

```sql
select course_id
from section
where year = 2009
group by course_id
having count(*) <= 1;
```

它和上面两种写法不等价。分组写法从 `section` 出发，只能看到 2009 年开过的课，2009 年一次都没开的课根本不会出现；`unique` 和计数子查询从 `course` 出发，这些课也算"至多一次"，会被选出来。按"至多一次"的字面意思，应以 `unique` 写法为准。

### 题 8：每个系的教师人数

思路：用标量子查询，对 `department` 的每个系数一次教师。

```sql
select dept_name,
       (select count(*)
        from instructor
        where department.dept_name = instructor.dept_name) as num_instructors
from department;
```

对比 `select dept_name, count(*) from instructor group by dept_name`：分组写法只能统计 `instructor` 里出现过的系，没有教师的系不会出现。标量子查询写法从 `department` 出发，这些系显示 0。用外连接也能做到，见 `04 中级SQL.md`。

### 题 9：住在 Harrison、至少有三个存款账户的顾客的平均余额

思路：城市条件和单个元组有关，放 where；"至少三个账户"是对整组的要求，放 having。

```sql
select R.customer_name, avg(balance)
from depositor as R, account as S, customer as T
where R.account_number = S.account_number
  and R.customer_name = T.customer_name
  and T.customer_city = 'Harrison'
group by R.customer_name
having count(distinct R.account_number) >= 3;
```

### 题 10：最大的系预算是哪个系的

思路：先用 with 求出最大预算，再和 `department` 连接找到对应的系。

```sql
with max_budget (value) as (
    select max(budget)
    from department
)
select dept_name, budget
from department, max_budget
where department.budget = max_budget.value;
```

### 题 11：分档计息

题目：余额不超过 10000 的账户按 5% 计息，其余按 6% 计息。

思路：两条 update 时先处理高余额的，或者用 `case` 在一条语句里分情况。

```sql
update account
set balance = case
                  when balance <= 10000 then balance * 1.05
                  else balance * 1.06
              end;
```

先执行 `<= 10000` 那条会让一部分账户涨过 10000 后再被加一次 6%，这是本题的考点。

### 题 12：删除余额低于平均余额的账户

思路：where 里用标量子查询求平均值。

```sql
delete from account
where balance < (select avg(balance)
                 from account);
```

平均值在删除开始前算好，删除过程中不变；所有要删的元组先确定下来，再一起删。

### 题 13：salary 有空值时 min 和 count 输出什么（作业题）

题目：`instructor` 的 `salary` 列有空值，`select min(salary) from instructor` 和 `select count(salary) from instructor` 分别输出什么？

思路：除 `count(*)` 外的聚集函数都先扔掉 null 再算。

解答：

- `min(salary)` 输出非空工资里的最小值；`salary` 全是 null 时输出 null。
- `count(salary)` 输出非空工资的个数；全是 null 时输出 0。
- 对照：`count(*)` 数的是元组个数，null 照样算进去。

### 题 14：用嵌套子查询找出比生物系所有教师工资都高的教师（作业题）

思路：和生物系每一位教师的工资比，全部比过才算，用 `> all`。

```sql
select name
from instructor
where salary > all (select salary
                    from instructor
                    where dept_name = 'Biology');
```

（更正：复习题整理第 117 页的答案写成 `from teacher`、`where department = 'Biology'`，这不是大学模式里的名字，应为 `instructor` 表和 `dept_name` 属性。老师资料原文的 `dept name` 也缺下划线。）

生物系没有教师时子查询为空，`> all` 为真，所有教师都会被选出。生物系有教师工资为 null 时，和 null 的比较是 unknown，结果一个也选不出。

### 历年题整理：银行模式 SQL 题（题 15 到题 20）

复习题整理第 117–118 页的历年题用了一个略有不同的银行模式，顾客用 `customer_id` 标识（原稿拼写错误已改正，主码按原稿下划线标出）：

| 关系 | 属性 | 主码 |
| --- | --- | --- |
| `branch` | `branch_name`, `branch_city`, `assets` | `branch_name` |
| `customer` | `customer_id`, `customer_name`, `customer_street`, `customer_city` | `customer_id` |
| `loan` | `loan_number`, `amount`, `branch_name` | `loan_number` |
| `account` | `account_number`, `balance`, `branch_name` | `account_number` |
| `borrower` | `customer_id`, `loan_number` | 两个属性合起来 |
| `depositor` | `customer_id`, `account_number` | 两个属性合起来 |

原题要求用关系代数和 SQL 表达，原稿没有给答案。下面只给 SQL（整理补充），关系代数写法和更多说明见 `25 SQL与关系代数应用题.md`。第 7 题定义视图，放在 `04 中级SQL.md` 的典型题里。

#### 题 15：在 Austin 银行有存款的客户的姓名、存款号和存款额

思路：`customer`、`depositor`、`account` 三表连接，再按支行名筛选。题目里"Austin 银行"按支行名理解；若指城市，就再连 `branch`，条件换成 `branch_city = 'Austin'`。

```sql
select C.customer_name, A.account_number, A.balance
from customer as C, depositor as D, account as A
where C.customer_id = D.customer_id
  and D.account_number = A.account_number
  and A.branch_name = 'Austin';
```

#### 题 16：资产至少比位于 Rye 的某一家支行高的支行名

思路："比某一家高"就是 `> some`，也可以自连接。

```sql
select branch_name
from branch
where assets > some (select assets
                     from branch
                     where branch_city = 'Rye');
```

#### 题 17：有贷款但无存款的客户姓名

思路：用 `customer_id` 判断，避免重名的客户被混在一起。

```sql
select customer_name
from customer
where customer_id in (select customer_id from borrower)
  and customer_id not in (select customer_id from depositor);
```

#### 题 18：每个支行的贷款户数

思路：`loan` 和 `borrower` 连接后按支行分组。一个客户在同一支行可能有多笔贷款，户数要 `count(distinct customer_id)`。

```sql
select L.branch_name, count(distinct B.customer_id) as borrower_count
from loan as L, borrower as B
where L.loan_number = B.loan_number
group by L.branch_name;
```

没有贷款的支行不会出现在结果里。要显示 0，改用外连接，写法同 `04 中级SQL.md` 里"每个系的教师人数"那题。

#### 题 19：所有存款余额增加 7%

```sql
update account
set balance = balance * 1.07;
```

#### 题 20：删除余额低于银行平均余额的账户记录

思路：同题 12。`depositor` 的 `account_number` 若声明为参照 `account` 的外码，直接删 `account` 会被拒绝，要先删 `depositor` 里对应的记录，或者外码带 `on delete cascade`。

```sql
delete from depositor
where account_number in (select account_number
                         from account
                         where balance < (select avg(balance) from account));

delete from account
where balance < (select avg(balance)
                 from account);
```

第一条执行完 `account` 没变，所以第二条算出的平均值和第一条一样。

### 历年题整理里的银行模式例题一览

复习题整理第 118–125 页把笔记第 3 章的银行模式例子又整理了一遍，SQL 都已写在正文里，按下表查找。

| 题目 | 要点 | 位置 |
| --- | --- | --- |
| 写出 borrower 与 loan 自然连接后投影的 SQL | 同名属性加表名前缀 | 4.3 |
| 街道名含 Main 的顾客 | `like '%Main%'` | 5.3 |
| 按字母序输出 Perryridge 的贷款顾客 | `order by` | 5.4 |
| 按银行名升序、贷款额降序输出 | 多个排序键；原稿 `account` 应为 `amount` | 5.4 |
| 有存款或有贷款、不去重版、两者都有、有存款无贷款 | `union`、`union all`、`intersect`、`except` | 6 |
| Perryridge 的平均余额，customer 的元组个数 | `avg`、`count(*)` | 7.1 |
| 各银行平均余额，各银行储户数 | `group by`，`count(distinct ...)` | 7.2 |
| 平均余额超过 1200 的银行 | `having` | 7.3 |
| 住在 Harrison、至少三个账户的顾客的平均余额 | where 与 having 分工 | 7.3、题 9 |
| 贷款额为空的贷款号 | `is null`；原稿 `account` 应为 `amount` | 8.2 |
| 既有存款又有贷款，有贷款无存款，名字不是 Smith 和 Jones | `in`、`not in` | 9.2 |
| 在 Perryridge 既有存款又有贷款 | 二元组 `in` | 9.2 |
| 资产高于 Brooklyn 某家银行、高于所有银行 | 自连接、`> some`、`> all` | 9.3 |
| 平均存款额最高的银行 | `>= all` | 9.3、题 4 |
| 既有存款又有贷款（exists 写法），在 Brooklyn 每家银行都有存款 | `exists`，`not exists` 加 `except` | 9.4 |
| 在 Perryridge 至多一个账户、多于一个账户 | `unique`、`not unique` | 9.5 |
| 各银行存款总额的最大值，平均存款额超过 1200 的银行 | from 子查询 | 10.1 |
| 余额最大的账户，存款总额大于平均值的银行 | `with` | 10.2 |

（更正：复习题整理第 122 页二元组 `in` 后面的子查询漏了括号，应写成 `(branch_name, customer_name) in (select ...)`；第 125 页两个 with 例子的错误和笔记相同，见 10.2 节。）

更多练习见期末复习题整理稿（`20` 号起的文件）和 `31 课后习题 第3章 SQL.md`。
