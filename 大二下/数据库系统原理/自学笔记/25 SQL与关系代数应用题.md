# SQL与关系代数应用题

本文整理自 `笔记/数据库期末复习题整理.pdf` 里历年卷的应用题：第 26–29 页（2020-2021 学年第 1 学期 五）、第 33 页（2019-2020 学年第 2 学期 七）、第 36–37 页（2018-2019 学年第 2 学期 五）、第 49–50 页（2015 年数据库原理A 五）、第 52–53 页（2014 年数据库原理A 四.1）、第 117–118 页"历年题整理"，另加第 19 页顶部一道统计选课人数的 SQL。原稿只有 2020-2021 卷和第 19 页那题带答案，其余题的解答是整理时补写的，标了"（整理补充）"。

课程目录 `README.md` 里原作者回忆：期末最后一题是 20 分的 SQL 题，给一整段题干，按题意写 SQL 和关系代数；22 级软件那年考到了视图的创建、关系代数里排序的写法、SQL 中 date 类型的操作。他的建议是不管会不会写，先用中文把思路写上去，因为阅卷老师没时间细看代码。

## 用到的关系模式

### university 模式

2020-2021 卷和实验题都用教材《数据库系统概念》的大学模式。原卷给出的模式有两处和教材不一样：`teaches` 写成了 `teaches(ID, course_id, section_id, semester, year)`，`advisor` 写成了 `Advisor(s_id, i_id)`。本文统一按教材写 `sec_id` 和 `advisor(s_ID, i_ID)`。如果照原卷写 `section_id`，`teaches` 和 `section`、`takes` 自然连接时就不会按课程段编号去比较，结果会出错。

| 关系 | 属性 | 主码 |
| --- | --- | --- |
| department | `dept_name, building, budget` | `dept_name` |
| instructor | `ID, name, dept_name, salary` | `ID` |
| course | `course_id, title, dept_name, credits` | `course_id` |
| section | `course_id, sec_id, semester, year, building, room_number, time_slot_id` | `course_id, sec_id, semester, year` |
| teaches | `ID, course_id, sec_id, semester, year` | 全部属性 |
| student | `ID, name, dept_name, tot_cred` | `ID` |
| takes | `ID, course_id, sec_id, semester, year, grade` | `ID, course_id, sec_id, semester, year` |
| advisor | `s_ID, i_ID` | `s_ID` |
| prereq | `course_id, prereq_id` | `course_id, prereq_id` |
| classroom | `building, room_number, capacity` | `building, room_number` |
| time_slot | `time_slot_id, day, start_time, end_time` | `time_slot_id, day, start_time` |

教材 DDL 里 `year` 是 `numeric(4,0)`，`semester` 取 `'Fall'`、`'Winter'`、`'Spring'`、`'Summer'`。

做自然连接之前先看同名属性，下面几组最容易出错：

- `student` 和 `instructor` 都有 `ID`、`name`、`dept_name`。
- `takes` 和 `teaches` 都有 `ID`，一个是学生号，一个是老师号。
- `course` 和 `student`、`instructor`、`department` 都有 `dept_name`。课程的 `dept_name` 是开课学院，学生的是所在学院，两者不一定相同。
- `department`、`section`、`classroom` 都有 `building`。`course natural join department natural join section` 会把系楼和上课楼也拿来比较。

### 银行模式（2019-2020 七）

原卷注明带下划线的属性为主码，并假设客户名字不重复。

| 关系 | 属性 | 主码 |
| --- | --- | --- |
| branch 分行表 | `branch_name, branch_city, assets` | `branch_name` |
| customer 客户表 | `customer_name, customer_street, customer_city` | `customer_name` |
| loan 贷款明细表 | `loan_number, branch_name, amount` | `loan_number` |
| borrower 客户贷款表 | `customer_name, loan_number` | `customer_name, loan_number` |
| account 存款明细表 | `account_number, branch_name, balance` | `account_number` |
| depositor 客户存款表 | `customer_name, account_number` | `customer_name, account_number` |

### 银行模式（历年题整理）

这一版用 `customer_id` 标识客户，属性顺序也和上面不同。原稿有几处笔误：`customer_street.cistomer_city` 应为 `customer_street, customer_city`，`loan_nunber`、`loan_numnber` 应为 `loan_number`。

| 关系 | 属性 | 主码 |
| --- | --- | --- |
| branch | `branch_name, branch_city, assets` | `branch_name` |
| customer | `customer_id, customer_name, customer_street, customer_city` | `customer_id` |
| loan | `loan_number, amount, branch_name` | `loan_number` |
| account | `account_number, balance, branch_name` | `account_number` |
| borrower | `customer_id, loan_number` | `customer_id, loan_number` |
| depositor | `customer_id, account_number` | `customer_id, account_number` |

### 手机游戏（2018-2019 五）

原卷没有标主码，下表按语义补。

| 关系 | 属性 | 主码 |
| --- | --- | --- |
| GAME | `Gid, Gname, version, type`：游戏 id、游戏名、版本号、游戏类别 | `Gid` |
| PERSON | `Pid, Pname, age, hobby`：玩家 id、玩家昵称、年龄、爱好 | `Pid` |
| PG | `Pid, Gid`：玩家下载的游戏 | `Pid, Gid` |

