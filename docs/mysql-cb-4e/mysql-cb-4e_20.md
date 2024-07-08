# 第二十章：执行事务

# 20.0 简介

MySQL 服务器可以同时处理多个客户端，因为它是多线程的。为了处理客户端之间的竞争，服务器执行任何必要的锁定，以防止两个客户端同时修改同一数据。然而，当服务器执行 SQL 语句时，来自同一客户端的连续语句可能与来自其他客户端的语句交错执行。如果一个客户端执行多个依赖于彼此的语句，其他客户端在这些语句之间更新表可能会引发困难。

如果多语句操作未能完成，语句失败也可能会带来问题。假设`flight`表包含有关航班时间表的信息，并且您想通过从可用飞行员中选择一个来更新 Flight 578 的行。您可以使用以下三个语句来执行：

```
SET @p_val = (SELECT pilot_id FROM pilot WHERE available = 'yes' LIMIT 1);
UPDATE pilot SET available = 'no' WHERE pilot_id = @p_val;
UPDATE flight SET pilot_id = @p_val WHERE flight_id = 578;
```

第一条语句选择一个可用的飞行员，第二条将该飞行员标记为不可用，第三条将该飞行员分配给航班。原则上说这很简单，但实际操作中存在显著困难：

并发问题

如果两个客户端想要安排飞行员，它们可以在设置飞行员状态为不可用之前同时运行初始的`SELECT`查询并检索相同的飞行员 ID 号。如果发生这种情况，同一飞行员会同时安排两个航班。

完整性问题

所有三个语句必须作为一个单元成功执行。例如，如果`SELECT`和第一个`UPDATE`成功运行，但第二个`UPDATE`失败，则飞行员的状态被设置为不可用，但飞行员未被分配到航班。数据库变得不一致。

为了在这些类型的情况下预防并发和完整性问题，事务是有帮助的。事务组合了一组语句并保证以下属性：

+   当事务正在进行时，其他客户端不能更新事务中使用的数据；就像你独占了服务器一样。例如，在您为一个航班预订飞行员时，其他客户端不能修改飞行员或航班记录。事务解决了 MySQL 服务器多客户端性质带来的并发问题。事实上，事务序列化了多语句操作对共享资源的访问。

+   事务中分组的语句将作为一个单元提交（生效），但仅当它们全部成功时。如果发生错误，则会回滚发生错误之前的所有操作，使相关表保持不受影响，就好像没有执行任何语句一样。这可以防止数据库变得不一致。例如，如果更新`flights`表失败，则回滚会取消对`pilots`表的更改，使飞行员仍然可用。回滚使您无需自行解决部分完成的操作的撤销问题。

本章展示了开始和结束事务的 SQL 语句的语法。还描述了如何从程序内部实现事务操作，使用错误检测来确定是提交还是回滚。

此处显示的示例相关脚本位于`recipes`发行版的*transactions*目录中。

# 20.1 选择事务存储引擎

## 问题

您想要使用事务。

## 解决方案

要使用事务，必须使用事务安全引擎。检查您的 MySQL 服务器以确定它支持哪些事务存储引擎。

## 讨论

MySQL 支持多种存储引擎。当前的事务引擎，包括 InnoDB 和 NDB，已随标准分发。要查看您的 MySQL 服务器支持哪些，请使用此语句：

```
mysql> `SELECT ENGINE FROM INFORMATION_SCHEMA.ENGINES`
    -> `WHERE SUPPORT IN ('YES','DEFAULT') AND TRANSACTIONS='YES';`
+--------+
| ENGINE |
+--------+
| InnoDB |
+--------+
```

如果启用了 MySQL Cluster，则还会看到一行显示`ndbcluster`。

事务引擎是那些具有`TRANSACTIONS`值为`YES`的引擎；实际可用的引擎具有`SUPPORT`值为`YES`或`DEFAULT`。

在确定了哪些事务存储引擎可用后，要创建使用特定引擎的表，请在您的`CREATE TABLE`语句中添加一个`ENGINE = ` *`tbl_engine`* 子句：

