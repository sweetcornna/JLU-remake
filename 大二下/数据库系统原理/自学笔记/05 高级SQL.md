# 高级 SQL

整理自 `笔记/数据库系统原理笔记.pdf` 第 60–71 页（5.3 嵌入式 SQL 到 5.7 递归查询）和 `老师资料/SQL复习 (1).docx` 2.22 节。第 5 章开头的数据类型和完整性约束放在 `04 中级SQL.md`；触发器、授权和 SQL 注入在 `08 应用开发、触发器与授权.md`。

## 本章要点

- 程序访问数据库有两类接口：语句级接口（嵌入式 SQL、动态 SQL，要先经过预编译器）和调用级接口（ODBC、JDBC，SQL 语句当作字符串交给库函数，不用预编译）。
- 嵌入式 SQL 用 `EXEC SQL ... END-EXEC` 标出 SQL 语句，SQL 里引用宿主语言变量时前面加冒号。
- 游标四步：`declare` 声明，`open` 执行查询，`fetch` 取一行并前移，`close` 关闭。`SQLSTATE` 为 `"02000"` 表示没有数据了。
- 通过游标改数据：声明时加 `for update`，修改语句写 `where current of 游标名`。游标只在 fetch 时移动。
- 动态 SQL 在运行时构造语句：`prepare` 预备，`execute ... using` 执行，`?` 是参数占位符。
- JDBC 的流程：加载驱动，建立连接，创建 `Statement`，用 `executeUpdate` 或 `executeQuery` 执行，遍历 `ResultSet`，最后关闭。
- **函数**有返回值（一个标量或一张表），在 SQL 表达式里调用；**过程**没有返回值，靠 `in`、`out` 参数传递信息，用 `call` 调用。
- 过程化结构：`begin ... end`、`declare`、`set`、`while`、`repeat`、`for`、`if-then-elseif-else`，以及异常处理。
- `with recursive` 的定义是"基查询 union 递归查询"，反复计算到不再产生新元组为止；递归部分必须是单调的，不能对递归视图做聚集，不能在引用它的子查询上用 `not exists`，差运算的右边不能是它。

## 1. 在应用程序中执行 SQL

有些应用既要用 SQL 访问数据，又要用通用编程语言做界面、计算等工作。容纳 SQL 的这门语言叫宿主语言（host language），比如 C、C++、Cobol、Pascal、Java、PL/I、Fortran、Visual Basic。

| 接口 | 做法 | 包括 |
| --- | --- | --- |
| 语句级接口 | SQL 语句写在程序源码里，编译前由预编译器检查语法，并转换成宿主语言能理解的代码 | 嵌入式 SQL、动态 SQL |
| 调用级接口（CLI） | 不需要预编译，SQL 语句以字符串的形式通过函数调用交给数据库 | ODBC、JDBC |

- 嵌入式 SQL：SQL 语句在编译时就确定，由预编译器处理。
- 动态 SQL：程序运行时才生成并执行 SQL 语句，更灵活。直接把用户输入拼进语句有 SQL 注入的风险。
- ODBC（Open DataBase Connectivity）：通用的数据库访问接口，只要数据库提供了 ODBC 驱动，程序就能和它交互。
- JDBC（Java DataBase Connectivity）：Java 访问各种关系数据库的接口。

## 2. 嵌入式 SQL

### 2.1 嵌入标识

| 宿主语言 | 写法 |
| --- | --- |
| C（课件写法） | `EXEC SQL <嵌入式 SQL 语句> END-EXEC` |
| Java（SQLJ） | `#sql { <嵌入式 SQL 语句> };` |

不同宿主语言的结尾写法不一样，教材里 C 语言写成 `EXEC SQL <语句>;`。本文沿用课件的 `END-EXEC`。

SQL 语句里用到的宿主变量，前面加冒号，如 `:amount`。整理补充：这些变量要在声明节里声明，`SQLSTATE` 也在这里声明：

```c
EXEC SQL begin declare section END-EXEC
int  amount;
char cn[21], cc[31];
char SQLSTATE[6];
EXEC SQL end declare section END-EXEC
```

### 2.2 连接数据库

```c
EXEC SQL connect to server user user_name END-EXEC
```

`server` 是数据库服务器，`user_name` 是用户名。

### 2.3 游标

查询结果通常有很多行，宿主语言的变量一次只能放一行，所以用游标一行一行地取。

| 语句 | 作用 |
| --- | --- |
| `declare c cursor for <查询>` | 声明游标 c，此时不执行查询 |
| `open c` | 执行查询，把结果存进临时关系，游标指向第一行之前；查询里宿主变量的值按 open 时刻取 |
| `fetch c into :v1, :v2` | 把当前行的各列存进宿主变量，游标下移一行 |
| `close c` | 关闭游标，释放临时关系 |

例：输出存款余额超过 `amount`（取 500）的顾客名字和所在城市。