### 连锁超市（2015 年数据库原理A 五）

原卷没有标主码，下表按语义补。

| 关系 | 属性 | 主码 |
| --- | --- | --- |
| 门店 MD | `Mid, Mname, MRnum, Maddress`：门店编号、门店名称、店员人数、地址 | `Mid` |
| 商品 SP | `Pid, Pname, Price`：商品编号、商品名称、价格 | `Pid` |
| 所属关系 MP | `Mid, Pid, Pnum`：门店编号、商品编号、商品数量 | `Mid, Pid` |

### 仓库物料（2014 年数据库原理A 四.1）

原卷没有标主码，下表按语义补。

| 关系 | 属性 | 主码 |
| --- | --- | --- |
| 仓库 W | `Wid, WN, Addr`：仓库号、仓库名称、地址 | `Wid` |
| 物料 M | `Mid, MN, Price, p_Time, q_Time`：物料号、物料名称、价格、生产日期、保质期 | `Mid` |
| 存储 ST | `Wid, Mid, SNum`：仓库号、物料号、存储数量 | `Wid, Mid` |

## 解题方法

1. 先用中文写一句思路，比如"全体学生减去 2020 年秋季选过课的学生"，再翻译成关系代数和 SQL。
2. 题目出现"所有""全部"，用**除法**。关系代数写 $r \div s$，除数 $s$ 的属性必须是被除数 $r$ 属性的子集，被除数要先投影掉多余属性。SQL 没有除法，写成 `not exists (要求的全部集合 except 这个人实际有的集合)`，差为空说明全包含了。MySQL 8.0.31 以前不支持 `except`，改成两层 `not exists`。
3. 题目出现"没有""不在""但不"，用差。关系代数写 $r - s$，SQL 用 `except`、`not in` 或 `not exists`。`not in` 的子查询结果里只要有一个 null，条件就变成 unknown，一行都查不出来，子查询列可能为空时改用 `not exists`。
4. 不许用聚集函数求最值：把关系和自己的换名做笛卡尔积，找出"存在比它大的"那些值，再用全体减掉。
5. 分组后的条件放 `having`，分组前的条件放 `where`。`select` 里没有被聚集的列都要写进 `group by`。关系代数先用 ${}_{G}\mathcal{G}_{F}(E)$ 分组聚集，再在外面套 $\sigma$。
6. 要求"没有就输出 0"或者要保留计数为 0 的组：不带 `group by` 的 `count` 在没有输入行时返回 0；按组统计时用左外连接加 `count(右表的列)`，或者在 `select` 里写标量子查询。
7. 更新类题目：关系代数写赋值 $r \leftarrow E$，只改部分元组时写成 $r \leftarrow \Pi_{\ldots}(\sigma_{P}(r)) \cup (r - \sigma_{P}(r))$。SQL 里多条 `update` 的先后顺序会影响结果，可以用一条带 `case` 的 `update`。
8. 排序：教材的关系代数里没有排序运算，关系是集合，元组本来就没有顺序。有的教材（如 Garcia-Molina、Ullman、Widom 的《数据库系统全书》）在扩展关系代数里用 $\tau_{L}$ 表示按属性表 $L$ 排序，例如 $\tau_{tot\_cred\ desc}(student)$。考试用到时写上，并注明这是扩展关系代数的运算。"取前 N 行"在关系代数里没有对应运算，SQL 里 MySQL 写 `limit N`，标准 SQL 写 `fetch first N rows only`。
9. 日期运算各数据库写法不一样，下面是 MySQL 的：`curdate()` 取今天，`date_add(d, interval 3 month)`、`date_sub(d, interval 3 month)` 加减月份，`datediff(d1, d2)` 得到 `d1 - d2` 的天数，`timestampdiff(month, d1, d2)` 得到相差的月数，`year(d)`、`month(d)` 取年、月。标准 SQL 写 `current_date + interval '3' month`。
10. 字符串常量用单引号。`year` 是数值型，写 `year = 2020` 不加引号；加了引号 MySQL 会隐式转换，也能查出来，但不规范。

## 2020-2021 学年第 1 学期 五、应用题

题干：根据 university 模式，使用关系代数和 SQL 语句完成下列查询。原稿的答案是手写的，下面照录后逐条核对。原答案的年份都写成了字符串（如 `year = '2020'`），这里统一改成数值，后面不再逐条说明。

### 2020-2021 五.1 2020 年秋季没有选课的学生 ID

题目：查询在 2020 年秋季没有选课的学生的 ID。

思路：没有选课 = 全部学生 − 2020 年秋季在 `takes` 里出现过的学生。

解答：

关系代数：

$$\Pi_{ID}(student) - \Pi_{ID}(\sigma_{year=2020 \wedge semester=\text{'Fall'}}(takes))$$

SQL 写法 1：

```sql
select ID
from student
where ID not in (select ID
                 from takes
                 where semester = 'Fall' and year = 2020);
```

