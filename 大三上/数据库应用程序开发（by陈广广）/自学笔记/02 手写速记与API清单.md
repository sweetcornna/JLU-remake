# 手写速记与 API 清单

前半部分整理自 `手写代码模板题.pdf`（11 页手写笔记），按场景归成最小代码片段，原文的错误在片段里用 `// 更正：` 标出，批注照录在注释里。后半部分的方法表、常量表、SQLSTATE 表是整理时按 JDBC 接口和 DB2 错误码补的，并和 `数据库程序答题模板.pdf` 对照过。完整程序见 `01 JDBC答题模板（任务1-11）.md`。

片段里的表名统一写成 `Employee`（原文作 `Employ`），和 `01` 保持一致。

## 一、场景速记

### 1. 读键盘输入

```java
BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
String empno = reader.readLine();
int age = Integer.parseInt(reader.readLine());   // 读整数
```

`new InputStreamReader(System.in)` 手写版标了红。要 `import java.io.*;`。

### 2. 公共开头：加载驱动、返回连接

手写版第 2 页"公共开头"：

```java
import java.sql.*;
import java.io.*;
import java.util.*;
import java.math.*;

public class EmployeeDatabase {
    static {
        try {
            Class.forName("COM.ibm.db2.jdbc.app.DB2Driver");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    static Connection getConnection() throws Exception {
        String url = "jdbc:db2:sample";
        String user = "db2admin";
        String passwd = "db2admin";
        return DriverManager.getConnection(url, user, passwd);
    }
}
```

别的程序里用 `Connection con = EmployeeDatabase.getConnection();` 拿连接。返回类型 `Connection` 和 `throws Exception` 手写版描了红。

### 3. 命令行参数两种连接

手写版第 1 页（标题写的 plus）：

```java
if (args.length == 0) {
    con = DriverManager.getConnection("jdbc:db2:sample");          // 默认连接
} else if (args.length == 2) {
    String user = args[0];
    String passwd = args[1];
    String url = "jdbc:db2:sample";
    con = DriverManager.getConnection(url, user, passwd);          // 用户名密码连接
} else {
    throw new Exception("Java MyJDBC[,user,password]");
}
```

### 4. 程序框架

手写版第 3 页：

```java
import java.sql.*;
import java.io.*;
import java.util.*;
import java.math.*;

public class Project {
    static {
        try {
            Class.forName("COM.ibm.db2.jdbc.app.DB2Driver");
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(1);
        }
    }   // 更正：原文漏了 static 块的右括号，文件末尾也少一个右括号，这里补齐

    public static void main(String[] args) throws Exception {
        try {
            Connection con = DriverManager.getConnection("jdbc:db2:sample");
            // ……具体操作，在这里声明 stmt、rs 等
            rs.close();
            stmt.close();
            con.close();
        } catch (Exception e) {
            e.printStackTrace();
            System.exit(1);
        }
    }
}
```

关闭顺序和打开顺序相反：先 `rs`，再 `stmt`，最后 `con`。

### 5. 事务

```java
con.setAutoCommit(false);   // 关闭自动提交
con.commit();               // 手动提交
con.rollback();             // 回滚
```

### 6. 预处理语句和常用 SQL

手写版红字："`setString(` 别忘记先写数字再赋值"。

```java
String sql = "INSERT INTO Employee(empno, name, age, salary) VALUES(?, ?, ?, ?)";
PreparedStatement pstmt = con.prepareStatement(sql);
pstmt.setString(1, empno);   // 第 1 个实参是 ? 的序号，从 1 开始
pstmt.setString(2, name);
pstmt.setInt(3, age);
pstmt.setDouble(4, salary);
int count = pstmt.executeUpdate();   // 返回影响的行数
```

手写版第 1 页抄的三种语句：

```sql
DELETE FROM Employee WHERE empno = ?
INSERT INTO Employee(empno, name, age, salary) VALUES(?, ?, ?, ?)
INSERT INTO Employee(empno, name, age, salary) SELECT empno, name, age, salary FROM ...
```

第三种是子查询插入，`FROM` 后面接另一张表和条件，原文没写完。

### 7. 批处理