```c
EXEC SQL
    declare c cursor for
        select depositor.customer_name, customer_city
        from depositor, customer, account
        where depositor.customer_name = customer.customer_name
          and account.account_number = depositor.account_number
          and account.balance > :amount
END-EXEC

amount = 500;
EXEC SQL open c END-EXEC
EXEC SQL fetch c into :cn, :cc END-EXEC
while (strcmp(SQLSTATE, "02000") != 0) {
    printf("%s, %s\n", cn, cc);
    EXEC SQL fetch c into :cn, :cc END-EXEC
}
EXEC SQL close c END-EXEC
```

- `:amount` 在 open 时被替换成 500，然后和 `account.balance` 比较。
- `strcmp` 两个串相同返回 0。`SQLSTATE` 等于 `"02000"` 说明游标已经到末尾，没有数据可取。

（更正：原文 select 写的是 `customer_name`，`depositor` 和 `customer` 都有这个属性，有歧义，应加前缀。原文循环条件作 `strcmp(SQLSTATE,"02000")!=1`，strcmp 相等时返回 0，应写 `!= 0`。原文每轮先 fetch 再 printf，最后一次 fetch 没取到数据时还会把上一行再打印一遍，所以改成循环前先 fetch 一次、循环体里先打印后 fetch。原文变量声明为 `char cn[20], cc[15]`，`customer_city` 是 `char(30)`，C 字符串还要留一个结束符，这里改成 `cn[21]`、`cc[31]`。）

笔记用的数据：

`account`

| `account_number` | `branch_name` | `balance` |
| --- | --- | --- |
| A-101 | Downtown | 900 |
| A-215 | Mianus | 700 |
| A-102 | Perryridge | 400 |
| A-305 | Round Hill | 350 |
| A-201 | Brighton | 900 |
| A-217 | Brighton | 750 |

`customer`

| `customer_name` | `customer_street` | `customer_city` |
| --- | --- | --- |
| Smith | 4 North St. | Rye |
| Jackson | 12 Alma St. | Palo Alto |
| Hayes | 3 Main St. | Harrison |
| Turner | 123 Putnam St. | Stamford |
| Jones | 100 Main St. | Harrison |
| Lindsay | 72 North | Rye |

`depositor`

| `customer_name` | `account_number` |
| --- | --- |
| Jackson | A-101 |
| Smith | A-201 |
| Hayes | A-215 |
| Turner | A-102 |
| Jones | A-305 |
| Lindsay | A-217 |

（更正：原图 `depositor` 第一行作 Johnson，但 `customer` 表里只有 Jackson，笔记给出的输出里也是 Jackson，这里统一成 Jackson。）

余额超过 500 的账户是 A-101、A-215、A-201、A-217，查询结果有 4 行。按笔记的顺序，每次 fetch 后变量的值和程序输出：

| 第几次 fetch | `cn` | `cc` | 程序已输出 |
| --- | --- | --- | --- |
| 1 | Smith | Rye | Smith, Rye |
| 2 | Jackson | Palo Alto | 再加 Jackson, Palo Alto |
| 3 | Hayes | Harrison | 再加 Hayes, Harrison |
| 4 | Lindsay | Rye | 再加 Lindsay, Rye |
| 5 | 没取到数据 | | `SQLSTATE` 为 `"02000"`，循环结束 |

结果行的先后顺序由数据库决定，没有 `order by` 时不保证。

### 2.4 用嵌入式 SQL 修改数据

任何合法的 `insert`、`delete`、`update` 都可以写成 `EXEC SQL ... END-EXEC`。

需要逐行判断再修改时，用可更新游标。例：为 Perryridge 银行的每个存款账户增加 100 美元。

```c
EXEC SQL
    declare c cursor for
        select *
        from account
        where branch_name = 'Perryridge'
    for update
END-EXEC

EXEC SQL open c END-EXEC
EXEC SQL fetch c into :acc_no, :br_name, :bal END-EXEC
while (strcmp(SQLSTATE, "02000") != 0) {
    EXEC SQL
        update account
        set balance = balance + 100
        where current of c
    END-EXEC
    EXEC SQL fetch c into :acc_no, :br_name, :bal END-EXEC
}
EXEC SQL close c END-EXEC
EXEC SQL commit work END-EXEC
```

- `for update` 表示要通过这个游标修改数据。
- `where current of c` 表示修改游标 c 当前指向的那一行。删除当前行写 `delete from account where current of c`。
- `acc_no`、`br_name`、`bal` 要在声明节里声明。

（更正：原文循环里只有 update 没有 fetch，并解释说"执行 UPDATE 后游标会自动移到下一行"。游标只在 fetch 时移动，不 fetch 的话一直改同一行，循环也停不下来。原文还说"只有关闭游标后修改才会被保存"，修改是在事务提交（`commit work`）时永久保存的，关闭游标只是释放资源。）