（更正：原文作 `where not in (select ID ...)`，`not in` 前面漏了被比较的属性 `ID`。）`takes.ID` 是主码的一部分，不会是 null，这里用 `not in` 没问题。

SQL 写法 2：

```sql
(select ID from student)
except
(select ID from takes where semester = 'Fall' and year = 2020);
```

### 2020-2021 五.2 选了 Einstin 老师开设的所有课程的学生姓名

题目：查询选了名叫 'Einstin' 的老师开设的所有的课程的学生的姓名。（老师和学生均无重名）

原稿题目和手写答案都拼作 Einstin，教材样例数据里的老师是 Einstein，这里照原题写。

思路："所有课程"用除法。被除数是（学生姓名，课程号），除数是 Einstin 开过的课程号。题目说学生无重名，所以可以直接按姓名除。

解答：

关系代数（原答案，核对无误）：

$$\Pi_{name,\ course\_id}(student \bowtie takes) \div \Pi_{course\_id}(\sigma_{name=\text{'Einstin'}}(instructor \bowtie teaches))$$

`student` 和 `takes` 只有 `ID` 同名，`instructor` 和 `teaches` 也只有 `ID` 同名，两个自然连接都没问题。被除数投影成 `(name, course_id)`，除数是 `(course_id)`，商是 `name`。

SQL（原答案思路：$A \div B$ 就是 `not exists (B except A)`，$B - A$ 为空说明 $B \subseteq A$）：

```sql
select S.name
from student as S
where not exists (
    (select T.course_id
     from instructor as I join teaches as T on I.ID = T.ID
     where I.name = 'Einstin')
    except
    (select X.course_id
     from takes as X
     where X.ID = S.ID));
```

（更正：原文第一个子查询作 `from instructor natural join teacher as T where T.name = 'Einstin'`。表名 `teacher` 应为 `teaches`；别名 `T` 只加在 `teaches` 上，`teaches` 没有 `name` 属性，`T.name` 会报错，姓名要从 `instructor` 取。）原文第二个子查询写的是 `from student natural join takes as X`，多连了一个 `student`，结果不受影响，直接用 `takes` 就够了。

MySQL 8.0.31 以前没有 `except`，可以写成两层 `not exists`：不存在一门 Einstin 开的课，是这个学生没选过的。

```sql
select S.name
from student as S
where not exists (
    select *
    from teaches as T join instructor as I on T.ID = I.ID
    where I.name = 'Einstin'
      and not exists (select *
                      from takes as X
                      where X.ID = S.ID and X.course_id = T.course_id));
```

### 2020-2021 五.3 2020 年秋季开课但 2021 年春季不开课的课程 ID

题目：查询在 2020 年秋季开课，但不在 2021 年春季开课的所有课程 ID。

思路：开课信息在 `section` 里，两个学期的课程号做差。

解答：

关系代数（原答案，核对无误）：

$$\Pi_{course\_id}(\sigma_{year=2020 \wedge semester=\text{'Fall'}}(section)) - \Pi_{course\_id}(\sigma_{year=2021 \wedge semester=\text{'Spring'}}(section))$$

SQL（原答案，核对无误）：

```sql
(select course_id
 from section
 where year = 2020 and semester = 'Fall')
except
(select course_id
 from section
 where year = 2021 and semester = 'Spring');
```

不支持 `except` 时：

```sql
select distinct course_id
from section
where year = 2020 and semester = 'Fall'
  and course_id not in (select course_id
                        from section
                        where year = 2021 and semester = 'Spring');
```

### 2020-2021 五.4 按年薪分档给老师涨工资

题目：给年薪大于 7 万元的老师涨 5% 的工资，给年薪小于等于 7 万元的老师涨 10% 的工资。

思路：两档条件互斥。关系代数用广义投影算出两部分新元组再合并；SQL 分两条 `update` 时要注意顺序，用 `case` 则不用管顺序。

解答：

关系代数（原答案，核对无误）：

$$instructor \leftarrow \Pi_{ID,\ name,\ dept\_name,\ salary \ast 1.05}(\sigma_{salary \gt 70000}(instructor)) \cup \Pi_{ID,\ name,\ dept\_name,\ salary \ast 1.1}(\sigma_{salary \le 70000}(instructor))$$

右边两部分都是从更新前的 `instructor` 算出来的，所以关系代数这里没有顺序问题。

SQL 写法一（原答案，核对无误）：

```sql
update instructor
set salary = salary * 1.05
where salary > 70000;

update instructor
set salary = salary * 1.1
where salary <= 70000;
```

顺序不能反过来。先涨 10% 的话，年薪 65000 的老师涨到 71500，第二条语句又会给他涨 5%。按原答案的顺序，涨过 5% 的人工资仍然大于 70000，不会被第二条语句再涨一次。

SQL 写法二（原答案，核对无误）：

```sql
update instructor
set salary = case
                 when salary > 70000 then salary * 1.05
                 else salary * 1.1
             end;
```

### 2020-2021 五.5 不用聚集函数求老师的最高工资

题目：不使用聚集函数，查询老师的最高工资。

