# 实验题与LeetCode

本文整理自 `笔记/数据库期末复习题整理.pdf` 第 97–99 页"实验题整理"、第 115 页顶部两道实验题的答案截图，以及第 111–114 页"Leetcode整理"。实验题原稿只列了题目。第 8 题和原稿第 19 页的一段 SQL 是同一题，第 27、28 题在第 115 页有答案截图，其余解答都是整理时补写的，标了"（整理补充）"。

## 使用说明

- 实验题都基于 university 模式，属性和主码见 `25 SQL与关系代数应用题.md` 开头的模式表；自然连接的同名属性陷阱和"所有""没有""输出 0"这类题的套路，也在那份文件的"解题方法"里。
- SQL 按 MySQL 写，用到方言的地方注明了标准 SQL 写法。`except`、`intersect` 要 MySQL 8.0.31 及以上版本才支持，低版本的替代写法附在题后。
- 题目里的"计算机学院""生物学院""张三""李四""匡亚明"等中文取值，要换成实验库里的实际值。第 115 页的答案截图用的是教材样例数据，计算机学院写作 `'Comp. Sci.'`，老师用的是 Brandt。学期按教材写 `'Spring'`、`'Fall'`。
- 原稿有两个第 11 题。这里按原顺序重新连续编号，共 36 题：第 1–11 题编号不变，第 12 题是原稿的第二个 11，第 13–36 题对应原稿第 12–35 题，标题里注明了原编号。

## 实验题

### 1 查询课程表的所有课程信息

题目：从课程表（course）中查询所有课程信息。

思路：`select *` 取全部列。

解答（整理补充）：

```sql
select *
from course;
```

### 2 查询课程名

题目：从课程表（course）中查询课程名。

思路：课程名是 `title`。

解答（整理补充）：

```sql
select title
from course;
```

### 3 从课程段表查询课程名称并去重

题目：从课程段（section）表中查询课程名称，要求消除值相同的那些行。

思路：`section` 里没有课程名，只有 `course_id`，要连 `course` 取 `title`。一门课可以开很多个课程段，所以要 `distinct`。

解答（整理补充）：

```sql
select distinct title
from section natural join course;
```

`section` 和 `course` 只有 `course_id` 同名，自然连接没问题。如果题目的"课程名称"指的是课程号，写 `select distinct course_id from section;`。

### 4 学生表只显示前 6 行

题目：从学生（student）表中查询所有的信息，要求只显示查询结果的前 6 行数据。

思路：MySQL 用 `limit` 限制行数。

解答（整理补充）：

```sql
select *
from student
limit 6;
```

标准 SQL 写法是 `select * from student fetch first 6 rows only;`，MySQL 不支持这种写法。没有 `order by` 时"前 6 行"是数据库碰巧返回的顺序，要固定顺序就先 `order by ID`。

### 5 选了计算机学院所有课程的学生姓名

题目：查询选了所有计算机学院开设课程的学生的姓名。

思路："所有"是除法：计算机学院开的全部课程减去这个学生选过的课程，差为空就符合。

解答（整理补充）：

```sql
select S.name
from student as S
where not exists (
    (select course_id from course where dept_name = '计算机学院')
    except
    (select T.course_id from takes as T where T.ID = S.ID));
```

不支持 `except` 时写两层 `not exists`：

```sql
select S.name
from student as S
where not exists (
    select *
    from course as C
    where C.dept_name = '计算机学院'
      and not exists (select *
                      from takes as T
                      where T.ID = S.ID and T.course_id = C.course_id));
```

### 6 2019 年春季开课但 2018 年不开课的课程编号

题目：查询 2019 年春季开课，但 2018 年不开课的课程的编号。

思路：2019 年春季的课程号减去 2018 年任一学期开过的课程号。

解答（整理补充）：

```sql
select distinct course_id
from section
where semester = 'Spring' and year = 2019
  and course_id not in (select course_id
                        from section
                        where year = 2018);
```

### 7 计算机学院学生距离 80 学分还差多少

题目：假设毕业要求为修够 80 学分，请统计计算机学院内学生距离毕业要求还差多少学分，并按所差分数的升序排列。