这道题其实不用游标，一条语句就行：

```c
EXEC SQL
    update account
    set balance = balance + 100
    where branch_name = 'Perryridge'
END-EXEC
```

## 3. 动态 SQL

动态 SQL 允许程序在运行时构造、提交、执行 SQL 语句。

```c
char *sqlprog = "update account set balance = balance * 1.05 where account_number = ?";
char account[10] = "A-101";

EXEC SQL prepare dynprog from :sqlprog END-EXEC
EXEC SQL execute dynprog using :account END-EXEC
```

| 阶段 | 做什么 |
| --- | --- |
| 预备 `prepare` | 把字符串 `sqlprog` 里的 SQL 语句转换成数据库能执行的内部形式，记为 `dynprog` |
| 执行 `execute` | 执行 `dynprog`，`using :account` 用变量的值 `"A-101"` 替换语句里的占位符 `?` |

- 预备一次可以执行多次，每次换一个参数。
- 参数通过 `?` 和 `using` 传进去，数据库把它当成值，不当成 SQL 的一部分，所以不会被注入。要是把用户输入的账号直接拼进 `sqlprog` 字符串，才有注入风险。

（更正：原文在 prepare 后面套了 `while (strcmp(SQLSTATE,"02000")!=1)` 循环反复 execute。这条 update 每执行一次就给账户再计一次息，循环没有意义，去掉。原文的字符串常量跨了两行，C 语言里字符串常量不能直接换行，这里写成一行。）

## 4. 调用级接口

### 4.1 ODBC

ODBC 定义了应用程序访问数据库的 API（Application Program Interface）。例：统计各银行的存款总额。

```c
void ODBCexample()
{
    RETCODE error;
    HENV env;     /* 环境 */
    HDBC conn;    /* 数据库连接 */

    SQLAllocEnv(&env);
    SQLAllocConnect(env, &conn);
    SQLConnect(conn, "db.example.edu", SQL_NTS, "user", SQL_NTS, "password", SQL_NTS);
    {
        char branchname[80];
        float balance;
        int lenOut1, lenOut2;
        HSTMT stmt;
        char *sqlquery = "select branch_name, sum(balance) from account group by branch_name";

        SQLAllocStmt(conn, &stmt);
        error = SQLExecDirect(stmt, sqlquery, SQL_NTS);
        if (error == SQL_SUCCESS) {
            SQLBindCol(stmt, 1, SQL_C_CHAR, branchname, 80, &lenOut1);
            SQLBindCol(stmt, 2, SQL_C_FLOAT, &balance, 0, &lenOut2);
            while (SQLFetch(stmt) == SQL_SUCCESS)
                printf("%s %g\n", branchname, balance);
        }
        SQLFreeStmt(stmt, SQL_DROP);
    }
    SQLDisconnect(conn);
    SQLFreeConnect(conn);
    SQLFreeEnv(env);
}
```

（更正：原文 `error SQLExecDirect(...)` 漏了等号，`SQLFreeconnect` 大小写错，查询字符串里 `branch name` 缺下划线。服务器名、用户名、密码换成了占位符。）

| 步骤 | 函数 | 说明 |
| --- | --- | --- |
| 分配句柄 | `SQLAllocEnv`、`SQLAllocConnect` | 环境句柄和连接句柄 |
| 连接 | `SQLConnect` | 参数依次是连接句柄、服务器、用户名、密码，每个字符串后跟长度；`SQL_NTS` 表示这是以 `'\0'` 结尾的字符串 |
| 执行 | `SQLAllocStmt`、`SQLExecDirect` | 分配语句句柄，执行 SQL |
| 绑定结果列 | `SQLBindCol` | 参数：语句句柄、第几列、C 类型、变量地址、缓冲区大小、实际长度的存放地址。实际长度返回负值表示该列是 null |
| 取结果 | `SQLFetch` | 每调一次取一行到绑定的变量里，返回 `SQL_SUCCESS` 表示取到了 |
| 释放 | `SQLFreeStmt`、`SQLDisconnect`、`SQLFreeConnect`、`SQLFreeEnv` | 按分配的相反顺序释放 |

### 4.2 JDBC

```java
public static void JDBCexample(String dbid, String userid, String passwd)
{
    try {
        Class.forName("oracle.jdbc.driver.OracleDriver");
        Connection conn = DriverManager.getConnection(dbid, userid, passwd);
        Statement stmt = conn.createStatement();
        try {
            stmt.executeUpdate("insert into account values ('A-9732', 'Perryridge', 1200)");
        } catch (SQLException sqle) {
            System.out.println("Could not insert tuple. " + sqle);
        }
        ResultSet rset = stmt.executeQuery(
            "select branch_name, avg(balance) from account group by branch_name");
        while (rset.next()) {
            System.out.println(rset.getString("branch_name") + " " + rset.getFloat(2));
        }
        stmt.close();
        conn.close();
    } catch (Exception e) {
        System.out.println("Exception: " + e);
    }
}
```