```java
// Statement 的批：放进去的是整条 SQL
stmt.addBatch(sql);
// ……
int[] counts = stmt.executeBatch();

// 预处理的批：放进去的是当前这组参数
PreparedStatement pstmt = con.prepareStatement(sql);
pstmt.setString(1, empno);   // 更正：原文作 pstmt.setString(empno)，漏了参数序号
pstmt.addBatch();
// ……
int[] rowcounts = pstmt.executeBatch();
```

`executeBatch()` 返回 `int[]`，每个元素是一条语句影响的行数。

### 8. 左对齐输出

```java
String blanks = "          ";   // 至少 10 个空格
System.out.println(str + blanks.substring(0, 10 - str.length()));
```

`str` 后面补 `10 - str.length()` 个空格，每列占满 10 个字符，多列输出时就对齐了。答题模板第 18 页也手写了这一句（变量名是 `blank`、`s`）。`str` 超过 10 个字符时 `substring` 的结束位置是负数，会抛 `StringIndexOutOfBoundsException`。同样的效果也可以写 `String.format("%-10s", str)`。

### 9. 读入简历、图片文件

手写版第 1 页的这两段作者划掉了，旁边写"看后面的"，完整用法见下面第 12 到 14 条。

```java
String path = reader.readLine();
FileReader resumeReader = path.isEmpty() ? null : new FileReader(path);     // 简历，文本
FileInputStream photo = path.isEmpty() ? null : new FileInputStream(path);  // 图片，二进制（原文漏了变量名，这里补作 photo）
```

### 10. 结果集重定位（重启点）

手写版第 4 页，只写到读取重启点：

```java
Connection con = EmployeeDatabase.getConnection();
try {
    String MinValue = "";   // 全部更新完之后用于将重启点重置
    String abValue = null;  // 更正：原文作 String abValue; 没有初值，后面 setString(1, abValue) 会编译报错
    String sql = "SELECT empno FROM RESTART";
    Statement stmt = con.createStatement();
    ResultSet rs = stmt.executeQuery(sql);
    while (rs.next()) {     /* 实现重启点的读取 */
        abValue = rs.getString(1);
    }
    rs.close();
    stmt.close();
    // 原文写到 String sql2 = "SELECT 为止，后续见 01 第 11 题
```

要点：查询按 `WHERE empno > ? ORDER BY empno` 从重启点往后读；每 50 次更新把当前编号写回 RESTART 并 `commit()`；全部处理完把重启点重置成 `MinValue`。

### 11. 游标定位更新

手写版第 8 页：

```java
String sqlSearch = "SELECT name, age FROM Employee FOR UPDATE";
String sqlUpdate = "UPDATE Employee SET name = ? WHERE CURRENT OF ";   // OF 后面必须留空格，原文 OF 和引号之间看不出空格
Statement stmt = con.createStatement();
ResultSet rs = stmt.executeQuery(sqlSearch);
String cursor = rs.getCursorName();                                     // 获取游标名称
PreparedStatement pstmt = con.prepareStatement(sqlUpdate + cursor);
while (rs.next()) {
    String name = rs.getString(1);
    int age = rs.getInt(2);
    if (age > 20) {
        String newName = "Joy";
        pstmt.setString(1, newName);
        pstmt.executeUpdate();   // 改的是游标当前所在的行
    }
}
```

`sqlUpdate + cursor` 手写版画了波浪线。三步记住：`FOR UPDATE` 打开游标，`getCursorName()` 取名字，`WHERE CURRENT OF 名字` 定位。

### 12. 大文件：CLOB 读取

手写版第 5 页。"1、文件的插入"标题下面原文是空白，只有"2、文件的读取"：

```java
String sql = "SELECT SUBSTR(resume, 1, ?) FROM Employee "
           + "WHERE empno = ?";         // 更正：原文 "... FROM Employ" + "WHERE ..." 拼接处没有空格，会变成 EmployWHERE
PreparedStatement pstmt = con.prepareStatement(sql);
pstmt.setInt(1, startper1);             // 这个值用 SELECT POSSTR(resume, 'Personal') 查出位置再减 1
pstmt.setString(2, empno);
ResultSet rs = pstmt.executeQuery();
while (rs.next()) {
    Clob resumelob = rs.getClob(1);     // 更正：原文作 getClob(1)，漏了 rs.
    int len1 = (int) resumelob.length();               // Clob 对象
    String Patti = resumelob.getSubString(1, len1);    // getSubString 读取中间文件
    System.out.println("Patti开头的部分" + Patti);
}
```