```
CREATE TABLE *`t`* (i INT) ENGINE = InnoDB;
```

如果需要修改现有应用程序以执行事务，但它使用非事务表，则可以修改表以使用事务存储引擎。例如，MyISAM 表是非事务性的，试图将它们用于事务将产生错误的结果，因为它们不支持回滚。在这种情况下，您可以使用`ALTER TABLE`将表转换为事务类型。假设`t`是一个 MyISAM 表。要将其转换为 InnoDB 表，请执行以下操作：

```
ALTER TABLE *`t`* ENGINE = InnoDB;
```

在修改表之前要考虑的一件事是，将其更改为使用事务存储引擎可能会影响其在其他方面的行为。例如，MyISAM 引擎对`AUTO_INCREMENT`列提供了比其他存储引擎更灵活的处理方式。如果依赖于仅支持 MyISAM 的序列特性，更改存储引擎将会导致问题。

# 20.2 使用 SQL 执行事务

## 问题

一组语句必须作为一个单元成功或失败，也就是说，您需要一个事务。

## 解决方案

修改 MySQL 的自动提交模式以启用多语句事务，然后根据成功或失败来提交或回滚语句。

## 讨论

本方法描述了在 MySQL 中控制事务行为的 SQL 语句。紧接着的方法讨论了如何从程序内部执行事务。一些 API 要求您通过执行本方法中讨论的 SQL 语句来实现事务；其他 API 则提供了一种特殊机制，允许在不直接编写 SQL 的情况下进行事务管理。然而，即使在后一种情况下，API 机制也会将程序操作映射到事务性 SQL 语句上，因此阅读本方法将让您更好地了解 API 在您的代表下所做的事情。

MySQL 通常在自动提交模式下运行，这会在每次执行语句时提交其效果。（实际上，每个语句都是其自己的事务。）要执行一个事务，必须禁用自动提交模式，执行构成事务的语句，然后要么提交，要么回滚更改。在 MySQL 中，可以通过两种方式实现这一点：

+   执行 `START TRANSACTION`（或 `BEGIN`）语句以暂停自动提交模式，然后执行构成事务的语句。如果语句成功，则记录其在数据库中的效果，并通过执行 `COMMIT` 语句终止事务：

    ```
    mysql> `CREATE TABLE t (i INT) ENGINE = InnoDB;`
    mysql> `START TRANSACTION;`
    mysql> `INSERT INTO t (i) VALUES(1);`
    mysql> `INSERT INTO t (i) VALUES(2);`
    mysql> `COMMIT;`
    mysql> `SELECT * FROM t;`
    +------+
    | i    |
    +------+
    |    1 |
    |    2 |
    +------+
    ```

    如果发生错误，请不要使用 `COMMIT`。而是通过执行 `ROLLBACK` 语句来取消事务。在下面的例子中，由于 `INSERT` 语句的效果被回滚，`t` 保持为空：

    ```
    mysql> `CREATE TABLE t (i INT) ENGINE = InnoDB;`
    mysql> `START TRANSACTION;`
    mysql> `INSERT INTO t (i) VALUES(1);`
    mysql> `INSERT INTO t (x) VALUES(2);`
    ERROR 1054 (42S22): Unknown column 'x' in 'field list'
    mysql> `ROLLBACK;`
    mysql> `SELECT * FROM t;`
    Empty set (0.00 sec)
    ```

+   另一种分组语句的方式是通过将 `autocommit` 会话变量显式设置为 0 来关闭自动提交模式。之后，您执行的每个语句都成为当前事务的一部分。要结束事务并开始下一个事务，请执行 `COMMIT` 或 `ROLLBACK` 语句：

    ```
    mysql> `CREATE TABLE t (i INT) ENGINE = InnoDB;`
    mysql> `SET autocommit = 0;`
    mysql> `INSERT INTO t (i) VALUES(1);`
    mysql> `INSERT INTO t (i) VALUES(2);`
    mysql> `COMMIT;`
    mysql> `SELECT * FROM t;`
    +------+
    | i    |
    +------+
    |    1 |
    |    2 |
    +------+
    ```

    要重新启用自动提交模式，请使用以下语句：

    ```
    mysql> `SET autocommit = 1;`
    ```