（更正：原文参数列表缺右括号，`rset next()` 漏了点，`System.outprintln` 漏了点。`Class.forName` 会抛出 `ClassNotFoundException`，外层只捕获 `SQLException` 编译不过，这里改成捕获 `Exception`。）

步骤：

1. `Class.forName(...)` 加载数据库驱动。
2. `DriverManager.getConnection(数据库, 用户名, 密码)` 建立连接。
3. `conn.createStatement()` 创建语句对象。
4. 更新用 `executeUpdate`，查询用 `executeQuery`，查询返回 `ResultSet`。
5. `rset.next()` 移到下一行，没有下一行时返回 false。取列值可以按列名 `getString("branch_name")`，也可以按列号 `getFloat(2)`，列号从 1 开始。
6. 用完关闭 `stmt` 和 `conn`。

判断取到的值是否为空。`getInt` 遇到 null 会返回 0，分不清是真的 0 还是空值，要紧接着调 `wasNull()`：

```java
int a = rs.getInt("a");
if (rs.wasNull())
    System.out.println("Got null value");
```

获取结果集的元数据，比如列名和列的类型：

```java
ResultSetMetaData rsmd = rs.getMetaData();
for (int i = 1; i <= rsmd.getColumnCount(); i++) {
    System.out.println(rsmd.getColumnName(i));
    System.out.println(rsmd.getColumnTypeName(i));
}
```

（更正：原文 `rs.wasNULL()` 应为 `wasNull()`，循环条件里 `rsmd` 拼成了 `remd`。）

整理补充：带参数的语句用 `PreparedStatement`，和动态 SQL 的 `?` 一个道理，能防注入。

```java
PreparedStatement pStmt = conn.prepareStatement(
    "insert into account values (?, ?, ?)");
pStmt.setString(1, "A-9732");
pStmt.setString(2, "Perryridge");
pStmt.setInt(3, 1200);
pStmt.executeUpdate();
```

## 5. 函数和过程

SQL 允许用 SQL 自己定义函数和过程：

- 定义好的函数、过程存储在数据库里，多个地方可以反复调用，不用每次重写。
- 外部应用程序除了直接查询、修改数据，也可以调用这些存储的函数和过程。
- 老师资料 2.22：函数和过程都存在数据库里，都可以被调用。函数用 `returns` 声明显式的返回值；过程没有返回值，但可以用 `out` 参数带回结果，而且可以带回多个。

### 5.1 函数

函数有返回值，对图像、几何图形这类特殊数据类型特别有用。

老师资料：给定系名，返回该系的教师人数。

```sql
create function dept_count (dept_name varchar(20))
    returns integer
begin
    declare d_count integer;
    select count(*) into d_count
    from instructor
    where instructor.dept_name = dept_name;
    return d_count;
end
```

（更正：原文函数体里 select 语句末尾缺分号。）

- `returns integer` 声明返回类型，`begin ... end` 之间是函数体。
- `declare` 声明局部变量；`select ... into d_count` 把查询结果存进变量。
- where 里右边的 `dept_name` 指参数。参数和属性同名容易出问题：有的数据库会把它当成 `instructor` 的属性，条件变成自己等于自己；有的会报歧义错误。写成 `dept_count.dept_name`（和下面过程的写法一样）最稳妥。

函数可以用在 SQL 表达式里。找出教师人数多于 12 的系的系名和预算：

```sql
select dept_name, budget
from department
where dept_count(dept_name) > 12;
```

笔记的例子：给定顾客名，返回他的存款账户个数。

```sql
create function account_count (customer_name varchar(20))
    returns integer
begin
    declare a_count integer;
    select count(*) into a_count
    from depositor
    where depositor.customer_name = account_count.customer_name;
    return a_count;
end
```

（更正：原文 where 作 `depositor.account_name = customer_name`，`depositor` 没有 `account_name` 属性，应为 `customer_name`；select 末尾同样缺分号。）

列出拥有多个存款账户的顾客和他们的地址：

```sql
select customer_name, customer_street, customer_city
from customer
where account_count(customer_name) > 1;
```

### 5.2 表函数

返回值是一张表的函数叫表函数。

老师资料：返回某个系全部教师的信息。

```sql
create function instructor_of (dept_name char(20))
    returns table (
        ID        varchar(5),
        name      varchar(20),
        dept_name varchar(20),
        salary    numeric(8,2))
    return table
        (select ID, name, dept_name, salary
         from instructor
         where instructor.dept_name = instructor_of.dept_name);

select *
from table(instructor_of('Music'));
```

笔记：给定顾客名，返回他的全部存款账户。