- `resume` 是 Clob 类型的列（手写版蓝字批注）。
- `SUBSTR(列, 起始, 长度)`：三个参数时第三个是长度；两个参数时从起始位置取到末尾。（更正：原文批注作"2个数值：起始→结束"。）
- `POSSTR(列, '串')` 返回该串第一次出现的位置，从 1 数，找不到返回 0。原文批注写的是 `POSSTR(resume,'Per')`，`Per` 太短容易撞上别的单词，按题目应查 `'Personal'`。
- `Clob.getSubString(long pos, int length)` 的 `pos` 从 1 开始；`length()` 返回 `long`，要强转 `int`。
- `SUBSTR`、`POSSTR` 是 DB2 的函数，MySQL 对应 `SUBSTRING`、`LOCATE`，SQL Server 对应 `SUBSTRING`、`CHARINDEX`。
- 写入 CLOB（原文空白的"文件的插入"）：`pstmt.setCharacterStream(i, reader, (int) file.length())`；JDBC 4.0 驱动也可以 `pstmt.setClob(i, reader)`，见 `01` 第 3 题。

### 13. 图像：BLOB 插入

手写版第 6 页"3. 图像的插入"：

```java
String sql = "UPDATE Employee SET photo = ? WHERE empno = ?";
PreparedStatement pstmt = con.prepareStatement(sql);
String empno = reader.readLine();
String path = reader.readLine();
File file = new File(path);                                                      // 首先创建文件类型
BufferedInputStream in = new BufferedInputStream(new FileInputStream(file));   // 创建流对象，传入的是文件类型
pstmt.setBinaryStream(1, in, (int) file.length());                              // 参数赋值（位置，流对象，文件长度）
pstmt.setString(2, empno);
pstmt.executeUpdate();
in.close();                                                                      // 整理补充：执行完关闭输入流
```

`setBinaryStream` 的三个参数：`?` 的位置、流对象、文件长度。`file.length()` 返回 `long`，强转 `int`。

### 14. 图像：BLOB 读取

手写版第 6 页"4. 图像的读取"：

```java
String sql = "SELECT photo FROM Employee WHERE empno = ?";
PreparedStatement pstmt = con.prepareStatement(sql);
String empno = reader.readLine();
String path = reader.readLine();
pstmt.setString(1, empno);
ResultSet rs = pstmt.executeQuery();
while (rs.next()) {
    Blob blob = rs.getBlob(1);                           // 读取 Blob 对象（原文作 (Blob) rs.getBlob(1)，强制转换多余）
    InputStream in = blob.getBinaryStream();             // Blob 对象转为 InputStream 流
    File file = new File(path);                          // 创建要写入的文件
    FileOutputStream out = new FileOutputStream(file);   // 定义文件的写入流
    int readbyte = 0;
    while ((readbyte = in.read()) != -1) {               // 一点一点读取流并写到文件中
        out.write(readbyte);
    }
    out.close();                                         // PS：别忘记关闭文件的写入流
    in.close();                                          // 更正：原文只关了输出流，输入流也要关
}
```

### 15. 元数据

手写版第 7 页"1. 获取数据库元数据"：

```java
Connection con = DriverManager.getConnection("jdbc:db2:sample");
DatabaseMetaData meta = con.getMetaData();
String[] tables = {"TABLE", "VIEW"};
ResultSet rs = meta.getTables(null, "JLU", "%", tables);   // 编目，模式，匹配表名，表的种类
while (rs.next()) {
    String catalog = rs.getString("TABLE_CAT");      // 编目
    String schema = rs.getString("TABLE_SCHEM");     // 模式
    String name = rs.getString("TABLE_NAME");        // 名称
    // 输出……
}
```

`getTables` 有 s（蓝字"别忘了s"），第一个参数写 `null`。

"2. 获取结果集元数据"：