###### 警告

事务有其局限性，因为并非所有语句都可以成为事务的一部分。例如，如果执行 `DROP DATABASE` 语句，则不要期望通过执行 `ROLLBACK` 来恢复数据库。

# 20.3 从程序内部执行事务

## 问题

您正在编写一个必须实现事务操作的程序。

## 解决方案

如果您的语言 API 提供了事务抽象化，可以使用该功能。如果没有，则使用 API 的常规语句执行机制直接执行事务性 SQL 语句。

## 讨论

要从程序内部执行事务处理，使用您的 API 语言检测错误并采取适当的操作。这个方法提供了关于如何执行这一操作的一般背景。接下来的方法将为 Perl、Ruby、PHP、Python、Go 和 Java 的 MySQL API 提供语言特定的详细信息。

每个 MySQL API 都支持事务，即使仅仅是显式地执行与事务相关的 SQL 语句如`START TRANSACTION`和`COMMIT`。然而，一些 API 还提供了事务抽象，使得可以控制事务行为，而无需直接使用 SQL。这种方法隐藏了细节，并且提供了更好的可移植性，以适应具有不同底层事务 SQL 语法的其他数据库引擎。本书中使用的每种语言都有相应的 API 抽象。

接下来的几个示例展示了如何执行基于程序的事务的相同示例，它们使用一个包含以下初始行的`money`表，展示了两个人拥有的金额：

```
+------+------+
| name | amt  |
+------+------+
| Eve  |   10 |
| Ida  |    0 |
+------+------+
```

样本交易是一笔简单的金融转账，使用两个`UPDATE`语句将伊娃的六美元转给艾达：

```
UPDATE money SET amt = amt - 6 WHERE name = 'Eve';
UPDATE money SET amt = amt + 6 WHERE name = 'Ida';
```

预期的结果是表应该如下所示：

```
+------+------+
| name | amt  |
+------+------+
| Eve  |    4 |
| Ida  |    6 |
+------+------+
```

必须在事务中执行这两个语句以确保它们同时生效。如果没有事务，如果第二个语句失败，则伊娃的钱将不会被记入艾达的账户。通过使用事务，如果语句失败，表将保持不变。

每种语言的示例程序位于`recipes`发布的*transactions*目录中。如果你进行比较，你会发现它们都使用类似的框架执行事务处理：

+   事务语句被包含在一个控制结构内，以及一个提交操作。

+   如果控制结构的状态表明它未成功执行完成，则回滚事务。

可以将该逻辑表达如下，其中`block`表示用于分组语句的控制结构：

```
block:
  statement 1
  statement 2
  ...
  statement *`n`*
  commit
if the block failed:
  roll back
```

如果块中的语句成功执行，你将到达块的末尾并执行提交。否则，出现错误会引发异常，触发错误处理代码的执行，其中你会回滚事务。

将代码结构化为刚才描述的方式的好处在于，它最大限度地减少了确定是否回滚所需的测试数量。另一种方法——在事务中检查每个语句的结果，并在个别语句错误时回滚——很快会使你的代码变得难以阅读。

在引发异常的语言中回滚时需要注意的一个微妙点是，回滚本身可能会失败，导致另一个异常被引发。如果你不处理这个问题，你的程序本身可能会终止。为了处理这个问题，在另一个没有异常处理程序的块内执行回滚是必要的。示例程序在必要时会这样做。

那些在执行事务时显式禁用自动提交模式的示例程序在之后执行事务后启用自动提交。在将所有数据库处理以事务方式执行的应用程序中，不必这样做。只需在连接到数据库服务器后一次禁用自动提交模式，并保持禁用状态。