```sql
create function accounts_of (customer_name varchar(20))
    returns table (
        account_number char(10),
        branch_name    char(15),
        balance        numeric(12,2))
    return table
        (select account_number, branch_name, balance
         from account
         where exists (select *
                       from depositor
                       where depositor.customer_name = accounts_of.customer_name
                         and depositor.account_number = account.account_number));

-- 查询 Smith 的全部存款账户
select *
from table(accounts_of('Smith'));
```

（更正：原文调用时写成 `account_of('Smith')`，函数名是 `accounts_of`。）

`accounts_of.customer_name` 用"函数名.参数名"指参数，和 `depositor.customer_name` 区分开。调用表函数要写 `table(...)`，结果可以像普通关系一样放在 from 里。

### 5.3 过程

过程没有返回值，通过参数传递信息，可以在 SQL 过程或嵌入式 SQL 里调用。参数分 `in`（输入）、`out`（输出），标准里还有 `inout`（既输入又输出）。

老师资料：

```sql
create procedure dept_count_proc (in dept_name varchar(20), out d_count integer)
begin
    select count(*) into d_count
    from instructor
    where instructor.dept_name = dept_count_proc.dept_name;
end

-- 调用
declare d_count integer;
call dept_count_proc('Physics', d_count);
```

（更正：原文过程体里 select 语句末尾缺分号。）

笔记：查询顾客的存款账户数目，并查 Smith 的账户数。

```sql
create procedure accounts_count_proc (in customer_name varchar(20), out a_count integer)
begin
    select count(*) into a_count
    from depositor
    where depositor.customer_name = accounts_count_proc.customer_name;
end

declare a_count integer;
call accounts_count_proc('Smith', a_count);
```

（更正：原文 where 同样作 `depositor.account_name`，应为 `depositor.customer_name`；题目里的 "Smitht" 应为 Smith。）

整理补充：MySQL 里过程体中有分号，要先改语句结束符；在命令行调用时，out 参数用 `@` 开头的会话变量接收。

```sql
delimiter //
create procedure dept_count_proc (in dname varchar(20), out d_count integer)
begin
    select count(*) into d_count
    from instructor
    where instructor.dept_name = dname;
end //
delimiter ;

call dept_count_proc('Physics', @d_count);
select @d_count;
```

参数改名成 `dname`，就不会和属性 `dept_name` 撞名。

### 5.4 外部语言例程

笔记 5.6.5 只有标题。整理补充：函数和过程也可以用 C、Java 等语言写，在 SQL 里只声明接口。

```sql
create function dept_count (dept_name varchar(20))
    returns integer
    language C
    external name '/usr/local/lib/dept_count';
```

外部语言写的例程执行效率高，能做 SQL 做不了的计算。风险是代码有 bug 时可能破坏数据库内部结构，所以常放在单独的进程或沙箱里运行，代价是调用变慢。

## 6. 过程化结构

SQL:1999 起支持循环、分支这类过程化结构，这部分标准叫持久存储模块 PSM（Persistent Stored Modules）。

（更正：原文作 Persistant Storage Module。）

### 6.1 复合语句与变量

```sql
begin
    ...
end
```

`begin ... end` 之间可以写多条 SQL 语句，作为一个单元执行；写成 `begin atomic ... end` 则这些语句构成一个事务。`declare` 声明变量，`set` 赋值。

### 6.2 循环

```sql
-- while：先判断，条件为真才执行循环体
declare n integer default 0;
while n < 10 do
    set n = n + 1;
end while;

-- repeat：先执行循环体，再判断，条件为真时退出
declare n integer default 10;
repeat
    set n = n - 1;
until n = 0
end repeat;

-- for：对查询结果的每一行执行一次循环体
declare n integer default 0;
for r as
    select balance
    from account
    where branch_name = 'Perryridge'
do
    set n = n + r.balance;
end for;
```

- while 例子：只要 n 小于 10 就加 1，结束时 n 为 10。
- repeat 例子：每次减 1，直到 n 等于 0 才退出，循环体执行 10 次。
- for 例子：`r` 依次指向查询结果的每一行，把 Perryridge 银行所有账户的余额加到 n 上。

while 和 repeat 的区别：while 每轮开始前检查条件，第一次检查就为假时循环体一次也不执行；repeat 先执行循环体再检查，至少执行一次。另外 while 是条件为真时继续，repeat 是条件为真时退出。

整理补充：`leave 标号` 跳出循环，`iterate 标号` 直接开始下一轮。MySQL 支持 while、repeat、loop，不支持 for 循环，逐行处理要用游标。

### 6.3 分支

```sql
if r.balance < 1000 then
    set l = l + r.balance;
elseif r.balance < 5000 then
    set m = m + r.balance;
else
    set h = h + r.balance;
end if;
```

余额小于 1000 的加到 l，1000 到 5000 之间（不含 5000）的加到 m，其余的加到 h。这段通常放在上面的 for 循环里，完整写法见典型题 8。

### 6.4 异常处理