```java
String sql = "SELECT * FROM Employee";
Statement stmt = con.createStatement();
ResultSet rs = stmt.executeQuery(sql);
ResultSetMetaData meta = rs.getMetaData();              // 获取结果集元数据
int columnCount = meta.getColumnCount();                // 获取结果集列数
for (int i = 1; i <= columnCount; i++) {                // 遍历获取每列名称、类型（更正：原文漏了 {）
    String columnName = meta.getColumnName(i);
    int columnType = meta.getColumnType(i);             // 更正：原文作 String columnType = meta.getColumnType(i)，该方法返回 int
    String columnTypeName = meta.getColumnTypeName(i);  // 要类型名字符串用这个
    // 输出……
}
```

### 16. 滚动结果集

手写版第 9 页：

```java
ResultSet rs = null;
String sql = "SELECT name, age FROM Employee WHERE deptNo = ?";   // 注意：员工表没有 deptNo 列，只是示意，换成表里有的列
// 更正：原文作 rs.TYPE_SCROLL_INSENSITIVE, rs.CONCUR_READ_ONLY，通过变量引用静态常量，应写类名
PreparedStatement pstmt = con.prepareStatement(sql, ResultSet.TYPE_SCROLL_INSENSITIVE, ResultSet.CONCUR_READ_ONLY);
String deptNo = "00";
pstmt.setString(1, deptNo);
rs = pstmt.executeQuery();   // 更正：原文漏了这句，rs 还是 null，下一句 rs.last() 会空指针

// 最后一个
rs.last();
String name = rs.getString(1);
int age = rs.getInt(2);
// 输出……

// 第一个
rs.first();
// ……

// 正序
rs.beforeFirst();
while (rs.next()) {
    // ……
}

// 倒序
rs.afterLast();
while (rs.previous()) {
    // ……
}
```

这段用的是只读并发 `CONCUR_READ_ONLY`。题目要求"结果集可更新"时（`01` 第 9 题）要换成 `ResultSet.CONCUR_UPDATABLE`。

### 17. 空值处理

手写版第 10 页：

```java
// 更正：原文作 "SELECT age FROM Employ FOR UPDATE"，只选了 age 一列，下面却用第 1 列取 empno、第 2 列取 age
String sql = "SELECT empno, age FROM Employee FOR UPDATE";
Statement stmt = con.createStatement();
ResultSet rs = stmt.executeQuery(sql);
String sql2 = "UPDATE Employee SET age = ? WHERE CURRENT OF ";   // OF 后面留空格
String cursorName = rs.getCursorName();
PreparedStatement pstmt = con.prepareStatement(sql2 + cursorName);
while (rs.next()) {
    String empno = rs.getString(1);
    int age = rs.getInt(2);
    if (rs.wasNull()) {                    // int、double、smallint 等都用 wasNull() 判断
        pstmt.setNull(1, Types.INTEGER);   // （位置，Types.类型大写）
        pstmt.executeUpdate();
    }
}
```

`wasNull()` 要紧跟在对应的 `getXxx` 后面调用。

### 18. 异常与警告

手写版第 11 页：

```java
Statement stmt = con.createStatement();
String sql = "SELECT name FROM Employee WHERE empno = '00010'";
ResultSet rs = stmt.executeQuery(sql);
SQLWarning sqlWarn = stmt.getWarnings();   // Connection、Statement、ResultSet 都可以获取 SQLWarning
if (sqlWarn != null) {
    System.out.println(sqlWarn);
}

try {
    // ……
} catch (SQLException e) {
    String SQLState = e.getSQLState();
    int SQLCode = e.getErrorCode();        // int
    String Message = e.getMessage();       // 没有 s!
    // 输出……
}
```

`getWarnings` 有 s（蓝字"别忘了s"），`getMessage` 没有 s，`getErrorCode` 返回 `int`。先执行语句再取警告。SQLSTATE 代码见第五节。

## 二、常用 JDBC 接口与方法

以下接口都在 `java.sql` 包里。

### DriverManager（以及驱动加载）

| 方法 | 用途 |
| --- | --- |
| `Class.forName(String className)` | 加载驱动类，放在静态块里。这是 `Class` 的方法，不属于 `DriverManager` |
| `static Connection getConnection(String url)` | 用默认身份连接 |
| `static Connection getConnection(String url, String user, String password)` | 用用户名和密码连接 |

### Connection