# 20.4 Perl 程序中执行事务

## 问题

您希望在 Perl DBI 脚本中执行事务。

## 解决方案

使用标准的 DBI 事务支持机制。

## 讨论

Perl DBI 事务机制基于显式操作自动提交模式：

1.  如果 `RaiseError` 属性未启用，则打开它；如果启用了 `PrintError`，则禁用它。您希望错误引发异常而不打印任何内容，保持 `PrintError` 启用可能会干扰某些情况下的故障检测。

1.  禁用 `AutoCommit` 属性，以便仅在您说时提交。

1.  在一个 `eval` 块中执行组成事务的语句，以便于错误会引发异常并终止该块。该块的最后一件事应该是调用 `commit()`，如果所有语句成功完成，则提交事务。

1.  在执行 `eval` 后，检查 `$@` 变量。如果 `$@` 包含空字符串，则事务成功。否则，由于出现错误，`eval` 将失败，`$@` 将包含错误消息。调用 `rollback()` 取消事务。要显示错误消息，请在调用 `rollback()` 之前打印 `$@`。

1.  如果需要，恢复 `RaiseError` 和 `PrintError` 属性的原始值。

如果应用程序执行多个事务，更改和恢复错误处理和自动提交属性可能会很混乱，让我们将开始和结束事务的代码放入方便处理前后处理的函数中，这些函数处理 `eval` 前后的处理：

```
sub transaction_init
{
my $dbh = shift;
my $attr_ref = {};  # create hash in which to save attributes

  $attr_ref->{RaiseError} = $dbh->{RaiseError};
  $attr_ref->{PrintError} = $dbh->{PrintError};
  $attr_ref->{AutoCommit} = $dbh->{AutoCommit};
  $dbh->{RaiseError} = 1; # raise exception if an error occurs
  $dbh->{PrintError} = 0; # don't print an error message
  $dbh->{AutoCommit} = 0; # disable auto-commit
  return $attr_ref;       # return attributes to caller
}

sub transaction_finish
{
my ($dbh, $attr_ref, $error) = @_;

  if ($error) # an error occurred
  {
    print "Transaction failed, rolling back. Error was:\n$error\n";
    # roll back within eval to prevent rollback
    # failure from terminating the script
    eval { $dbh->rollback (); };
  }
  # restore error-handling and auto-commit attributes
  $dbh->{AutoCommit} = $attr_ref->{AutoCommit};
  $dbh->{PrintError} = $attr_ref->{PrintError};
  $dbh->{RaiseError} = $attr_ref->{RaiseError};
}
```

通过使用这两个函数，我们的示例事务可以轻松执行如下：

```
$ref = transaction_init ($dbh);
eval
{
  # move some money from one person to the other
  $dbh->do ("UPDATE money SET amt = amt - 6 WHERE name = 'Eve'");
  $dbh->do ("UPDATE money SET amt = amt + 6 WHERE name = 'Ida'");
  # all statements succeeded; commit transaction
  $dbh->commit ();
};
transaction_finish ($dbh, $ref, $@);
```

在 Perl DBI 中，手动操作 `AutoCommit` 属性的替代方法是通过调用 `begin_work()` 开始事务。此方法禁用 `AutoCommit`，并在稍后调用 `commit()` 或 `rollback()` 时自动启用它。

# 20.5 Ruby 程序中执行事务

## 问题

您希望在 Ruby Mysql2 脚本中执行事务。

## 解决方案

发送事务管理语句，例如 `START TRANSACTIONS`、`BEGIN`、`COMMIT` 和 `ROLLBACK` 作为常规查询。

## 讨论

Ruby Mysql2 模块没有内置的事务支持功能。相反，它期望其用户将事务管理语句作为常规查询运行。

要开始事务，请执行 `client.query("START TRANSACTION")`，然后执行所需的更新，并用 `client.query("COMMIT")` 结束块。

将您的事务放入 `begin ... rescue` 块中，以便在出现问题时调用 `ROLLBACK`。