思路：`select` 里直接算 `80 - tot_cred`，起别名后在 `order by` 里用。

解答（整理补充）：

```sql
select ID, name, 80 - tot_cred as cred_gap
from student
where dept_name = '计算机学院'
order by cred_gap asc;
```

已经修满的学生差值是 0 或负数。只看还没修够的，在 `where` 里加 `and tot_cred < 80`。

### 8 2019 年春季课程段选课人数的最大值

题目：统计 2019 年春季所开课程段选课人数的最大值。

思路：先按课程段分组计数，再对计数结果求最大值。原稿第 19 页给过一个答案，表名、属性名和分组都有错，更正过程见 `25 SQL与关系代数应用题.md` 的最后一节。

解答：

```sql
select max(enrollment) as max_enrollment
from (select course_id, sec_id, count(ID) as enrollment
      from takes
      where semester = 'Spring' and year = 2019
      group by course_id, sec_id) as sec_enrollment;
```

### 9 各学院老师的平均年薪

题目：统计各个学院老师的平均年薪。

思路：按 `dept_name` 分组求 `avg(salary)`。

解答（整理补充）：

```sql
select dept_name, avg(salary) as avg_salary
from instructor
group by dept_name;
```

### 10 没有选 2019 年春季课程段的学生姓名

题目：请输出没有选择 2019 年春季开课的课程段的学生的姓名。

思路：全部学生里去掉 2019 年春季在 `takes` 里出现过的学生。

解答（整理补充）：

```sql
select name
from student
where ID not in (select ID
                 from takes
                 where semester = 'Spring' and year = 2019);
```

### 11 张三指导的学生 2019 年春季获得的总学分

题目：请输出"张三"指导的学生在 2019 年春季开设课程段中所获得的总学分。（我们默认只要选择某个 course 的一个 section，就可以获得这个 course 的学分；若选择同一个 course 的多个 section，也只获得 1 次这个 course 的学分）

思路：`instructor` 按姓名找到张三，经 `advisor` 找到他指导的学生。先把这些学生 2019 年春季的选课按 `(ID, course_id)` 去重，保证同一门课只算一次，再连 `course` 按学生求学分和。

解答（整理补充）：

```sql
select S.ID, S.name, sum(C.credits) as total_credits
from instructor as I
     join advisor as A on A.i_ID = I.ID
     join student as S on S.ID = A.s_ID
     join (select distinct ID, course_id
           from takes
           where semester = 'Spring' and year = 2019) as T on T.ID = S.ID
     join course as C on C.course_id = T.course_id
where I.name = '张三'
group by S.ID, S.name;
```

`student` 和 `instructor` 同有 `ID`、`name`、`dept_name`，这里全部用 `on` 写明连接条件，不能用自然连接。2019 年春季一门课都没选的学生不会出现在结果里；要让他们显示 0，把连 `T` 和 `course` 的两个 `join` 改成 `left join`，再用 `coalesce(sum(C.credits), 0)`。

### 12（原稿第二个 11）数据库系统原理和离散数学的共同先修课程

题目：请输出"数据库系统原理"和"离散数学"的共同的先修课程的 ID。

思路：分别查出两门课的先修课程号，再取交集。`prereq` 里只有课程号，课程名要连 `course` 查。

解答（整理补充）：

```sql
(select P.prereq_id
 from prereq as P join course as C on P.course_id = C.course_id
 where C.title = '数据库系统原理')
intersect
(select P.prereq_id
 from prereq as P join course as C on P.course_id = C.course_id
 where C.title = '离散数学');
```

不支持 `intersect` 时：

```sql
select distinct P.prereq_id
from prereq as P join course as C on P.course_id = C.course_id
where C.title = '数据库系统原理'
  and P.prereq_id in (select P2.prereq_id
                      from prereq as P2 join course as C2 on P2.course_id = C2.course_id
                      where C2.title = '离散数学');
```

这样查到的是直接先修课，先修课的先修课不算在内。

### 13（原 12）2016–2018 年计算机学院每年开设的课程段数量

题目：请统计 2016 年–2018 年，计算机学院每年开设的课程段的数量。