| 方法 | 用途 |
| --- | --- |
| `Statement createStatement()` | 创建普通语句，结果集默认只向前、只读 |
| `Statement createStatement(int resultSetType, int resultSetConcurrency)` | 指定结果集的滚动类型和并发类型 |
| `PreparedStatement prepareStatement(String sql)` | 创建预处理语句，SQL 里可以有 `?` |
| `PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency)` | 预处理语句并指定结果集类型，SQL 在第一个参数 |
| `PreparedStatement prepareStatement(String sql, int resultSetType, int resultSetConcurrency, int resultSetHoldability)` | 再指定提交后游标是否保持（JDBC 3.0 起） |
| `void setAutoCommit(boolean autoCommit)` | 传 `false` 关闭自动提交 |
| `void commit()` | 提交事务 |
| `void rollback()` | 回滚事务 |
| `DatabaseMetaData getMetaData()` | 取数据库元数据 |
| `SQLWarning getWarnings()` | 取这个连接上的警告 |
| `void close()` | 关闭连接 |

### Statement

| 方法 | 用途 |
| --- | --- |
| `ResultSet executeQuery(String sql)` | 执行 SELECT，返回结果集 |
| `int executeUpdate(String sql)` | 执行 INSERT、UPDATE、DELETE，返回影响的行数 |
| `void addBatch(String sql)` | 把一条 SQL 加入批 |
| `int[] executeBatch()` | 执行整批，返回每条语句影响的行数 |
| `SQLWarning getWarnings()` | 取执行语句产生的警告 |
| `void close()` | 关闭语句 |

### PreparedStatement（继承 Statement）

| 方法 | 用途 |
| --- | --- |
| `void setString(int parameterIndex, String x)` | 给第 `parameterIndex` 个 `?` 赋字符串，序号从 1 开始 |
| `void setInt(int parameterIndex, int x)` | 赋整数 |
| `void setDouble(int parameterIndex, double x)` | 赋小数 |
| `void setNull(int parameterIndex, int sqlType)` | 赋 SQL NULL，`sqlType` 写 `Types.INTEGER`、`Types.DOUBLE` 等 |
| `void setBinaryStream(int parameterIndex, InputStream x, int length)` | 用字节流写 BLOB，第三个参数是长度 |
| `void setCharacterStream(int parameterIndex, Reader reader, int length)` | 用字符流写 CLOB |
| `void setBlob(int parameterIndex, InputStream inputStream)` | 写 BLOB，JDBC 4.0 起才有 |
| `void setClob(int parameterIndex, Reader reader)` | 写 CLOB，JDBC 4.0 起才有 |
| `ResultSet executeQuery()` | 执行查询，不带参数，SQL 在创建时已经给了 |
| `int executeUpdate()` | 执行更新，不带参数，返回影响的行数 |
| `void addBatch()` | 把当前这组参数加入批，不带参数 |
| `int[] executeBatch()` | 执行整批（继承自 `Statement`） |

`java.sql.Types` 里常用的常量：`INTEGER`、`SMALLINT`、`DOUBLE`、`DECIMAL`、`CHAR`、`VARCHAR`、`CLOB`、`BLOB`，全大写。

### ResultSet

| 方法 | 用途 |
| --- | --- |
| `boolean next()` | 移到下一行，没有下一行返回 `false` |
| `String getString(int columnIndex)` | 按列号取字符串，列号从 1 开始 |
| `String getString(String columnLabel)` | 按列名取，元数据结果集常用，比如 `getString("TABLE_NAME")` |
| `int getInt(int columnIndex)` | 取整数，NULL 时返回 0 |
| `double getDouble(int columnIndex)` | 取小数，NULL 时返回 0.0 |
| `Clob getClob(int columnIndex)` | 取 CLOB 对象 |
| `Blob getBlob(int columnIndex)` | 取 BLOB 对象 |
| `boolean wasNull()` | 上一次 `getXxx` 读到的值是不是 SQL NULL |
| `String getCursorName()` | 取游标名，拼到 `WHERE CURRENT OF ` 后面 |
| `ResultSetMetaData getMetaData()` | 取结果集元数据 |
| `SQLWarning getWarnings()` | 取结果集上的警告 |
| `void close()` | 关闭结果集 |

滚动定位的方法见第四节。

### ResultSetMetaData