思路：最高工资 = 全部工资 − 不是最高的工资。"不是最高的"就是存在别人比它高，用 `instructor` 和它的换名做笛卡尔积找出来。

解答：

关系代数（原答案，核对无误）：

$$\Pi_{salary}(instructor) - \Pi_{instructor.salary}(\sigma_{instructor.salary \lt A.salary}(instructor \times \rho_{A}(instructor)))$$

SQL（原答案，核对无误）：

```sql
(select salary
 from instructor)
except
(select A.salary
 from instructor as A, instructor as B
 where A.salary < B.salary);
```

不支持 `except` 时：

```sql
select distinct A.salary
from instructor as A
where not exists (select *
                  from instructor as B
                  where B.salary > A.salary);
```

## 2019-2020 学年第 2 学期 七、应用题

题干：银行企业的数据库由 branch、customer、loan、borrower、account、depositor 六张表组成（见上面的银行模式表），带下划线的属性为主码，假设客户的名字不相同。原卷无答案，下面是整理补充的解答。

题目里的"Brighton 银行"按分行名 `branch_name = 'Brighton'` 理解。如果指的是分行所在城市，要再连 `branch`，条件改成 `branch_city = 'Brighton'`。

### 2019-2020 七.1 用 SQL 创建表

题目：用 SQL 语句创建表。

思路：先建被参照的表（`branch`、`customer`），再建带外码的表。两个客户关联表的主码是两列组合。

解答（整理补充）：数据类型和长度原卷没给，下面是自己设的。

```sql
create table branch (
    branch_name  varchar(30),
    branch_city  varchar(30),
    assets       numeric(16,2),
    primary key (branch_name)
);

create table customer (
    customer_name    varchar(30),
    customer_street  varchar(50),
    customer_city    varchar(30),
    primary key (customer_name)
);

create table loan (
    loan_number  varchar(10),
    branch_name  varchar(30),
    amount       numeric(12,2),
    primary key (loan_number),
    foreign key (branch_name) references branch (branch_name)
);

create table borrower (
    customer_name  varchar(30),
    loan_number    varchar(10),
    primary key (customer_name, loan_number),
    foreign key (customer_name) references customer (customer_name),
    foreign key (loan_number) references loan (loan_number)
);

create table account (
    account_number  varchar(10),
    branch_name     varchar(30),
    balance         numeric(12,2),
    primary key (account_number),
    foreign key (branch_name) references branch (branch_name)
);

create table depositor (
    customer_name   varchar(30),
    account_number  varchar(10),
    primary key (customer_name, account_number),
    foreign key (customer_name) references customer (customer_name),
    foreign key (account_number) references account (account_number)
);
```

### 2019-2020 七.2 Brighton 银行存款客户的姓名、存款号和存款额

题目：使用关系代数和 SQL 语句找出在 Brighton 银行中有存款的所有客户的姓名、存款号和存款额。

思路：`depositor` 连 `account`（公共属性只有 `account_number`），按分行名选择后投影。

解答（整理补充）：

$$\Pi_{customer\_name,\ account\_number,\ balance}(\sigma_{branch\_name=\text{'Brighton'}}(depositor \bowtie account))$$

```sql
select D.customer_name, A.account_number, A.balance
from depositor as D join account as A on D.account_number = A.account_number
where A.branch_name = 'Brighton';
```

### 2019-2020 七.3 平均余额小于 5000 元的支行

题目：使用关系代数和 SQL 语句找出账户平均余额小于 5000 元的支行，显示支行名称及账户平均余额。

思路：按 `branch_name` 分组求平均余额，分组后的条件放 `having`。

解答（整理补充）：

$$\sigma_{avg\_balance \lt 5000}({}_{branch\_name}\mathcal{G}_{avg(balance)\ as\ avg\_balance}(account))$$

```sql
select branch_name, avg(balance) as avg_balance
from account
group by branch_name
having avg(balance) < 5000;
```

### 2019-2020 七.4 有贷款但无账户的客户

题目：使用关系代数和 SQL 语句找出所有在银行中有贷款但无账户的客户。

思路：贷款客户 − 存款客户。客户名字不重复，可以直接按名字做差。

解答（整理补充）：

$$\Pi_{customer\_name}(borrower) - \Pi_{customer\_name}(depositor)$$

```sql
(select customer_name from borrower)
except
(select customer_name from depositor);
```

不支持 `except` 时：

```sql
select distinct customer_name
from borrower
where customer_name not in (select customer_name from depositor);
```

### 2019-2020 七.5 余额大于平均存款额的账户付 3% 利息

题目：使用关系代数和 SQL 语句对所有存款余额大于平均存款额的账户付 3% 的利息。

思路：先算出平均余额，选出余额大于它的账户乘 1.03，其余账户不变，两部分合并后赋值回 `account`。

解答（整理补充）：

关系代数：

$$temp \leftarrow \mathcal{G}_{avg(balance)\ as\ avg\_bal}(account)$$

$$high \leftarrow \Pi_{account\_number,\ branch\_name,\ balance}(\sigma_{balance \gt avg\_bal}(account \times temp))$$