思路：课程段属于哪个学院看 `course.dept_name`。`section` 连 `course`，按年份分组计数。

解答（整理补充）：

```sql
select S.year, count(*) as sec_count
from section as S join course as C on S.course_id = C.course_id
where C.dept_name = '计算机学院'
  and S.year between 2016 and 2018
group by S.year;
```

某一年一个课程段都没开时，这一年不会出现在结果里。

### 14（原 13）课程编号不是 004、007、013 的课程

题目：查询课程编号不为"004""007""013"的课程编号和课程名称。

思路：`not in` 后面跟常量列表。

解答（整理补充）：

```sql
select course_id, title
from course
where course_id not in ('004', '007', '013');
```

### 15（原 14）课程名以 D 开头、以 e 结尾的课程

题目：查询课程名以字母 D 开始，以"e"结尾的课程信息。

思路：`like` 里 `%` 匹配任意多个字符（包括 0 个）。

解答（整理补充）：

```sql
select *
from course
where title like 'D%e';
```

MySQL 常用的默认排序规则不区分大小写，`d` 开头、`E` 结尾的也会查出来。要严格区分大小写，需要指定二进制排序规则，例如 `title like 'D%e' collate utf8mb4_bin`（表的字符集是 utf8mb4 时）。

### 16（原 15）课程名中间含"制作"的课程

题目：查询课程名以"制作"两字作为中间字的课程信息。（要求"制作"不做开头和结尾）

思路：`_` 恰好匹配一个字符。`'_%制作%_'` 要求"制作"前后至少各有一个字。

解答（整理补充）：

```sql
select *
from course
where title like '_%制作%_';
```

也可以把条件拆开写：`title like '%制作%' and title not like '制作%' and title not like '%制作'`。两种写法只在"制作"出现多次的课程名上有区别，比如"制作与视频制作入门"：中间那个"制作"前后都有字，第一种能查出来；它又以"制作"开头，第二种查不出来。

### 17（原 16）姓名第二个字为"宝"的学生

题目：查询姓名第二个字为"宝"的学生信息。

思路：第一个位置用 `_` 占一个字，后面用 `%`。

解答（整理补充）：

```sql
select *
from student
where name like '_宝%';
```

MySQL 的 `_` 按字符匹配，一个汉字算一个字符。

### 18（原 17）不姓刘的学生

题目：查询不姓"刘"的学生信息。

思路：`not like` 排除以"刘"开头的姓名。

解答（整理补充）：

```sql
select *
from student
where name not like '刘%';
```

### 19（原 18）2018 年至少有两个课程段的课程 ID

题目：查询那些在 2018 年有至少两个课程段的课程的 ID。

思路：`section` 的主码是 `(course_id, sec_id, semester, year)`，一行就是一个课程段。按 `course_id` 分组，分组后的条件写在 `having`。

解答（整理补充）：

```sql
select course_id
from section
where year = 2018
group by course_id
having count(*) >= 2;
```

### 20（原 19）计算机学院学生选了外院老师开的本院课程

题目：查询计算机学院学生选了非本学院老师开设的属于本学院的课程的情况，统计这些学生的 ID、姓名、总学分。

思路：三个条件：学生属于计算机学院，课程属于计算机学院，给这个课程段上课的老师不属于计算机学院。`takes` 和 `teaches` 要按课程段的四个属性对上，老师再连 `instructor` 取学院。"总学分"按 `student.tot_cred` 理解。

解答（整理补充）：

```sql
select distinct S.ID, S.name, S.tot_cred
from student as S
     join takes as T on T.ID = S.ID
     join course as C on C.course_id = T.course_id
     join teaches as E on E.course_id = T.course_id
                      and E.sec_id = T.sec_id
                      and E.semester = T.semester
                      and E.year = T.year
     join instructor as I on I.ID = E.ID
where S.dept_name = '计算机学院'
  and C.dept_name = '计算机学院'
  and I.dept_name <> '计算机学院';
```

这题的连接不能偷懒写成自然连接：`takes` 和 `teaches` 的 `ID` 一个是学生号一个是老师号，`student` 和 `instructor` 的 `name`、`dept_name` 也同名，自然连接会把它们当成相等条件。