| 方法 | 用途 |
| --- | --- |
| `int getColumnCount()` | 列数 |
| `String getColumnName(int column)` | 第 `column` 列的列名，从 1 开始 |
| `int getColumnType(int column)` | 列类型，返回 `java.sql.Types` 的常量值（`int`） |
| `String getColumnTypeName(int column)` | 数据库里的类型名，比如 `CHAR`、`INTEGER` |
| `String getColumnClassName(int column)` | 对应 Java 类的全限定名，比如 `java.lang.String` |

### DatabaseMetaData

| 方法 | 用途 |
| --- | --- |
| `String getDriverName()` | 驱动名称 |
| `String getDriverVersion()` | 驱动版本 |
| `ResultSet getTableTypes()` | 表类型列表，列名 `TABLE_TYPE` |
| `ResultSet getSchemas()` | 模式列表，列名 `TABLE_SCHEM` |
| `ResultSet getTables(String catalog, String schemaPattern, String tableNamePattern, String[] types)` | 表和视图，结果列依次为 `TABLE_CAT`、`TABLE_SCHEM`、`TABLE_NAME`、`TABLE_TYPE` 等 |
| `ResultSet getColumns(String catalog, String schemaPattern, String tableNamePattern, String columnNamePattern)` | 列信息，常用结果列 `TABLE_NAME`、`COLUMN_NAME`、`TYPE_NAME` |

### Clob

| 方法 | 用途 |
| --- | --- |
| `long length()` | 字符数 |
| `String getSubString(long pos, int length)` | 从第 `pos` 个字符起取 `length` 个，`pos` 从 1 开始 |
| `Reader getCharacterStream()` | 以字符流读取 |

### Blob

| 方法 | 用途 |
| --- | --- |
| `long length()` | 字节数 |
| `InputStream getBinaryStream()` | 以字节流读取，配合 `read()` 写到文件 |
| `byte[] getBytes(long pos, int length)` | 取一段字节，`pos` 从 1 开始 |

### SQLException 和 SQLWarning

| 方法 | 用途 |
| --- | --- |
| `String getSQLState()` | 5 位 SQLSTATE 字符串 |
| `int getErrorCode()` | 厂商错误码，DB2 里就是 SQLCODE |
| `String getMessage()` | 错误信息，继承自 `Throwable`，没有 s |
| `SQLException getNextException()` | 取链上的下一个异常 |
| `SQLWarning getNextWarning()` | `SQLWarning` 的方法，取下一条警告 |

`SQLWarning` 是 `SQLException` 的子类，上面前三个方法它也能用。警告不会被抛出，要自己用 `getWarnings()` 取。

## 三、结果集类型与并发常量

| 常量 | 类别 | 含义 | 用在 |
| --- | --- | --- | --- |
| `ResultSet.TYPE_FORWARD_ONLY` | 滚动类型 | 只能用 `next()` 向前移动，默认值 | `01` 第 10 题 |
| `ResultSet.TYPE_SCROLL_INSENSITIVE` | 滚动类型 | 可以前后滚动，结果集打开后数据库里的修改看不到 | `01` 第 9、11 题 |
| `ResultSet.TYPE_SCROLL_SENSITIVE` | 滚动类型 | 可以前后滚动，能反映打开后数据库里的修改 | 原稿没用到 |
| `ResultSet.CONCUR_READ_ONLY` | 并发类型 | 只读，不能通过结果集修改数据，默认值 | `01` 第 10、11 题 |
| `ResultSet.CONCUR_UPDATABLE` | 并发类型 | 可以通过结果集修改数据 | `01` 第 9 题 |
| `ResultSet.HOLD_CURSORS_OVER_COMMIT` | 保持性（整理补充） | 提交后游标不关闭，对应 SQL 的 `WITH HOLD` | `01` 第 11 题易错点 |
| `ResultSet.CLOSE_CURSORS_AT_COMMIT` | 保持性（整理补充） | 提交时关闭游标 | |

- 写法：`con.createStatement(滚动类型, 并发类型)`，`con.prepareStatement(sql, 滚动类型, 并发类型)`。
- 常量用类名 `ResultSet.` 引用。原稿写的 `rs.TYPE_SCROLL_INSENSITIVE` 能编译，但不规范。
- DB2 的 INSENSITIVE 游标只能读，"不敏感 + 可更新"的组合驱动一般会降为只读并给出警告。要通过结果集改数据，用 `TYPE_SCROLL_SENSITIVE` 配 `CONCUR_UPDATABLE`。