$$account \leftarrow \Pi_{account\_number,\ branch\_name,\ balance \ast 1.03}(high) \cup (account - high)$$

`temp` 只有一个元组，和 `account` 做笛卡尔积相当于给每个账户附上平均余额。

SQL（标准写法）：

```sql
update account
set balance = balance * 1.03
where balance > (select avg(balance) from account);
```

标准 SQL 的语义是先把所有满足 `where` 的元组找出来，再统一更新，所以子查询里的平均值是更新前的值，不会边改边变。

MySQL 不允许 `update` 语句的子查询直接读被更新的表，上面这句会报错 1093。把平均值包进派生表就可以执行：派生表里有聚集函数，MySQL 会先把它物化成临时结果。

```sql
update account
set balance = balance * 1.03
where balance > (select avg_bal
                 from (select avg(balance) as avg_bal from account) as t);
```

也可以先把平均值存进用户变量 `@avg_bal`，再在 `update` 里比较。

## 2018-2019 学年第 2 学期 五、应用题

题干：有一个手机游戏平台，有若干游戏供玩家下载，该应用抽象成 GAME、PERSON、PG 三个关系模式（见上面的手机游戏表）。原卷无答案，下面是整理补充的解答。

### 2018-2019 五.1 玩家 wxy 下载的所有游戏名称

题目：分别用关系代数和 SQL 语句查询玩家"wxy"下载的所有游戏的名称。

思路：`PERSON`、`PG`、`GAME` 依次按 `Pid`、`Gid` 连接，按昵称选择。这里的"所有"只是把游戏全列出来，不用除法。

解答（整理补充）：

$$\Pi_{Gname}(\sigma_{Pname=\text{'wxy'}}(PERSON \bowtie PG \bowtie GAME))$$

三个关系之间只有 `Pid`、`Gid` 同名，自然连接没有问题。

```sql
select distinct G.Gname
from PERSON as P
     join PG on P.Pid = PG.Pid
     join GAME as G on PG.Gid = G.Gid
where P.Pname = 'wxy';
```

### 2018-2019 五.2 下载数量超过 500 次的游戏名称

题目：分别用关系代数和 SQL 语句查询下载数量超过 500 次的游戏名称。

思路：`PG` 里一条元组就是一次下载，按 `Gid` 分组计数，筛出大于 500 的，再连 `GAME` 取名称。

解答（整理补充）：

$$\Pi_{Gname}(\sigma_{cnt \gt 500}({}_{Gid}\mathcal{G}_{count(Pid)\ as\ cnt}(PG)) \bowtie GAME)$$

```sql
select G.Gname
from GAME as G join PG on G.Gid = PG.Gid
group by G.Gid, G.Gname
having count(*) > 500;
```

按 `Gid` 分组，防止两款同名游戏被算在一起；`Gname` 出现在 `select` 里，所以也要写进 `group by`。

### 2018-2019 五.3 下载了 G01 没下载 G02 的玩家姓名

题目：分别用关系代数和 SQL 语句查询下载了游戏 G01 没下载游戏 G02 的玩家姓名。

思路：下载过 G01 的玩家 id − 下载过 G02 的玩家 id，再连 `PERSON` 取昵称。差要在 `Pid` 上做，昵称可能重名。

解答（整理补充）：

$$\Pi_{Pname}(PERSON \bowtie (\Pi_{Pid}(\sigma_{Gid=\text{'G01'}}(PG)) - \Pi_{Pid}(\sigma_{Gid=\text{'G02'}}(PG))))$$

```sql
select P.Pname
from PERSON as P
     join ((select Pid from PG where Gid = 'G01')
           except
           (select Pid from PG where Gid = 'G02')) as T
     on P.Pid = T.Pid;
```

不支持 `except` 时：

```sql
select Pname
from PERSON
where Pid in (select Pid from PG where Gid = 'G01')
  and Pid not in (select Pid from PG where Gid = 'G02');
```

### 2018-2019 五.4 下载了所有益智类游戏的玩家 id

题目：用关系代数完成查询：下载了所有益智类游戏的玩家 id。（原卷"玩家d"应为"玩家id"）

思路："所有"用除法。`PG(Pid, Gid)` 除以益智类游戏的 `Gid` 集合，商就是 `Pid`。类别的具体取值原卷没给，按 `'益智'` 写。

解答（整理补充）：

$$PG \div \Pi_{Gid}(\sigma_{type=\text{'益智'}}(GAME))$$

`PG` 只有 `Pid`、`Gid` 两列，不用先投影。题目只要求关系代数，下面附上对应的 SQL 方便对照：

```sql
select P.Pid
from PERSON as P
where not exists (
    (select Gid from GAME where type = '益智')
    except
    (select Gid from PG where PG.Pid = P.Pid));
```

### 2018-2019 五.5 年龄 15–25 岁的玩家下载了哪些游戏

题目：用 SQL 语句完成查询：查询年龄 15–25 岁之间的玩家都下载了哪些游戏。

思路：三表连接，按年龄筛选，游戏去重。`between` 两端都包含。