### 21（原 20）2019 年春季选课人数在 15 到 25 之间的课程段

题目：查询 2019 年春季开设的，选课人数少于 25 并且多于 15 人的课程段信息。

思路：先在 `takes` 里按课程段分组计数，作为派生表，再连回 `section` 取课程段的全部信息。

解答（整理补充）：

```sql
select S.*
from section as S
     join (select course_id, sec_id, semester, year, count(*) as enrollment
           from takes
           where semester = 'Spring' and year = 2019
           group by course_id, sec_id, semester, year) as E
     using (course_id, sec_id, semester, year)
where E.enrollment > 15 and E.enrollment < 25;
```

### 22（原 21）2018 年由非本学院开设的课程段总数

题目：查询 2018 年由非本学院开设的课程段的总数。如果没有，输出结果为 0。

思路："非本学院开设"按"授课老师的学院和课程所属学院不同"理解。一个课程段可能有几位老师，先对课程段去重再计数。外层 `count` 不带 `group by`，没有符合条件的课程段时返回 0。

解答（整理补充）：

```sql
select count(*) as sec_count
from (select distinct E.course_id, E.sec_id, E.semester, E.year
      from teaches as E
           join instructor as I on I.ID = E.ID
           join course as C on C.course_id = E.course_id
      where E.year = 2018
        and I.dept_name <> C.dept_name) as X;
```

如果"本学院"指的是计算机学院，统计 2018 年非计算机学院课程的课程段数，写成：

```sql
select count(*) as sec_count
from section as S join course as C on S.course_id = C.course_id
where S.year = 2018
  and C.dept_name <> '计算机学院';
```

### 23（原 22）报名人数大于 25 或少于 15 的课程

题目：查询报名人数大于 25 或者少于 15 人的课程信息，要求查询结果按照报名人数降序排列。

思路：`course` 左外连接 `takes`，按课程分组数人数，没人报名的课程人数为 0，也满足"少于 15"。同一个学生可能重修同一门课，用 `count(distinct T.ID)` 数人头。

解答（整理补充）：

```sql
select C.course_id, C.title, C.dept_name, C.credits,
       count(distinct T.ID) as enrollment
from course as C left join takes as T on C.course_id = T.course_id
group by C.course_id, C.title, C.dept_name, C.credits
having count(distinct T.ID) > 25 or count(distinct T.ID) < 15
order by enrollment desc;
```

### 24（原 23）与计算机学院同楼的其他学院老师的平均工资

题目：查询与计算机学院同处于一座大楼的其他学院的老师的平均工资。

思路：学院所在大楼在 `department.building`。先查出计算机学院的大楼，再找同楼的其他学院，最后对这些学院的老师求平均工资。

解答（整理补充）：

```sql
select avg(I.salary) as avg_salary
from instructor as I join department as D on I.dept_name = D.dept_name
where D.dept_name <> '计算机学院'
  and D.building in (select building
                     from department
                     where dept_name = '计算机学院');
```

要分学院列出，在最后加 `group by D.dept_name`，并把 `D.dept_name` 加进 `select`。

### 25（原 24）匡亚明大楼老师的工资涨到 1.5 倍

题目：给在"匡亚明"大楼内办公的老师的工资增加到原来工资的 1.5 倍。

思路：`instructor` 里没有办公楼，老师的办公楼按所在学院的 `department.building` 算，用子查询找出楼里的学院。

解答（整理补充）：

```sql
update instructor
set salary = salary * 1.5
where dept_name in (select dept_name
                    from department
                    where building = '匡亚明');
```

子查询只读 `department`，没有读被更新的 `instructor`，MySQL 可以直接执行。

### 26（原 25）各学院的学生数量

题目：统计各个学院的学生的数量。

思路：从 `department` 左外连接 `student`，没有学生的学院也显示 0。

解答（整理补充）：

```sql
select D.dept_name, count(S.ID) as student_count
from department as D left join student as S on D.dept_name = S.dept_name
group by D.dept_name;
```

`count(S.ID)` 不计 null，没有学生的学院得 0；写成 `count(*)` 会得到 1。只统计有学生的学院，直接 `select dept_name, count(*) from student group by dept_name;`。