```
begin
  client.query("START TRANSACTION")
  client.query("UPDATE money SET amt = amt - 6 WHERE name = 'Eve'")
  client.query("UPDATE money SET amt = amt + 6 WHERE name = 'Ida'")
  client.query("COMMIT")
rescue Mysql2::Error => e
  puts "Transaction failed, rolling back. Error was:"
  puts "#{e.errno}: #{e.message}"
  begin           # empty exception handler in case rollback fails
    client.query("ROLLBACK")
  rescue
  end
end
```

# 20.6 在 PHP 程序中执行事务

## 问题

您希望在 PHP 脚本中执行事务。

## 解决方案

使用标准的 PDO 事务支持机制。

## 讨论

PDO 扩展支持事务抽象，可用于执行事务。要开始事务，请使用`beginTransaction()`方法。然后，在执行语句后，调用`commit()`或`rollback()`来提交或回滚事务。以下代码说明了这一点。它使用异常来检测事务失败，因此假定 PDO 错误已启用异常：

```
try
{
  $dbh->beginTransaction ();
  $dbh->exec ("UPDATE money SET amt = amt - 6 WHERE name = 'Eve'");
  $dbh->exec ("UPDATE money SET amt = amt + 6 WHERE name = 'Ida'");
  $dbh->commit ();
}
catch (Exception $e)
{
  print ("Transaction failed, rolling back. Error was:\n");
  print ($e->getMessage () . "\n");
  # empty exception handler in case rollback fails
  try
  {
    $dbh->rollback ();
  }
  catch (Exception $e2) { }
}
```

# 20.7 在 Python 程序中执行事务

## 问题

您希望在 Python DB API 脚本中执行事务。

## 解决方案

使用标准的 DB API 事务支持机制。

## 讨论

Python DB API 抽象通过连接对象方法提供事务处理控制。DB API 规范指出，数据库连接应该以禁用自动提交模式开始。因此，当您打开与数据库服务器的连接时，Connector/Python 会禁用自动提交模式，这隐式地开始了一个事务。每个事务要么在`commit()`语句中结束，要么在`except`子句中的`rollback()`中取消事务，以取消事务如果发生错误：

```
try:
  cursor = conn.cursor()
  # move some money from one person to the other
  cursor.execute("UPDATE money SET amt = amt - 6 WHERE name = 'Eve'")
  cursor.execute("UPDATE money SET amt = amt + 6 WHERE name = 'Ida'")
  cursor.close()
  conn.commit()
except mysql.connector.Error as e:
  print("Transaction failed, rolling back. Error was:")
  print(e)
  try:  # empty exception handler in case rollback fails
    conn.rollback()
  except:
    pass
```

# 20.8 在 Go 程序中执行事务

## 问题

您希望在 Go 程序中执行事务

## 解决方案

使用由`database/sql`包提供的标准事务支持机制。

## 讨论

Go `sql`接口支持事务抽象，可用于执行事务。要开始事务，请使用`DB.Begin()`函数。然后，在执行语句后，调用`Tx.Commit()`或`Tx.Rollback()`来提交或回滚事务。以下代码说明了这一点。

```
var queries = []string{
  "UPDATE money SET amt = amt - 6 WHERE name = 'Eve'",
  "UPDATE money SET amt = amt + 6 WHERE name = 'Ida'",
}

tx, err := db.Begin()
if err != nil {
  log.Fatal(err)
}

for _, query := range queries {
  _, err := tx.Exec(query)
  if err != nil {
    fmt.Printf("Transaction failed, rolling back.\nError was: %s\n",
               err.Error())
    if txerr := tx.Rollback(); txerr != nil {
      fmt.Println("Rollback failed")
      log.Fatal(txerr)
    }
  }
}

if err := tx.Commit(); err != nil {
  log.Fatal(err)
}
```

# 20.9 使用上下文感知函数处理 Go 中的事务

## 问题