解答（整理补充）：

```sql
select distinct G.Gid, G.Gname
from PERSON as P
     join PG on P.Pid = PG.Pid
     join GAME as G on PG.Gid = G.Gid
where P.age between 15 and 25;
```

如果要按玩家列出，把 `select` 改成 `P.Pname, G.Gname`。如果题意是"这个年龄段每个玩家都下载过的游戏"，就变成除法题，写法参照 五.4。

### 2018-2019 五.6 用户 p01 下载了游戏 G02

题目：用户 'p01' 下载了游戏 'G02'，在数据库系统中应当发布的 SQL 语句是什么？

思路：下载记录新增一条，是对 `PG` 的插入。

解答（整理补充）：

```sql
insert into PG (Pid, Gid)
values ('p01', 'G02');
```

### 2018-2019 五.7 游戏重新上传后版本号变化

题目：用 SQL 语句完成：游戏开发者修改了游戏后重新上传，版本号发生了变化，数据库完成什么操作。

思路：游戏还是同一个 `Gid`，只改 `version` 属性，做 `update`。不要删掉再插入，那样 `PG` 里参照这款游戏的下载记录会受外码约束影响。

解答（整理补充）：新版本号和游戏 id 原卷没给，下面用 `'2.0'` 和 `'G01'` 占位。

```sql
update GAME
set version = '2.0'
where Gid = 'G01';
```

## 2015 年数据库原理A 五、应用题

题干：某连锁超市商品管理系统中，包含 MD、SP、MP 三个关系模式（见上面的连锁超市表）。原卷无答案，下面是整理补充的解答。门店编号的类型原卷没给，按字符串写 `'256'`。

### 2015 五.1 店员人数不少于 50 人的门店名称和地址

题目：用关系代数表达式查询：店员人数不少于 50 人的门店名称和地址。

思路：对 `MD` 选择再投影。

解答（整理补充）：

$$\Pi_{Mname,\ Maddress}(\sigma_{MRnum \ge 50}(MD))$$

### 2015 五.2 至少供应了 256 号商店全部商品的其它商店

题目：用关系代数表达式查询：找出至少供应了代号为"256"的商店所供应的全部商品的其它商店名和地址。

思路：除法。被除数是 `MP` 投影到 `(Mid, Pid)`，一定要去掉 `Pnum`，否则商不对；除数是 256 号店的商品集合。256 号店自己也满足条件，题目要"其它商店"，要从商里去掉。

解答（整理补充）：

$$\Pi_{Mname,\ Maddress}(MD \bowtie \sigma_{Mid \ne \text{'256'}}(\Pi_{Mid,\ Pid}(MP) \div \Pi_{Pid}(\sigma_{Mid=\text{'256'}}(MP))))$$

商只有 `Mid` 一个属性，和 `MD` 自然连接时只按 `Mid` 比较。

### 2015 五.3 库存数量小于 10 件的商品名称及所在门店名称

题目：用 SQL 语句查询：库存数量小于 10 件的商品名称及所在的门店名称。

思路：`MP` 连 `SP` 取商品名、连 `MD` 取门店名，按 `Pnum` 筛选。

解答（整理补充）：

```sql
select SP.Pname, MD.Mname
from MP
     join SP on MP.Pid = SP.Pid
     join MD on MP.Mid = MD.Mid
where MP.Pnum < 10;
```

### 2015 五.4 各门店名称和所拥有商品的总价格

题目：用 SQL 语句查询：各门店名称和所拥有的商品的总价格。

思路：每种商品的总价是 `Price * Pnum`，按门店分组求和。

解答（整理补充）：

```sql
select MD.Mid, MD.Mname, coalesce(sum(SP.Price * MP.Pnum), 0) as total_price
from MD
     left join MP on MD.Mid = MP.Mid
     left join SP on MP.Pid = SP.Pid
group by MD.Mid, MD.Mname;
```

用左外连接是为了让没有商品的门店也出现，这时 `sum` 得到 null，用 `coalesce` 换成 0。只统计有商品的门店时，把两个 `left join` 换成 `join`、去掉 `coalesce` 即可。按 `Mid` 分组，防止同名门店被合并。

### 2015 五.5 256 号门店的泉阳泉数量增加 2000

题目：用 SQL 语句，将"256"号门店的"泉阳泉"的数量增加 2000。

思路：`MP` 里只有商品编号，商品名要到 `SP` 里查，用子查询找出编号。

解答（整理补充）：

```sql
update MP
set Pnum = Pnum + 2000
where Mid = '256'
  and Pid in (select Pid from SP where Pname = '泉阳泉');
```

## 2014 年数据库原理A 四、应用题 1

题干：有仓库-物料存储系统，其关系模式为 W、M、ST（见上面的仓库物料表）。原卷无答案，下面是整理补充的解答。

### 2014 四.1.1 存储了全部物料的仓库名称

题目：用关系代数表达式，查询存储了全部物料的仓库名称。

思路：除法。`ST` 投影到 `(Wid, Mid)` 去掉 `SNum`，除以全部物料号，得到仓库号，再连 `W` 取名称。