### 27（原 26）计算机学院总学分前 10 名的学生

题目：统计计算机学院中所获总学分排名前 10 位的学生的信息。

思路：筛出计算机学院的学生，按 `tot_cred` 降序排序，取前 10 行。

解答：

原答案（第 115 页截图）：

```sql
select *
from
(select *
from student
where dept_name='Comp.Sci'
order by tot_cred desc)
as comp_order limit 10;
```

截图里派生表别名 `comp_order` 和 `limit 10` 都写在括号外面，这条语句在 MySQL 里能执行。有两处要改：

（更正：原文把 `order by tot_cred desc` 写在 `from` 后面的子查询里。标准 SQL 规定子查询的结果没有顺序，外层查询不保证保留这个排序，有的数据库还直接不允许子查询里单独写 `order by`。MySQL 只在外层查询很简单时才会沿用子查询的排序，写法不可靠，应把 `order by` 和 `limit` 都放到最外层。）

原文的条件是 `dept_name='Comp.Sci'`，教材样例数据里计算机系写作 `'Comp. Sci.'`（中间有空格，末尾有点），照原文写在样例数据上查不到行，要按自己库里的实际取值写。

更正后：

```sql
select *
from student
where dept_name = 'Comp. Sci.'
order by tot_cred desc
limit 10;
```

标准 SQL 写法是把 `limit 10` 换成 `fetch first 10 rows only`。第 10 名有并列时，`limit` 留下并列者中的哪几个是不确定的；标准 SQL 的 `fetch first 10 rows with ties` 会把并列的都保留，MySQL 不支持。

### 28（原 27）某老师 2017–2019 年开设课程段的数量

题目：查询"李四"老师在 2017–2019 年开设课程段的数量。（第 115 页截图里这题写的是 Brandt 老师）

思路：`teaches` 的一行就是某位老师教的一个课程段，连 `instructor` 按姓名筛选，按年份范围计数。

解答：

原答案（第 115 页截图）：

```sql
select count(*)
from teaches natural join section join instructor using (ID)
where name='Brandt' and year in ('2017','2018','2019');
```

逐项核对：

- `teaches natural join section` 的公共属性是 `course_id, sec_id, semester, year`，正好按课程段连接，结果正确。但 `teaches` 本身就有 `year`，这里不需要连 `section`。如果按原卷模式把 `teaches` 的属性写成 `section_id`，这个自然连接就只比较 `course_id, semester, year`，同一门课同一学期的不同课程段会交叉配对，计数偏大。
- `join instructor using (ID)` 只按 `ID` 连接，老师号对老师号，正确。`name` 只在 `instructor` 里有，不加前缀也不会有歧义。
- `year in ('2017','2018','2019')` 把年份写成了字符串。`year` 是数值型，MySQL 会隐式转换，所以能查出正确结果，但应该写数值，连续年份用 `between` 更简洁。

原答案的结果是对的，下面是简化后的写法：

```sql
select count(*) as sec_count
from teaches as T join instructor as I on T.ID = I.ID
where I.name = 'Brandt'
  and T.year between 2017 and 2019;
```

按题目换成 `I.name = '李四'` 即可。有两位同名老师时，这句会把两人的课程段加在一起；要分开统计就按 `I.ID` 分组。

### 29（原 28）工资介于生物学院最低和最高之间的计算机学院老师

题目：查询计算机学院老师中比生物学院工资最高的老师工资低，但是比生物学院最低工资高的老师的信息。

思路：两个标量子查询分别求生物学院的最高和最低工资，外层比较。

解答（整理补充）：

```sql
select *
from instructor
where dept_name = '计算机学院'
  and salary < (select max(salary) from instructor where dept_name = '生物学院')
  and salary > (select min(salary) from instructor where dept_name = '生物学院');
```

也可以写成 `salary < some (...)` 和 `salary > some (...)`，子查询里查生物学院全部老师的工资。

### 30（原 29）2019 年计算机学院比生物学院多开设几个课程段

题目：请统计 2019 年计算机学院比生物学院多开设了几个课程段。

思路：两个学院各数一次 2019 年的课程段数，在 `select` 里相减。课程段归属看 `course.dept_name`。