整理补充（笔记没有这部分，教材有）：

```sql
declare out_of_classroom_seats condition;          -- 声明一个异常条件

declare exit handler for out_of_classroom_seats    -- 声明处理程序
begin
    ...
end
```

- 在过程体里用 `signal out_of_classroom_seats` 抛出这个异常，控制转到处理程序。
- `exit` 处理程序执行完后退出所在的 `begin ... end`；`continue` 处理程序执行完后接着执行抛出异常的下一条语句。
- 预定义的条件有 `sqlexception`、`sqlwarning`、`not found`。MySQL 里常写 `declare continue handler for not found set done = 1;`，游标取完时把 done 置 1，见典型题 9。

### 6.5 循环不变式

笔记 5.6.4"系统软件程序设计"讲的是用霍尔逻辑分析程序。写程序时要尽量用好 CPU 时间、内存、磁盘这些资源，也要能说明程序是对的。

`{Q} S {R}` 表示：执行语句 S 之前前置条件 Q 成立，那么 S 执行完后后置条件 R 成立。

循环不变式（loop invariant）是每轮循环开始和结束时都成立的变量关系。对循环 `while (B) S`，如果 I 和 B 同时成立时执行一遍 S 之后 I 仍成立，那么：

$$\{I\}\ \mathrm{while}(B)\ S\ \{I \land \lnot B\}$$

也就是进入循环前 I 成立，退出循环时 I 仍成立，并且循环条件 B 为假。

（更正：原文解释成"进入循环前 I 为真（即 B 为真），退出后 I 仍为真"，漏了"S 要保持 I"这个前提，也把"进入前"和"B 为真"混在一起。）

例：只用加法求两个正整数 a、b 的乘积。

```c
x = a;
y = b;
z = 0;
while (y > 0) {
    y = y - 1;
    z = z + x;
}
```

- 目标：结束时 z = a × b。
- 循环不变式：z + x × y = a × b。
- 进入循环前：z = 0，x = a，y = b，0 + a × b = a × b，成立。
- 执行一轮后：新的 z 是 z + x，新的 y 是 y − 1，(z + x) + x × (y − 1) = z + x × y，所以不变式仍成立。
- 退出时：y 从正整数每次减 1，退出时 y = 0，代入不变式得 z = a × b。

## 7. 递归查询

"找出某人直接和间接管理的所有人""找出某门课的全部先修课"这类问题要求传递闭包。普通查询只能连接固定的次数，层数不固定就写不出来，要用递归查询。

递归查询由两个子查询 union 而成：

- 基查询（base query）：不引用递归视图，是递归的起点；
- 递归查询（recursive query）：引用递归视图本身。

例：从关系 `manager(employee_name, manager_name)` 中查出所有直接和间接的管理关系。

```sql
with recursive empl (employee_name, manager_name) as (
        select employee_name, manager_name
        from manager
    union
        select manager.employee_name, empl.manager_name
        from manager, empl
        where manager.manager_name = empl.employee_name
)
select *
from empl;
```

- `with recursive` 定义一个临时视图，只在这条查询里可用，`recursive` 表示它是递归的。
- 基查询取出 `manager` 里所有直接管理关系。
- 递归查询：某人的直接上级是 X，而 X 在 `empl` 里的上级是 Y，那么 Y 也是这个人的（间接）上级。
- `union` 合并时会去重。

（更正：原文 with 定义后面没有主查询。with 子句后面必须跟一条使用它的查询，这里补上 `select * from empl`。）

执行过程：先算基查询，再反复执行递归查询，把新产生的元组加进 `empl`，直到某一轮不再产生新元组。用一组示例数据走一遍（整理补充）：`manager` 里有 (A, B)、(B, C)、(C, D)，表示 A 的上级是 B，B 的上级是 C，C 的上级是 D。

| 轮次 | 新增元组 | 说明 |
| --- | --- | --- |
| 基查询 | (A, B)、(B, C)、(C, D) | 直接关系 |
| 第 1 轮 | (A, C)、(B, D) | A 的上级 B 的上级是 C；B 的上级 C 的上级是 D |
| 第 2 轮 | (A, D) | A 的上级 B 在 empl 里有上级 D |
| 第 3 轮 | 无 | 停止，`empl` 共 6 个元组 |

限制条件：

- 递归查询必须是**单调的**：递归视图的输入元组变多时，结果只会变多或不变，不会减少。
- 因此递归部分不能用：对递归视图做聚集；在引用递归视图的子查询上用 `not exists`；右边是递归视图的差运算（`except`）。

（更正：原文把单调解释成"不能在递归过程中改变查询结果的排序"，和排序无关。这几种结构被禁止，是因为它们会让"多了元组、结果反而变少"，迭代就可能不收敛。）

另外：