解答（整理补充）：

$$\Pi_{WN}(W \bowtie (\Pi_{Wid,\ Mid}(ST) \div \Pi_{Mid}(M)))$$

### 2014 四.1.2 存储数量高于 100 的物料号

题目：用关系代数表达式，查询所有存储数量高于 100 的物料号。

思路：按"某个仓库里的存储数量高于 100"理解，直接对 `ST` 选择再投影。

解答（整理补充）：

$$\Pi_{Mid}(\sigma_{SNum \gt 100}(ST))$$

如果题意是各仓库合计的存储数量高于 100，先按物料分组求和：

$$\Pi_{Mid}(\sigma_{total \gt 100}({}_{Mid}\mathcal{G}_{sum(SNum)\ as\ total}(ST)))$$

### 2014 四.1.3 没有生产日期的物料号和物料名称

题目：用 SQL 语句，查询所有没有生产日期的物料号和物料名称。

思路："没有生产日期"就是 `p_Time` 为空。判断空值只能用 `is null`，写 `= null` 结果是 unknown，一行也查不出来。

解答（整理补充）：

```sql
select Mid, MN
from M
where p_Time is null;
```

### 2014 四.1.4 建立视图 V：还有三个月就到保质期的物料

题目：用 SQL 语句，建立视图 V，统计截至到今天，还有三个月就要到保质期的物料号和物料名称。

思路：到期日 = 生产日期 + 保质期。"还有三个月就要到保质期"按"还没过期，且到期日在今天到三个月后之间"理解。原卷没说 `q_Time` 存的是保质期长度还是到期日期，两种都写。日期函数用 MySQL 的。

解答（整理补充）：

`q_Time` 存保质期的月数（整数）时：

```sql
create view V as
select Mid, MN
from M
where date_add(p_Time, interval q_Time month)
      between curdate() and date_add(curdate(), interval 3 month);
```

`q_Time` 存到期日期（date 类型）时：

```sql
create view V as
select Mid, MN
from M
where q_Time between curdate() and date_add(curdate(), interval 3 month);
```

视图里只存查询定义，每次查询视图时才重新计算，所以 `curdate()` 总是取查询当天的日期，符合"截至到今天"。标准 SQL 里 `curdate()` 写作 `current_date`，`date_add(curdate(), interval 3 month)` 写作 `current_date + interval '3' month`。

### 2014 四.1.5 物料海绵的库存增加 100 件

题目：用 SQL 语句，修改物料名为"海绵"的库存，增加 100 件。

思路：库存在 `ST` 里，物料名在 `M` 里，用子查询按名称找出物料号。

解答（整理补充）：

```sql
update ST
set SNum = SNum + 100
where Mid in (select Mid from M where MN = '海绵');
```

题目没指定仓库，这句会给存有海绵的每个仓库各加 100 件。只改某个仓库时，再加条件 `and Wid = '仓库号'`。

## 历年题整理：银行模式 7 问

题干：使用关系代数和 SQL 表达下列查询，模式见上面的"银行模式（历年题整理）"。原稿只列了题目，下面是整理补充的解答。题目里的"Austin 银行""Austin 支行"按分行名 `branch_name = 'Austin'` 理解。

### 历年题整理 1 Austin 银行存款客户的姓名、存款号及存款额

题目：找出在 Austin 银行中有存款的所有客户的姓名、存款号及存款额。

思路：这版模式里 `depositor` 只有 `customer_id`，要再连 `customer` 取姓名。

解答（整理补充）：

$$\Pi_{customer\_name,\ account\_number,\ balance}(\sigma_{branch\_name=\text{'Austin'}}(customer \bowtie depositor \bowtie account))$$

```sql
select C.customer_name, A.account_number, A.balance
from customer as C
     join depositor as D on C.customer_id = D.customer_id
     join account as A on D.account_number = A.account_number
where A.branch_name = 'Austin';
```

### 历年题整理 2 资产至少比位于 Rye 的某一家支行高的支行名

题目：找出资产至少比位于 Rye 的某一家支行高的支行名。

思路："比某一家高"就是存在一家 Rye 的支行资产比它低。`branch` 和自己的换名做笛卡尔积再比较。

解答（整理补充）：

$$\Pi_{T.branch\_name}(\sigma_{T.assets \gt S.assets \wedge S.branch\_city=\text{'Rye'}}(\rho_{T}(branch) \times \rho_{S}(branch)))$$

```sql
select distinct T.branch_name
from branch as T, branch as S
where T.assets > S.assets and S.branch_city = 'Rye';
```

也可以用 `some`：

```sql
select branch_name
from branch
where assets > some (select assets from branch where branch_city = 'Rye');
```

### 历年题整理 3 有贷款但无存款的客户姓名

题目：找出在银行中有贷款但无存款的客户姓名。

思路：在 `customer_id` 上做差，再连 `customer` 取姓名。这版模式没有假设客户不重名，不能直接按姓名做差。

解答（整理补充）：

$$\Pi_{customer\_name}(customer \bowtie (\Pi_{customer\_id}(borrower) - \Pi_{customer\_id}(depositor)))$$