解答（整理补充）：

```sql
select
    (select count(*)
     from section as S join course as C on S.course_id = C.course_id
     where C.dept_name = '计算机学院' and S.year = 2019)
  - (select count(*)
     from section as S join course as C on S.course_id = C.course_id
     where C.dept_name = '生物学院' and S.year = 2019) as diff;
```

MySQL 允许没有 `from` 的 `select`，Oracle 要在末尾加 `from dual`。结果为负数表示生物学院开得更多。

### 31（原 30）2019 年春季没有学生选的课程段数量

题目：请统计 2019 年春季，没有学生选课的课程段的数量。若没有，请输出 0。

思路：对 2019 年春季的每个课程段，检查 `takes` 里不存在对应的选课记录。外层 `count(*)` 不分组，一个都没有时返回 0。

解答（整理补充）：

```sql
select count(*) as empty_sec_count
from section as S
where S.semester = 'Spring' and S.year = 2019
  and not exists (select *
                  from takes as T
                  where T.course_id = S.course_id
                    and T.sec_id = S.sec_id
                    and T.semester = S.semester
                    and T.year = S.year);
```

### 32（原 31）计算机学院学生的指导老师姓名

题目：请输出计算机学院学生的指导老师的姓名，去掉重复信息。

思路：`student` 经 `advisor` 连 `instructor`，按学生学院筛选，`distinct` 去重。

解答（整理补充）：

```sql
select distinct I.name
from student as S
     join advisor as A on A.s_ID = S.ID
     join instructor as I on I.ID = A.i_ID
where S.dept_name = '计算机学院';
```

`student` 和 `instructor` 直接自然连接会按 `ID`、`name`、`dept_name` 三个属性比较，结果基本为空，这里必须经过 `advisor` 并写明连接条件。

### 33（原 32）低于全校平均预算的学院预算提高到 1.2 倍

题目：将低于整个学校的平均预算的各个学院的预算提高到原来的 1.2 倍。

思路：`where` 里用子查询求平均预算。标准 SQL 先找出所有要改的元组再更新，平均值用的是更新前的值。

解答（整理补充）：

```sql
update department
set budget = budget * 1.2
where budget < (select avg(budget) from department);
```

MySQL 不允许 `update` 的子查询直接读被更新的表，会报错 1093。把平均值包进派生表：

```sql
update department
set budget = budget * 1.2
where budget < (select avg_budget
                from (select avg(budget) as avg_budget from department) as t);
```

### 34（原 33）李四老师指导的学生数量

题目：统计"李四"老师指导的学生数量。若没有，请输出 0。

思路：`instructor` 按姓名连 `advisor`，数 `s_ID`。不写 `group by` 时，连接结果为空，`count` 返回 0，满足"没有输出 0"。

解答（整理补充）：

```sql
select count(A.s_ID) as student_count
from instructor as I join advisor as A on A.i_ID = I.ID
where I.name = '李四';
```

### 35（原 34）选了数据库原理所有直接先修课的学生

题目：查询已经选了"数据库原理"这门课的所有直接先修课程的学生的信息。

思路：除法。数据库原理的全部直接先修课，减去这个学生选过的课，差为空就符合。

解答（整理补充）：

```sql
select *
from student as S
where not exists (
    (select P.prereq_id
     from prereq as P join course as C on P.course_id = C.course_id
     where C.title = '数据库原理')
    except
    (select T.course_id
     from takes as T
     where T.ID = S.ID));
```

不支持 `except` 时：

```sql
select *
from student as S
where not exists (
    select *
    from prereq as P join course as C on P.course_id = C.course_id
    where C.title = '数据库原理'
      and not exists (select *
                      from takes as T
                      where T.ID = S.ID and T.course_id = P.prereq_id));
```

如果这门课没有先修课，除数为空，所有学生都会被查出来，这是除法本身的性质。

### 36（原 35）2019 年春季选了自己导师开的课程段的学生

题目：查询 2019 年春季选了自己指导教师开设的课程段的学生的姓名、指导老师的姓名和课程段的 ID。