- 标准 SQL 也可以用 `create recursive view` 定义持久的递归视图，代替 `with recursive`。
- 整理补充：MySQL 8.0 起支持 `with recursive`。数据里有环时（比如 A 管 B、B 又管 A），用 `union` 去重能保证停下来；换成 `union all` 会无限循环。

## 易混对比

| 对比项 | 区别 |
| --- | --- |
| 语句级接口 / 调用级接口 | 前者 SQL 写在源码里，要预编译（嵌入式、动态 SQL）；后者 SQL 当字符串传给库函数，不预编译（ODBC、JDBC） |
| 嵌入式 SQL / 动态 SQL | 语句在编译时确定，预编译器处理 / 语句在运行时构造，用 prepare、execute 执行 |
| ODBC / JDBC | 通用的 C 语言 API / Java 的 API |
| `Statement` / `PreparedStatement` | 每次把完整的 SQL 字符串交给数据库 / 先预编译带 `?` 的语句，再填参数，可重复执行，能防注入 |
| `open` / `fetch` | open 执行查询，游标在第一行之前 / fetch 取当前行并前移 |
| 游标 `fetch` / `where current of` | fetch 移动游标并读数据 / where current of 修改或删除游标当前行，不移动游标 |
| 函数 / 过程 | 函数有返回值，写在 SQL 表达式里调用；过程没有返回值，用 out 参数带回结果（可以多个），用 `call` 调用 |
| 标量函数 / 表函数 | 返回一个值 / 返回一张表，调用时写 `table(函数名(参数))` |
| `in` / `out` 参数 | 调用者传给过程 / 过程传回调用者 |
| `while` / `repeat` | 先判断再执行，可能一次都不执行，条件真时继续 / 先执行再判断，至少执行一次，条件真时退出 |
| `exit` / `continue` 处理程序 | 处理完退出所在的复合语句 / 处理完接着往下执行 |
| `with` / `with recursive` | 定义普通临时关系 / 定义可以引用自身的临时关系，迭代到不动点 |
| 递归中用 `union` / `union all` | 去重，有环也能停 / 不去重，有环时停不下来 |

## 典型题与解答

### 题 1：找出嵌入式 SQL 循环里的错误

题目：笔记里的循环如下，指出问题。

```c
EXEC SQL open c END-EXEC
while (strcmp(SQLSTATE, "02000") != 1) {
    EXEC SQL fetch c into :cn, :cc END-EXEC
    printf("%s,%s\n", cn, cc);
}
EXEC SQL close c END-EXEC
```

思路：想清楚 strcmp 的返回值，以及最后一次 fetch 失败之后发生了什么。

解答：

1. `strcmp` 两串相等时返回 0，不相等时返回的是某个非零值，不一定是 1。条件应写 `!= 0`。
2. 最后一次 fetch 没取到数据，变量里还是上一行的值，紧接着的 printf 会把最后一行再打印一次。应在循环前先 fetch，循环体里先打印、后 fetch，见 2.3 节的改正版。

### 题 2：给 Perryridge 银行每个账户加 100 美元

思路：逐行处理用可更新游标，每轮修改当前行之后必须 fetch 下一行；更简单的是一条 update。

解答：游标写法见 2.4 节。一条语句的写法：

```c
EXEC SQL
    update account
    set balance = balance + 100
    where branch_name = 'Perryridge'
END-EXEC
```

### 题 3：用动态 SQL 给运行时输入的账户计息

思路：语句里账号的位置用 `?` 占位，prepare 一次；每读入一个账号 execute 一次。

```c
char *sqlprog = "update account set balance = balance * 1.05 where account_number = ?";
char account[11];

EXEC SQL prepare dynprog from :sqlprog END-EXEC
while (scanf("%10s", account) == 1) {
    EXEC SQL execute dynprog using :account END-EXEC
}
```

账号通过 `using` 作为参数传入，不拼进 SQL 字符串，用户输入 `A-101' or '1'='1` 这类内容也只会被当成一个查不到的账号。

### 题 4：定义函数统计系的教师人数，并找出人数多于 12 的系

思路：函数体用 `select count(*) into` 把结果存进变量再 return；调用写在 where 里。

```sql
create function dept_count (dept_name varchar(20))
    returns integer
begin
    declare d_count integer;
    select count(*) into d_count
    from instructor
    where instructor.dept_name = dept_count.dept_name;
    return d_count;
end

select dept_name, budget
from department
where dept_count(dept_name) > 12;
```

调用处的 `dept_count(dept_name)` 里，`dept_name` 是 `department` 当前元组的属性。

### 题 5：用表函数查 Music 系的全部教师

思路：`returns table (...)` 声明返回表的结构，`return table (select ...)` 给出内容；调用时放在 from 里，外面套 `table(...)`。

```sql
select *
from table(instructor_of('Music'));
```

函数定义见 5.2 节。等价的普通查询是 `select * from instructor where dept_name = 'Music'`，表函数的好处是可以像参数化视图一样复用。