```sql
select customer_name
from customer
where customer_id in (select customer_id from borrower)
  and customer_id not in (select customer_id from depositor);
```

### 历年题整理 4 每个支行的贷款户数

题目：找出每个支行的贷款户数。

思路："户数"按贷款客户人数算，一个客户在同一支行有几笔贷款也只算一户。关系代数先投影到 `(branch_name, customer_id)` 去重，再分组计数。

解答（整理补充）：

$${}_{branch\_name}\mathcal{G}_{count(customer\_id)\ as\ num}(\Pi_{branch\_name,\ customer\_id}(borrower \bowtie loan))$$

```sql
select br.branch_name, count(distinct B.customer_id) as num_borrowers
from branch as br
     left join loan as L on br.branch_name = L.branch_name
     left join borrower as B on L.loan_number = B.loan_number
group by br.branch_name;
```

用左外连接，没有贷款的支行也会列出来，户数为 0（`count` 不计 null）。如果"户数"按贷款笔数算，把 `count(distinct B.customer_id)` 换成 `count(distinct L.loan_number)`。

### 历年题整理 5 所有存款余额增加 7%

题目：将所有存款的余额增加 7%。

思路：全部元组都改，关系代数用广义投影后赋值。

解答（整理补充）：

$$account \leftarrow \Pi_{account\_number,\ balance \ast 1.07,\ branch\_name}(account)$$

```sql
update account
set balance = balance * 1.07;
```

### 历年题整理 6 删除余额低于平均余额的账户

题目：删除余额低于银行平均余额的账户记录。

思路：先算平均余额，找出低于它的账户，从 `account` 里减掉。

解答（整理补充）：

$$temp \leftarrow \mathcal{G}_{avg(balance)\ as\ avg\_bal}(account)$$

$$account \leftarrow account - \Pi_{account\_number,\ balance,\ branch\_name}(\sigma_{balance \lt avg\_bal}(account \times temp))$$

```sql
delete from account
where balance < (select avg(balance) from account);
```

和 七.5 一样，标准 SQL 先找出所有要删的元组再删，平均值不会边删边变。MySQL 会报错 1093，改成派生表：

```sql
delete from account
where balance < (select avg_bal
                 from (select avg(balance) as avg_bal from account) as t);
```

如果 `depositor` 对 `account` 建了外码又没写 `on delete cascade`，要先删掉 `depositor` 里对应的行，否则删除会被拒绝。

### 历年题整理 7 定义视图 v_customer

题目：用 SQL 定义视图 `v_customer`，求出 Austin 支行的所有客户姓名。

思路：支行的客户包括在这里存款的和在这里贷款的，两部分用 `union` 合起来。旧版教材用银行模式讲视图时，`all_customer` 视图也是这样写的。

解答（整理补充）：

```sql
create view v_customer as
    select C.customer_name
    from customer as C
         join depositor as D on C.customer_id = D.customer_id
         join account as A on D.account_number = A.account_number
    where A.branch_name = 'Austin'
    union
    select C.customer_name
    from customer as C
         join borrower as B on C.customer_id = B.customer_id
         join loan as L on B.loan_number = L.loan_number
    where L.branch_name = 'Austin';
```

查询时直接 `select * from v_customer;`。如果只算存款客户，去掉 `union` 后面那一段。

## 第 19 页：统计选课人数的最大值

原稿把这段代码放在第 19 页"判断第三范式"的笔记下面，题目和实验题第 8 题相同。

### 统计 2019 年春季所开课程段选课人数的最大值

题目：统计 2019 年春季所开课程段选课人数的最大值。

思路：先按课程段分组数出每段的选课人数，放进 `from` 子查询，外层再求 `max`。学期和年份已经在 `where` 里固定，分组只需要 `course_id, sec_id`。

解答：

原答案：

```sql
SELECT MAX(student_count) AS max_enrollment
FROM (
  SELECT course_id, COUNT(student_id) AS student_count
  FROM enrollment
  WHERE semester = 'Spring' AND year = 2019
  GROUP BY course_id
) AS course_enrollment;
```

（更正：原文用的表 `enrollment` 和属性 `student_id` 在 university 模式里都不存在，应为 `takes` 和 `ID`；原文只按 `course_id` 分组，统计的是每门课所有课程段的人数之和，题目要的是每个课程段的人数，应按 `course_id, sec_id` 分组。）

更正后：

```sql
select max(enrollment) as max_enrollment
from (select course_id, sec_id, count(ID) as enrollment
      from takes
      where semester = 'Spring' and year = 2019
      group by course_id, sec_id) as sec_enrollment;
```

对应的关系代数（整理补充）：

$$\mathcal{G}_{max(enrollment)}({}_{course\_id,\ sec\_id}\mathcal{G}_{count(ID)\ as\ enrollment}(\sigma_{semester=\text{'Spring'} \wedge year=2019}(takes)))$$

没人选的课程段不会出现在 `takes` 里，但它的人数是 0，不影响最大值。只有 2019 年春季一个选课记录都没有时，`max` 才返回 null。