## 四、滚动定位方法

| 方法 | 返回值 | 作用 | 说明 |
| --- | --- | --- | --- |
| `next()` | `boolean` | 移到下一行 | 没有下一行返回 `false`。只向前的结果集只能用它 |
| `previous()` | `boolean` | 移到上一行 | 配合 `afterLast()` 从后往前遍历 |
| `first()` | `boolean` | 移到第一行 | 结果集为空时返回 `false` |
| `last()` | `boolean` | 移到最后一行 | 结果集为空时返回 `false` |
| `beforeFirst()` | `void` | 移到第一行之前 | 之后用 `next()` 从头遍历 |
| `afterLast()` | `void` | 移到最后一行之后 | 之后用 `previous()` 从尾遍历 |
| `absolute(int row)` | `boolean` | 移到第 `row` 行 | 正数从开头数，1 是第一行；负数从末尾数，-1 是最后一行；超出范围返回 `false` |
| `relative(int rows)` | `boolean` | 从当前行移动 `rows` 行 | 正数往后、负数往前。停在第 6 行时 `relative(-3)` 到第 3 行 |

- 除 `next()` 以外都要求结果集可滚动，在 `TYPE_FORWARD_ONLY` 的结果集上调用会抛 `SQLException`。
- 游标停在第一行之前或最后一行之后时不能 `getXxx`，先移到有效行。

## 五、DB2 SQLSTATE 代码

| 情况 | SQLSTATE | 常见 SQLCODE | 手写版 | 答题模板 | 核对结果 |
| --- | --- | --- | --- | --- | --- |
| 未找到行（查询没有结果、游标读到末尾） | `02000` | `+100` | 不存在 02000 | 没写 | 一致 |
| 字符串过长 | `22001` | `-302`、`-433` | 字符串过长 22001 | 22001，未写 SQLCODE | 一致 |
| 数值超出范围（溢出） | `22003` | `-802` | 数值溢出 22003 | 22003，-802 | 一致 |
| 违反非空约束（往 NOT NULL 列放 NULL） | `23502` | `-407` | 违反非空约束 23502 | 23502，-407 | 一致 |
| 外键值无效（插入或修改子表时，父表里没有对应的行） | `23503` | `-530` | 没写 | 没写 | 整理补充 |
| 父键因 NO ACTION 规则不能更新或删除（子表还有行引用它） | `23504` | `-531`（更新父键）、`-532`（删除父行） | 违反外键约束 23504 | 违反外键约束 23504，-532 | 两份都笼统写成"违反外键约束"，23504 只对应删除或修改父表行这种情况 |
| 违反唯一约束或主键重复 | `23505` | `-803` | 违反主键约束 23505 | 违反唯一性约束 23505，-803 | 一致，主键重复和唯一约束冲突都是它 |
| 运算符或函数的操作数类型不兼容 | `42818` | `-401` | 类型不兼容 42818 | 42818 | 一致 |

- SQLCODE 一列里，原稿只给了 `-802`、`-803`、`-407`、`-532`，其余是整理补充。
- SQLSTATE 前两位是类别：`00` 成功，`01` 警告，`02` 没有数据，`22` 数据异常，`23` 违反约束，`42` 语法或访问规则错误。
- SQLCODE：0 成功，正数是警告（`+100` 没有数据），负数是错误。
- 23 开头的四个连着背：502 非空，503 外键值无效，504 父键删改受限，505 唯一或主键重复。

## 附：直接改掉的笔误

下面这些是拼写或格式问题，片段里直接改了，没有逐处标注：

- `prinkStackTrace` 改为 `printStackTrace`（第 2 页）。
- `Comeection` 改为 `Connection`（第 4 页）。
- `Result rs` 改为 `ResultSet rs`（第 4、5 页）。
- `Prepared pstmt` 改为 `PreparedStatement pstmt`（第 6 页）。
- `PrepareStatement` 改为 `PreparedStatement`（第 8 页）。
- 提示串 `passward` 改为 `password`（第 1 页）。
- 表名 `Employ` 统一为 `Employee`。
- 第 6 页插入图片的 SQL 那行漏了分号，第 11 页 `try` 块和 `catch` 之间漏了右括号，都已补上。