### 题 6：定义过程并调用

题目：定义过程 `dept_count_proc`，输入系名，用输出参数返回教师人数；查询 Physics 系的人数。

思路：`in` 参数传系名，`out` 参数接结果，用 `call` 调用，调用前先声明接收结果的变量。

```sql
create procedure dept_count_proc (in dept_name varchar(20), out d_count integer)
begin
    select count(*) into d_count
    from instructor
    where instructor.dept_name = dept_count_proc.dept_name;
end

declare d_count integer;
call dept_count_proc('Physics', d_count);
```

MySQL 写法见 5.3 节。

### 题 7：while 和 repeat 各执行几次

题目：两段代码执行完后 n 分别是多少？

```sql
declare n integer default 10;
while n < 10 do
    set n = n + 1;
end while;
```

```sql
declare n integer default 10;
repeat
    set n = n + 1;
until n >= 10
end repeat;
```

思路：while 先判断，repeat 先执行。

解答：第一段一开始 `n < 10` 就为假，循环体一次也不执行，n = 10。第二段先执行一次得 n = 11，再判断 `n >= 10` 为真，退出，n = 11。

### 题 8：按余额分档统计总额

题目：写一个过程，把所有账户的余额分三档累加：小于 1000、1000 到 5000（不含 5000）、不少于 5000，用三个输出参数返回。

思路：for 循环遍历 `account`，循环体里用 if-elseif-else 分档。

```sql
create procedure balance_levels (out l numeric(12,2),
                                 out m numeric(12,2),
                                 out h numeric(12,2))
begin
    set l = 0;
    set m = 0;
    set h = 0;
    for r as
        select balance
        from account
    do
        if r.balance < 1000 then
            set l = l + r.balance;
        elseif r.balance < 5000 then
            set m = m + r.balance;
        else
            set h = h + r.balance;
        end if;
    end for;
end
```

这是标准 SQL/PSM 写法。MySQL 没有 for 循环，要改成题 9 那样的游标加 loop。

### 题 9：在 MySQL 里用游标求 Perryridge 银行的余额总和

思路：MySQL 用 `declare ... cursor` 声明游标，用 `not found` 的 continue 处理程序标记取完，`loop` 里 fetch，取完就 `leave`。

```sql
delimiter //
create procedure perryridge_total (out total numeric(12,2))
begin
    declare done int default 0;
    declare b numeric(12,2);
    declare c cursor for
        select balance
        from account
        where branch_name = 'Perryridge';
    declare continue handler for not found set done = 1;

    set total = 0;
    open c;
    read_loop: loop
        fetch c into b;
        if done = 1 then
            leave read_loop;
        end if;
        set total = total + b;
    end loop;
    close c;
end //
delimiter ;

call perryridge_total(@t);
select @t;
```

MySQL 要求声明的顺序是：先变量，再游标，最后处理程序。fetch 取不到数据时触发 `not found`，处理程序把 done 置 1 后继续执行，下一句的 if 就会跳出循环。

### 题 10：求某门课的全部先修课（直接和间接）

题目：`prereq(course_id, prereq_id)` 里有 (CS-347, CS-315)、(CS-315, CS-190)、(CS-190, CS-101)（示例数据），求 CS-347 的全部先修课。

思路：基查询是直接先修关系；递归部分把"已知的先修课"再往前找一层。

```sql
with recursive rec_prereq (course_id, prereq_id) as (
        select course_id, prereq_id
        from prereq
    union
        select rec_prereq.course_id, prereq.prereq_id
        from rec_prereq, prereq
        where rec_prereq.prereq_id = prereq.course_id
)
select prereq_id
from rec_prereq
where course_id = 'CS-347';
```

| 轮次 | 新增元组 |
| --- | --- |
| 基查询 | (CS-347, CS-315)、(CS-315, CS-190)、(CS-190, CS-101) |
| 第 1 轮 | (CS-347, CS-190)、(CS-315, CS-101) |
| 第 2 轮 | (CS-347, CS-101) |
| 第 3 轮 | 无，停止 |

结果：CS-315、CS-190、CS-101。

### 题 11：用循环不变式证明加法求乘积的程序正确

题目：证明 6.5 节的程序结束时 z = a × b。

思路：找一个每轮都成立的式子，再用"退出时循环条件为假"补上最后一步。

解答：取不变式 I：z + x × y = a × b。

1. 初始 x = a，y = b，z = 0，I 成立。
2. 设某轮开始时 I 成立且 y > 0。执行后 y' = y − 1，z' = z + x，有 z' + x × y' = z + x + x × (y − 1) = z + x × y = a × b，I 仍成立。
3. y 是正整数，每轮减 1，循环一定结束，结束时 y > 0 为假，即 y = 0。代入 I 得 z = a × b。

更多练习见期末复习题整理稿（`20` 号起的文件）和 `33 课后习题 第5章 高级SQL.md`。