您希望在 Go 程序中自动回滚事务。

## 解决方案

Go-MySQL-Driver 支持上下文取消。这意味着您可以取消数据库操作，例如取消运行查询，如果取消上下文。

## 讨论

要使用[context](https://pkg.go.dev/context)包与 SQL，您需要首先创建`Context`类型的对象，然后将其传递给数据库函数。支持上下文的`sql`接口函数与不支持的函数名称相似，但具有前缀`Context`。例如，函数`Query()`不支持`Context`，而函数`QueryContext()`支持。

下面的示例使用`Context`来处理数据库事务。您可以在`recipes`分发的`transactions`目录中的文件*transaction_context.go*中找到其代码。

```
// transaction_context.go: simple transaction demonstration 
//                         with use of Context

// By default, this creates an InnoDB table.  If you specify a storage
// engine on the command line, that will be used instead.  Normally,
// this should be a transaction-safe engine that is supported by your
// server.  However, you can pass a nontransactional storage engine
// to verify that rollback doesn't work properly for such engines.

// The script uses a table named "money" and drops it if necessary.
// Change the name if you have a valuable table with that name. :-)
package main

import (
  "log"
  "fmt"
  "flag"
  "context" ![1](img/1.png)
  "database/sql"
  "github.com/svetasmirnova/mysqlcookbook/recipes/lib"
)

func initTable(ctx context.Context, db *sql.DB, tblEngine string) (error) { ![2](img/2.png)
  queries := [4]string {
    "DROP TABLE IF EXISTS money",
    "CREATE TABLE money (name CHAR(5), amt INT, PRIMARY KEY(name)) ENGINE = " + tblEngine,
    "INSERT INTO money (name, amt) VALUES('Eve', 10)",
    "INSERT INTO money (name, amt) VALUES('Ida', 0)",
  }

  for _, query := range queries {
    _, err = db.ExecContext(ctx, query) ![3](img/3.png)
    if err != nil {
      fmt.Println("Cannot initialize test table")
      fmt.Printf("Error: %s\n", err.Error())
      return err
    }
  }

  return nil
}

func displayTable(ctx context.Context, db *sql.DB) (error) {
  rows, err := db.QueryContext(ctx, "SELECT name, amt FROM money") ![4](img/4.png)
  if err != nil {
    return err
  }
  defer rows.Close()

  for rows.Next() {
    var (
      name string
      amt  int32
    )
    if err := rows.Scan(&name, &amt); err != nil {
      fmt.Println("Cannot display contents of test table")
      fmt.Printf("Error: %s\n", err.Error())
      return err
    }

    fmt.Printf("%s has $%d\n", name, amt)
  }

  return nil
}

func runTransaction(ctx context.Context, 
                    db *sql.DB, queries []string) (error) {
  tx, err := db.BeginTx(ctx, nil) ![5](img/5.png)
  if err != nil {
    return err
  }

  for _, query := range queries {
    _, err := tx.ExecContext(ctx, query) ![6](img/6.png)
    if err != nil {
      fmt.Printf("Transaction failed, rolling back.\nError was: %s\n",
                 err.Error())
      if txerr := tx.Rollback(); err != nil {
        return txerr
      }
      return err
    }
  }

  if err := tx.Commit(); err != nil {
    return err
  }

  return nil
}

func main() {
  db, err := cookbook.Connect()
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()

  var tblEngine string = "InnoDB"
  flag.Parse()
  values := flag.Args()
  if len(values) > 0 {
    tblEngine = values[0]
  }
  fmt.Printf("Using storage engine %s to test transactions\n", tblEngine)

  ctx, cancel := context.WithCancel(context.Background()) ![7](img/7.png)
  defer cancel()

  fmt.Println("----------")
  fmt.Println("This transaction should succeed.")
  fmt.Println("Table contents before transaction:")

  if err := initTable(ctx, db, tblEngine); err != nil {
    log.Fatal(err)
  }

  if err = displayTable(ctx, db); err != nil {
    log.Fatal(err)
  }

  var trx = []string{
    "UPDATE money SET amt = amt - 6 WHERE name = 'Eve'",
    "UPDATE money SET amt = amt + 6 WHERE name = 'Ida'",
  }

  if err = runTransaction(ctx, db, trx); err != nil {
    log.Fatal(err)
  }

  fmt.Println("Table contents after transaction:")
  if err = displayTable(ctx, db); err != nil {
    log.Fatal(err)
  }

  fmt.Println("----------")
  fmt.Println("This transaction should fail.")
  fmt.Println("Table contents before transaction:")

  if err := initTable(ctx, db, tblEngine); err != nil {
    log.Fatal(err)
  }

  if err = displayTable(ctx, db); err != nil {
    log.Fatal(err)
  }

  trx = []string{
    "UPDATE money SET amt = amt - 6 WHERE name = 'Eve'",
    "UPDATE money SET xamt = amt + 6 WHERE name = 'Ida'",
  }

  if err = runTransaction(ctx, db, trx); err != nil {
    log.Fatal(err)
  }

  fmt.Println("Table contents after transaction:")
  if err = displayTable(ctx, db); err != nil {
    log.Fatal(err)
  }
}
```

![1](img/#co_nch-xact-xact-go-context_import_co)

上下文支持的导入语句。

![2](img/#co_nch-xact-xact-go-context_funcparam_co)

我们的用户定义函数以 `context.Context` 作为参数。

![3](img/#co_nch-xact-xact-go-context_exec_co)

要执行不返回结果集的语句，请使用支持上下文的函数 `ExecContext()`。

![4](img/#co_nch-xact-xact-go-context_query_co)

要执行返回结果集的查询，请使用支持上下文的函数 `QueryContext()`。

![5](img/#co_nch-xact-xact-go-context_begin_co)

要启动一个如果上下文取消将自动回滚的事务，请使用支持上下文的函数 `BeginTx()`。

![6](img/#co_nch-xact-xact-go-context_tx_co)

要执行可能在事务中取消的语句，请使用支持上下文的函数 `Tx.ExecContext()`。

![7](img/#co_nch-xact-xact-go-context_ctx_co)

在使用上下文之前，您需要创建它。在我们的示例中，我们创建了一个可取消的上下文。函数 `context.WithCancel()` 接受父上下文作为参数，并返回刚创建的新上下文及一个 `cancel()` 函数。我们推迟了它在 `main()` 函数执行结束时的调用。您可以在代码的任何适当位置调用 `cancel()` 函数。您可以选择使用 `context.WithDeadline()` 或 `context.WithTimeout()`，以便在超过一定时间后取消 SQL 执行代码。

# 20.10 在 Java 程序中执行事务

## 问题

您希望在 JDBC 应用程序中执行事务。

## 解决方案

使用标准的 JDBC 事务支持机制。

## 讨论

要在 Java 中执行事务，请使用您的 `Connection` 对象关闭自动提交模式。然后，在执行语句后，使用该对象的 `commit()` 方法提交事务或 `rollback()` 取消事务。通常，您在事务的 `try` 块中执行语句，并在块末尾调用 `commit()`。若出现故障，请在相应的异常处理程序中调用 `rollback()`：

```
try
{
  conn.setAutoCommit (false);
  Statement s = conn.createStatement ();
  // move some money from one person to the other
  s.executeUpdate ("UPDATE money SET amt = amt - 6 WHERE name = 'Eve'");
  s.executeUpdate ("UPDATE money SET amt = amt + 6 WHERE name = 'Ida'");
  s.close ();
  conn.commit ();
  conn.setAutoCommit (true);
}
catch (SQLException e)
{
  System.err.println ("Transaction failed, rolling back. Error was:");
  Cookbook.printErrorMessage (e);
  // empty exception handler in case rollback fails
  try
  {
    conn.rollback ();
    conn.setAutoCommit (true);
  }
  catch (Exception e2) { }
}
```