思路：学生经 `advisor` 找到导师；学生的选课记录 `takes` 和导师的授课记录 `teaches` 按课程段四个属性相等，并且授课人就是导师。

解答（整理补充）：

```sql
select S.name as student_name, I.name as advisor_name,
       T.course_id, T.sec_id
from student as S
     join advisor as A on A.s_ID = S.ID
     join instructor as I on I.ID = A.i_ID
     join takes as T on T.ID = S.ID
     join teaches as E on E.ID = I.ID
                      and E.course_id = T.course_id
                      and E.sec_id = T.sec_id
                      and E.semester = T.semester
                      and E.year = T.year
where T.semester = 'Spring' and T.year = 2019;
```

课程段由 `(course_id, sec_id, semester, year)` 确定，学期和年份已经固定，输出 `course_id` 和 `sec_id` 就能区分。

## LeetCode

原稿整理了 5 道 LeetCode 的 SQL 入门题，都用 MySQL 提交。题目描述下面用中文转述，完整题面和示例去 LeetCode 上看。

### 1757 可回收且低脂的产品

题目：`Products` 表有 `product_id`（主键）、`low_fats`、`recyclable` 三列，后两列取值 `'Y'` 或 `'N'`。找出既是低脂又可回收的产品编号，结果顺序不限。

思路：两个条件同时成立，用 `and`。

解答：

```sql
SELECT product_id
FROM Products
WHERE low_fats = 'Y' and recyclable = 'Y'
```

### 584 寻找用户推荐人

题目：`Customer` 表有 `id`（主键）、`name`、`referee_id`（推荐人的 id，可以为空）。找出推荐人不是 id = 2 的客户姓名，没有推荐人的客户也要算上，结果顺序不限。

思路：这题考 null 的判断。`referee_id` 为 null 时，`referee_id <> 2` 的结果是 unknown，`where` 只保留结果为 true 的行，这些客户会被漏掉，所以要单独加 `is null`。

解答：

```sql
SELECT name
FROM Customer
WHERE referee_id <> 2 OR referee_id IS NULL
```

也可以先把 null 换成一个不可能出现的值再比较：`WHERE coalesce(referee_id, 0) <> 2`。

### 595 大的国家

题目：`World` 表有 `name`（主键）、`continent`、`area`、`population`、`gdp`。面积至少 300 万平方公里，或者人口至少 2500 万的国家算大国，输出大国的名称、人口和面积。

思路：两个条件满足一个就行，用 `or`。原稿提醒要看清输出哪几列、按什么顺序：题目要求按 `name, population, area` 的顺序输出，和表里的列顺序不同。"至少"包含等于，用 `>=`。

解答：

```sql
SELECT name, population, area
FROM World
WHERE area >= 3000000 OR population >= 25000000
```

### 1148 文章浏览 I

题目：`Views` 表有 `article_id`、`author_id`、`viewer_id`、`view_date`，没有主键，可能有重复行；同一个人的 `author_id` 和 `viewer_id` 相同。找出所有浏览过自己文章的作者，结果列名为 `id`，按 `id` 升序。

思路：浏览过自己的文章就是 `author_id = viewer_id`。一个作者可能看过自己好几篇文章，表里也有重复行，要 `distinct`；列名用 `as` 改成 `id`；最后 `order by`。原稿把这题标注为 distinct、更名运算、order by 三个知识点。

解答：

```sql
select distinct author_id as id
from views
where author_id = viewer_id
order by id
```

### 1683 无效的推文

题目：`Tweets` 表有 `tweet_id`（主键）和 `content`。推文内容的字符数严格大于 15 时为无效推文，查出所有无效推文的编号，结果顺序不限。

思路：MySQL 里求字符串长度有两个函数。`char_length(str)` 返回字符个数；`length(str)` 返回字节数，一个字符可能占多个字节。按原稿的例子，`'¥'` 的 `char_length` 是 1，`length` 是 2；utf8mb4 下一个汉字的 `length` 是 3。这题的 `content` 只有英文字符，两个函数结果相同，但按题意"字符数"应该用 `char_length`。"严格大于"用 `>`。

解答：

```sql
select tweet_id
from Tweets
where CHAR_LENGTH(content) > 15
```
