# 第二十四章：安全

# 24.0 简介

本章涵盖与安全相关的主题：

+   包含 MySQL 账户信息的`mysql.user`表

+   管理 MySQL 用户账户的语句

+   密码强度检查和策略

+   密码过期

+   查找和移除允许从多个主机连接的匿名账户和账户

如果你愿意，可以跳过描述`mysql.user`表的初始部分，但我认为阅读它将有助于你更好地理解后面的部分，这些部分通常讨论 SQL 操作如何映射到该表的基础变更。

本章展示的脚本位于`recipes`分发的*routines*目录中。

###### 注意

无论你使用 MySQL 5.7 还是 8.0 版本系列，最好使用系列内的最新版本。早期开发版本对身份验证系统的更改可能会导致与这里描述不同的结果。

###### 提示

这里展示的许多技术需要管理访问权限，例如修改`mysql`系统数据库中的表或使用需要管理员权限的语句来管理 MySQL 服务器。因此，为了执行这里描述的操作，请以`root`身份连接到服务器，而不是`cbuser`。

# 24.1 理解 mysql.user 表

MySQL 将用户账户信息存储在`mysql`系统数据库的表中。`user`表最为重要，因为它包含账户名和凭据。要查看其结构，请使用以下语句：

```
SHOW CREATE TABLE mysql.user;
```

这里关注的`user`表列指定了账户名称和身份验证信息：

+   `User`和`Host`列标识账户。MySQL 账户名由用户名和主机名值的组合构成。例如，在`'cbuser'@'localhost'`账户的`user`表行中，`User`和`Host`列的值分别为`cbuser`和`localhost`。对于`'myuser'@'myhost.example.com'`账户，这些列分别为`myuser`和`myhost.example.com`。

+   `plugin`和`authentication_string`列存储身份验证凭据。MySQL 不会在`user`系统表中存储明文密码，因为那样是不安全的。相反，服务器会从密码计算哈希值并存储哈希字符串。

    +   `plugin`列指示服务器使用的身份验证插件，用于检查试图使用账户的客户端的凭据。不同的插件实现了不同加密强度的密码哈希方法。表 24-1 显示了本章讨论的插件。

        表 24-1 身份验证插件

        | 插件 | 身份验证方法 |
        | --- | --- |
        | `mysql_native_password` | 原生密码哈希 |
        | `sha256_password` | 使用 SHA-256 密码哈希（从 MySQL 5.6.6 到 MySQL 8.0） |
        | `caching_sha2_password` | 使用 SHA-256 密码哈希和服务器端缓存（MySQL 5.7 或更高版本） |

        MySQL Enterprise，MySQL 的商业版本，包含额外的插件，用于使用 PAM 或 Windows 凭据进行身份验证。这些插件使得可以使用 MySQL 外部的密码，例如 Unix 登录密码或本机 Windows 服务。

    +   `authentication_string` 列以所需的格式表示哈希密码。例如，`sha256_password` 使用 `authentication_string` 存储 SHA-256 密码哈希值，这比 `mysql_native_password` 插件使用的本机哈希方法更为安全。空的 `authentication_string` 值表示“无密码”，这是不安全的。

在 MySQL 5.7.2 之前，服务器允许 `plugin` 值为空。从 MySQL 5.7.2 开始，`plugin` 列必须非空，并且服务器将禁用任何空插件帐户，直到分配了非空插件。

# 24.2 管理用户帐户

## 问题

您负责在 MySQL 服务器上设置帐户。

## 解决方案

学习使用帐户管理的 SQL 语句。

## 讨论

可以直接使用 SQL 语句（如 `INSERT` 或 `UPDATE`）修改 `mysql` 数据库中的授权表，但是使用 MySQL 的帐户管理语句更为方便。本节描述了它们的使用，并涵盖了以下主题：

+   创建帐户（`CREATE` `USER`，`SET` `PASSWORD`，`ALTER USER`）

+   分配和检查权限（`GRANT`，`REVOKE`，`SHOW` `GRANTS`）

+   移除和重命名帐户（`DROP` `USER`，`RENAME` `USER`）

### 创建帐户

要创建帐户，请使用 `CREATE` `USER` 语句，在 `mysql.user` 表中创建一行。但在执行此操作之前，请决定以下三件事：

+   帐户名以 `'`*`user_name`*`'@'`*`host_name`*`'` 格式表示，命名了用户和用户将连接的主机

+   帐户密码

+   客户端尝试使用帐户时服务器应执行的身份验证插件

身份验证插件使用哈希算法加密密码以进行存储和传输。MySQL 提供了几个内置的插件供选择：

+   `mysql_native_password` 在 8.0 版本之前实现了默认的密码哈希方法。

+   `sha256_password` 使用 SHA-256 密码哈希值进行身份验证，这比由 `mysql_native_password` 生成的哈希在密码学上更安全。该插件从 MySQL 5.6.6 开始可用，在 8.0 版本中被其改进版本 `caching_sha2_password` 取代。它提供比 `mysql_native_password` 更高的安全性，但需要额外的设置才能使用（客户端必须使用 SSL 连接或提供 RSA 证书）。

+   `caching_sha2_password` 类似于 `sha256_password`，但在服务器端使用缓存以获得更好的性能。这是 MySQL 8.0 以后的默认身份验证插件。

`CREATE` `USER` 语句通常以以下形式之一使用：

```
CREATE USER '*`user_name`*'@'*`host_name`*' IDENTIFIED BY '*`password`*';
CREATE USER '*`user_name`*'@'*`host_name`*' IDENTIFIED WITH '*`auth_plugin`*' BY '*`auth_string`*';
```

第一个语法用单个语句创建账户并设置其密码。它还隐式地将认证插件分配给由`--default-authentication-plugin`设置指定的插件（默认为`caching_sha2_password`，除非在服务器启动时更改）。

要为初始没有任何权限的新账户分配权限，请使用本节后面描述的`GRANT`语句。

如果账户已经存在，则`CREATE` `USER`会失败。

### 分配和检查权限

假设您刚刚创建了一个名为`'user1'@'localhost'`的账户。您可以使用`GRANT`为其分配权限，使用`REVOKE`撤销其权限，并使用`SHOW` `GRANTS`检查其权限。

`GRANT`的语法如下：

```
GRANT *`privileges`* ON *`scope`* TO *`account`*;
```

在这里，*`account`*表示要授予权限的账户，*`privileges`*表示权限内容，*`scope`*表示权限的范围或适用级别。*`privileges`*的值可以是`ALL`（或`ALL` `PRIVILEGES`）以指定在给定级别上所有可用的权限，或是一个逗号分隔的一个或多个权限名称，如`SELECT`或`CREATE`。（有关可用权限和未在此处显示的`GRANT`语法的全面讨论，请参阅*MySQL 参考手册*。）

以下示例说明了在每个级别授予权限的语法。

+   在全局级别授予权限使得该账户能够执行管理操作或在任何数据库上执行操作：

    ```
    GRANT FILE ON *.* TO 'user1'@'localhost';
    GRANT CREATE TEMPORARY TABLES, LOCK TABLES ON *.* TO 'user1'@'localhost';
    ```

+   在数据库级别授予权限使得该账户能够在指定数据库中的对象上执行操作：

    ```
    GRANT ALL ON cookbook.* TO 'user1'@'localhost';
    ```

+   在表级别授予权限使得该账户能够在指定表上执行操作：

    ```
    GRANT SELECT ON mysql.user TO 'user1'@'localhost';
    ```

+   在列级别授予权限使得该账户能够在指定表列上执行操作：

    ```
    GRANT SELECT(User,Host), UPDATE(password_expired)
    ON mysql.user TO 'user1'@'localhost';
    ```

+   在过程级别授予权限使得该账户能够在指定存储过程上执行操作：

    ```
    GRANT EXECUTE ON PROCEDURE cookbook.exec_stmt TO 'user1'@'localhost';
    ```

    如果例程是存储函数，请使用`FUNCTION`而不是`PROCEDURE`。

要验证权限分配，请使用`SHOW` `GRANTS`：

```
mysql> `SHOW GRANTS FOR 'user1'@'localhost';`
+--------------------------------------------------------------------------+
| Grants for user1@localhost                                               |
+--------------------------------------------------------------------------+
| GRANT FILE, CREATE TEMPORARY TABLES, LOCK TABLES                         |
| ON *.* TO 'user1'@'localhost'                                            |
| GRANT ALL PRIVILEGES ON `cookbook`.* TO 'user1'@'localhost'              |
| GRANT SELECT, SELECT (User, Host), UPDATE (password_expired)             |
| ON `mysql`.`user` TO 'user1'@'localhost'                                 |
| GRANT EXECUTE ON PROCEDURE `cookbook`.`exec_stmt` TO 'user1'@'localhost' |
+--------------------------------------------------------------------------+
```

要查看自己的权限，请省略`FOR`子句。

`REVOKE`语法通常与`GRANT`类似，但使用`FROM`而不是`TO`：

```
REVOKE *`privileges`* ON *`scope`* FROM *`account`*;
```

因此，要撤销刚刚授予给`'user1'@'localhost'`的权限，请使用这些`REVOKE`语句（并使用`SHOW` `GRANTS`来验证它们是否已被移除）：

```
mysql> `REVOKE FILE ON *.* FROM 'user1'@'localhost';`
mysql> `REVOKE CREATE TEMPORARY TABLES, LOCK TABLES`
    -> `ON *.* FROM 'user1'@'localhost';`
mysql> `REVOKE ALL ON cookbook.* FROM 'user1'@'localhost';`
mysql> `REVOKE SELECT ON mysql.user FROM 'user1'@'localhost';`
mysql> `REVOKE SELECT(User,Host), UPDATE(password_expired)`
    -> `ON mysql.user FROM 'user1'@'localhost';`
mysql> `REVOKE EXECUTE ON PROCEDURE cookbook.exec_stmt`
    -> `FROM 'user1'@'localhost';`
mysql> `SHOW GRANTS FOR 'user1'@'localhost';`
+-------------------------------------------+
| Grants for user1@localhost                |
+-------------------------------------------+
| GRANT USAGE ON *.* TO 'user1'@'localhost' |
+-------------------------------------------+
```

### 删除账户

要删除一个账户，请使用`DROP` `USER`语句：

```
DROP USER 'user1'@'localhost';
```

该语句从所有授权表中移除与账户关联的所有行；您无需使用`REVOKE`先删除其权限。如果账户不存在，则会发生错误。

### 重命名账户

要更改账户名称，请使用`RENAME` `USER`，指定当前名称和新名称：

```
RENAME USER 'currentuser'@'localhost' TO 'newuser'@'localhost';
```

如果当前账户不存在或新账户已经存在，则会发生错误。

# 24.3 实施密码策略

## 问题

您希望确保 MySQL 账户不使用弱密码。

## 解决方案

使用`validate_password`插件实施密码策略。新密码必须符合策略，无论是由 DBA 为新帐户选择还是由现有用户更改密码。

## 讨论

此技术要求启用`validate_password`插件。有关插件安装说明，请参见 Recipe 22.2。

当启用`validate_password`时，它会暴露一组系统变量，使您能够对其进行配置。这些是默认值：

```
mysql> `SHOW VARIABLES LIKE 'validate_password%';`
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password_dictionary_file    |        |
| validate_password_length             | 8      |
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 1      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+
```

假设您希望实施一个策略，强制执行这些密码要求：

+   至少为 10 个字符长

+   包含大写和小写字符

+   包含至少两个数字

+   包含至少一个特殊（非字母数字）字符

要实施该策略，可以在服务器选项文件中添加启用插件并设置配置策略要求的值的选项。例如，将以下行放入您的服务器选项文件中：

```
[mysqld]
plugin-load-add=validate_password.so
validate_password_length=10
validate_password_mixed_case_count=1
validate_password_number_count=2
validate_password_special_char_count=1
```

启动服务器后，请验证这些设置：

```
mysql> `SHOW VARIABLES LIKE 'validate_password%';`
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password_dictionary_file    |        |
| validate_password_length             | 10     |
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 2      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+
```

现在`validate_password`插件防止分配太弱的密码：

```
mysql> `SET PASSWORD = 'weak-password';`
ERROR 1819 (HY000): Your password does not satisfy the current
policy requirements
mysql> `SET PASSWORD = 'Str0ng-Pa33w@rd';`
Query OK, 0 rows affected (0.00 sec)
```

前述说明使`validate_password_policy`系统变量保持其默认值（`MEDIUM`），但您可以更改以控制服务器如何测试密码：

+   `MEDIUM`启用密码长度和数字、大写/小写字符以及特殊字符的测试。

+   要减少严格性，将策略设置为`LOW`，仅启用长度测试。还可以允许更短的密码，减少所需长度（`validate_password_length`）。

+   要更严格地设置策略为`STRONG`，类似于`MEDIUM`但还允许您根据字典文件检查密码，以防止使用文件中的任何单词作为密码。比较不区分大小写。

    要使用字典文件，请在服务器启动时将`validate_password_dictionary_file`的值设置为文件名。该文件应包含每行一个小写单词。MySQL 发行版包括一个*dictionary.txt*文件在*share*目录中，您可以使用，Unix 系统通常有一个*/usr/share/dict/words*文件。

设定密码策略对现有密码没有影响。若要求用户选择符合策略的新密码，需将当前密码过期（见 Recipe 24.5）。

# 24.4 检查密码强度

## 问题

您想要分配或更改密码，但首先验证它是否强密码。

## 解决方案

使用`VALIDATE_PASSWORD_STRENGTH()`函数。

## 讨论

`validate_password`插件不仅实施新密码的策略，还提供了一个 SQL 函数，`VALIDATE_PASSWORD_STRENGTH()`，可用于测试潜在密码的强度。此函数的用途包括：

+   管理员希望检查分配给新帐户的密码。

+   一个个体用户想要选择一个新密码，但事先希望得到它的强度保证。

要使用`VALIDATE_PASSWORD_STRENGTH()`，必须启用`validate_password`插件。有关插件安装说明，请参见 Recipe 22.2。

`VALIDATE_PASSWORD_STRENGTH()`返回一个从 0（弱）到 100（强）的值：

```
mysql> `SELECT VALIDATE_PASSWORD_STRENGTH('abc') ;`
+-----------------------------------+
| VALIDATE_PASSWORD_STRENGTH('abc') |
+-----------------------------------+
|                                 0 |
+-----------------------------------+
mysql> `SELECT VALIDATE_PASSWORD_STRENGTH('weak-password');`
+---------------------------------------------+
| VALIDATE_PASSWORD_STRENGTH('weak-password') |
+---------------------------------------------+
|                                          50 |
+---------------------------------------------+
mysql> `SELECT VALIDATE_PASSWORD_STRENGTH('Str0ng-Pa33w@rd');`
+-----------------------------------------------+
| VALIDATE_PASSWORD_STRENGTH('Str0ng-Pa33w@rd') |
+-----------------------------------------------+
|                                           100 |
+-----------------------------------------------+
```

# 24.5 密码过期

## 问题

您希望用户选择一个新的 MySQL 密码。

## 解决方案

`ALTER USER`语句会使密码过期。

## 讨论

MySQL 5.6.7 及以上版本提供了一个`ALTER USER`语句，使管理员可以使账户的密码过期：

```
ALTER USER 'cbuser'@'localhost' PASSWORD EXPIRE;
```

这里有一些密码过期的用途：

+   您可以实施一个策略，要求新用户在首次连接时选择一个新密码：立即使每个新创建的账户的密码过期。

+   如果您对可接受密码施加了更严格的策略（参见 Recipe 24.3），您可以使所有现有密码过期，要求每个用户选择一个符合更严格要求的新密码。

`ALTER USER` 影响单个账户。它通过将适当的`mysql.user`行的`password_expired`列设置为`Y`来工作。为了 <q>欺骗</q> 并立即使所有非匿名账户的密码过期，请执行以下操作（匿名用户无法重置其密码，因此使这些用户的密码过期会产生相同效果）：

```
UPDATE mysql.user SET password_expired = 'Y' WHERE User <> '';
FLUSH PRIVILEGES;
```

或者，为了影响所有账户但避免直接修改授权表，使用一个存储过程，循环遍历所有账户并为每个执行`ALTER USER`：

```
CREATE PROCEDURE expire_all_passwords()
BEGIN
  DECLARE done BOOLEAN DEFAULT FALSE;
  DECLARE account TEXT;
  DECLARE cur CURSOR FOR
    SELECT CONCAT(QUOTE(User),'@',QUOTE(Host)) AS account
    FROM mysql.user WHERE User <> '';
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

  OPEN cur;
  expire_loop: LOOP
    FETCH cur INTO account;
    IF done THEN
      LEAVE expire_loop;
    END IF;
    CALL exec_stmt(CONCAT('ALTER USER ',account,' PASSWORD EXPIRE'));
  END LOOP;
  CLOSE cur;
END;
```

该过程需要`exec_stmt()`辅助程序（参见 Recipe 11.6）。用于创建这些例程的脚本位于`recipes`分发的*routines*目录中。

# 24.6 分配一个新密码给自己

## 问题

您想要更改您的密码。

## 解决方案

使用`ALTER USER`或`SET PASSWORD`语句。

## 讨论

要分配一个新密码给自己，请使用`SET PASSWORD`语句：

```
SET PASSWORD = '*`my-new-password`*';
```

`SET PASSWORD` 允许使用`FOR`子句，您可以指定哪个账户获得新密码：

```
SET PASSWORD FOR '*`user_name`*'@'*`host_name`*' = '*`my-new-password`*';
```

这种后续语法主要面向 DBA，因为它需要对`mysql`数据库具有`UPDATE`权限。

或者，使用`ALTER USER`语句：

```
ALTER USER '*`user_name`*'@'*`host_name`*' IDENTIFIED BY '*`my-new-password`*';
```

如果您想要使用`ALTER USER`语句为自己分配密码，您可以首先通过运行函数*CURRENT_USER*检查您的账户名：

```
mysql> `SELECT` `CURRENT_USER``(``)``;`
+----------------+ | CURRENT_USER() |
+----------------+ | cbuser@%       |
+----------------+ 1 row in set (0.00 sec)
```

要检查您考虑的密码的强度，请使用`VALIDATE_PASSWORD_STRENGTH()`函数（参见 Recipe 24.4）。

# 24.7 重置已过期密码

## 问题

由于您的 DBA 使您的密码过期，您无法使用 MySQL。

## 解决方案

分配一个新密码给自己。

## 讨论

如果 MySQL 管理员已经使您的密码过期，MySQL 会允许您连接，但不能做其他任何事情：

```
$ `mysql --user=cbuser --password`
Enter password: `******`
mysql> `SELECT CURRENT_USER();`
ERROR 1820 (HY000): You must SET PASSWORD before executing this statement
```

如果您看到该消息，请重置您的密码，以便您可以正常工作：

```
mysql> `SET PASSWORD = 'my-new-password';`
Query OK, 0 rows affected (0.00 sec)

mysql> `SELECT CURRENT_USER();` -- now you can work again
+------------------+
| CURRENT_USER()   |
+------------------+
| cbuser@localhost |
+------------------+
1 row in set (0.00 sec)
```

从技术上讲，MySQL 不要求用一个 *新* 密码替换过期的密码，因此你可以用当前的密码来取消过期状态。唯一的例外是，如果密码策略变得更加严格，你当前的密码不再符合条件，就必须选择一个更强的密码。

关于如何修改密码的更多信息，请参见 Recipe 24.6。

# 24.8 查找和移除匿名账户

## 问题

你希望确保你的 MySQL 服务器只能被与特定用户名关联的账户使用。

## 解决方案

识别并移除匿名账户。

## 讨论

一个 <q>匿名</q> 账户是指账户名中用户部分为空，例如 `''@'localhost'`。空用户匹配任何名称，因为匿名账户的目的是允许知道其密码的任何人从指定的主机连接（在这种情况下是 `localhost`）。这样做是为了方便，因为数据库管理员不需要为不同的用户设置单独的账户。但这也存在安全风险：

+   这类账户通常没有密码，使得它们可以在无需任何认证的情况下使用。

+   你无法将数据库活动与特定用户关联起来（例如，通过检查服务器查询日志或检查 `SHOW PROCESSLIST` 输出），这使得更难以确定谁在执行何种操作。

如果以上几点让你确信匿名账户并不是好事，请使用以下说明来识别并移除它们：

1.  对于匿名账户，`mysql.user` 行中的 `User` 列为空，因此你可以这样识别它们：

    ```
    mysql> `SELECT User, Host FROM mysql.user WHERE User = '';`
    +------+---------------+
    | User | Host          |
    +------+---------------+
    |      | %.example.com |
    |      | localhost     |
    +------+---------------+
    ```

1.  `SELECT` 输出显示了两个匿名账户。使用相应的账户名和 `DROP USER` 语句分别移除它们：

    ```
    mysql> `DROP USER ''@'localhost';`
    mysql> `DROP USER ''@'%.example.com';`
    ```

# 24.9 修改 <q>任何主机</q> 和 <q>多主机</q> 账户

## 问题

你希望确保 MySQL 账户不能从过于广泛的主机使用。

## 解决方案

查找和修复主机部分包含 `%` 或 `_` 的账户。

## 讨论

MySQL 账户名的主机部分可以包含 SQL 模式字符 `%` 和 `_`（见 Recipe 7.10）。这些名称匹配任何与模式匹配的主机的客户端连接尝试。例如，账户 `'user1'@'%'` 允许 `user1` 从任何主机连接，而 `'user2'@'%.example.com'` 允许 `user2` 从 `example.com` 域中的任何主机连接。

账户名中主机部分的模式提供了便利，使得数据库管理员可以创建一个允许从多个主机连接的账户。但这同时增加了安全风险，因为这样做会增加入侵者尝试连接的主机数量。如果你认为这是一个问题，识别这些账户并删除它们或将主机部分修改为更具体的。

有几种方法可以查找在主机部分包含 `%` 或 `_` 的账户。以下是两种方法：

```
WHERE Host LIKE '%\%%' OR Host LIKE '%\_%';
WHERE Host REGEXP '[%_]';
```

`LIKE` 表达式更复杂，因为我们必须单独查找每个模式字符并转义它以搜索文字实例。`REGEXP` 表达式不需要转义，因为这些字符在正则表达式中不是特殊字符，字符类允许使用单个模式找到这两个。因此，让我们使用该表达式：

1.  在 `mysql.user` 表中如此识别模式主机账户：

    ```
    mysql> `SELECT User, Host FROM mysql.user WHERE Host REGEXP '[%_]';`
    +-------+---------------+
    | User  | Host          |
    +-------+---------------+
    | user1 | %             |
    | user2 | %.example.com |
    | user3 | _.example.com |
    +-------+---------------+
    ```

1.  要移除已识别的账户，使用 `DROP` `USER`：

    ```
    mysql> `DROP USER 'user1'@'%';`
    mysql> `DROP USER 'user3'@'_.example.com';`
    ```

    或者，重命名账户以使主机部分更具体：

    ```
    mysql> `RENAME USER 'user2'@'%.example.com' TO 'user2'@'host17.example.com';`
    ```

# 24.10 使用 TLS（SSL）

## 问题

您希望在 MySQL 客户端和服务器之间加密流量。

## 解决方案

使用 TLS（传输层安全）协议。

## 讨论

MySQL 在客户端和服务器之间加密流量时，并不使用除标准 TCP 协议外的任何其他内容。因此，如果有人想要读取传输的数据，无论是从客户端到服务器还是反过来，他们可以轻松地使用 *tcpdump* 和类似工具完成。任何敏感信息，比如用户密码或存储的信用卡号码，都可能被泄露。为了防止这种情况发生，MySQL 支持 TLS 协议来保护通信。

###### 注意

现代版本的 MySQL 使用 TLS 协议来加密客户端和服务器之间的流量。然而，由于历史原因，配置选项和用户参考手册通常将 TLS 称为 SSL（安全套接字层），尽管后者已不再使用，因为其加密较弱。在本书中，我们将尽可能在文本中使用 TLS 这一术语。

要在 MySQL 客户端和服务器之间保护流量，您需要：

在服务器端

+   启用 `ssl` 选项。这是默认值，您只需确保在配置文件中未禁用它。

+   可用于验证证书的证书颁发机构（CA）文件。它可以是单个文件，由 `ssl_ca` 选项指定，或者是包含多个此类文件的目录路径，由 `ssl_capath` 选项指定。

+   由 `ssl_cert` 选项指定的公钥证书文件。此证书将发送给客户端，用于对抗客户端的证书颁发机构进行身份验证。

+   由 `ssl_key` 选项指定的私钥。

在客户端

+   为 `ssl-mode` 选项指定以下值之一：

    `PREFERRED`

    如果服务器支持 TLS，则建立加密连接，并在不支持时回退到未加密连接。这是默认值。

    `REQUIRED`

    如果服务器支持 TLS，则建立加密连接，并在不支持时失败连接尝试。

    `VERIFY_CA`

    执行与 `REQUIRED` 相同的检查，并额外验证服务器 CA 文件与配置的 CA 证书的一致性。

    `VERIFY_IDENTITY`

    执行与 `VERIFY_CA` 相同的检查，并额外进行主机名验证。也就是说，服务器证书应在 `"Subject Alternative Name"` 或 `"Common Name"` 字段中包含客户端主机名。

    值`DISABLED`会禁用 TLS 连接，如果需要加密客户端-服务器流量，则不应使用。

+   可用于验证证书的证书颁发机构（CA）文件。可以是单个文件，由选项`ssl_ca`指定，也可以是包含多个此类文件的目录路径，由选项`ssl_capath`指定。

+   由选项`ssl_cert`指定的公钥证书文件。此证书将发送到服务器，用于根据服务器的证书颁发机构进行身份验证。

+   由选项`ssl_key`指定的私钥。

###### 注意

所有的证书颁发机构、证书和密钥文件应该采用`PEM`格式。

如果 MySQL 服务器启用了`ssl`选项，但其他与加密相关的选项为空值，则会在数据目录中搜索 TLS 密钥和证书。如果找到，将使用它们。否则，将禁用 TLS 支持。

一旦您具备所有这些先决条件，可以测试 TLS 连接：

```
$ `mysql`
mysql> `\s`
--------------
../bin/mysql  Ver 8.0.21 for Linux on x86_64 (Source distribution)

Connection id:		534
Current database:	
Current user:		root@localhost
SSL:			Cipher in use is TLS_AES_256_GCM_SHA384
...
mysql> `SHOW VARIABLES LIKE 'have_ssl';`
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| have_ssl      | YES   |
+---------------+-------+
1 row in set (0.01 sec)
```

MySQL 支持进一步限制 TLS 连接的选项，比如`ssl-cipher`，要求只使用指定的加密算法。详细信息请参考 MySQL 用户参考手册中的[“配置 MySQL 使用加密连接”](https://dev.mysql.com/doc/refman/8.0/en/using-encrypted-connections.html)。

### 创建自签名证书

MySQL 分发包含*mysql_ssl_rsa_setup*命令，可用于创建自签名密钥和证书。它调用*openssl*命令，可以这样使用：

```
$ `mysql_ssl_rsa_setup --datadir=./data`
Ignoring -days; not generating a certificate
Generating a RSA private key
..............................................................................+++++
..............+++++
writing new private key to 'ca-key.pem'
-----
Ignoring -days; not generating a certificate
Generating a RSA private key
....................................+++++
...........................................+++++
writing new private key to 'server-key.pem'
-----
Ignoring -days; not generating a certificate
Generating a RSA private key
..............................................................................+++++
.....................................+++++
writing new private key to 'client-key.pem'
-----
```

完成后，将在数据目录中创建以下表中列出的文件 Table 24-2

表 24-2\. *mysql_ssl_rsa_setup*创建的文件

| 文件 | 描述 |
| --- | --- |
| `ca.pem` | 自签名证书颁发机构（CA） |
| `ca-key.pem` | CA 私钥 |
| `server-cert.pem` | 服务器证书 |
| `server-key.pem` | 服务器密钥 |
| `client-cert.pem` | 客户端证书 |
| `client-key.pem` | 客户端密钥 |
| `private_key.pem` | RSA 私钥，用于未加密连接的帐户，由`sha256_password`或`caching_sha2_password`插件认证 |
| `public_key.pem` | RSA 公钥，用于未加密连接的帐户，由`sha256_password`或`caching_sha2_password`插件认证 |

*mysql_ssl_rsa_setup*创建的密钥和证书非常基础，不包含像`"Common Name"`这样的字段。如果希望向 TLS 文件添加这些自定义值，需要手动创建。本书不包含此过程的详细说明，因为网上有大量文档，包括 MySQL 用户参考手册中的[创建 SSL 和 RSA 证书及密钥](https://dev.mysql.com/doc/refman/8.0/en/creating-ssl-rsa-files.html)。或者，您可以使用*mysql_ssl_rsa_setup*命令的`--verbose`选项进行测试运行，它会打印出执行的*openssl*命令。然后您只需重复执行这些命令并加入自定义选项即可。

###### 提示

如果您只想测试 MySQL TLS 连接的工作原理，并且不想创建任何新的密钥和证书，您可以使用 MySQL 安装中`mysql-test/std_data`目录中的[MySQL 测试套件](https://dev.mysql.com/doc/extending-mysql/8.0/en/mysql-test-suite.html)中的标准密钥和证书。

# 24.11 使用角色

## 问题

您希望将相同的权限集授予不同的用户，但不希望他们共享同一个用户帐户。

## 解决方案

使用角色。

## 讨论

当多人使用 MySQL 安装时，您可能需要为其中一些人提供类似的特权。例如，应用程序用户可能需要访问其应用程序数据库中的表，而管理员可能需要执行管理命令。当您只有一个应用程序用户或一个数据库管理员时，可以简单地创建两个用户帐户。但是，当您的组织和 MySQL 使用增长时，您可能需要允许不同的人执行相同的任务。

您可以通过将单个用户帐户共享给不同的人来解决此类问题。但由于各种原因，包括用户离开公司或应该失去对数据库的访问权限的情况，这种做法是不安全的。或者，如果组中的某人泄露了他们的访问凭据，所有数据库用户都会受到威胁。

另一个解决方案是为各个用户帐户复制特权列表。虽然这样更安全，但在需要添加或删除权限时容易出错。对几十个用户手动执行此操作可能很容易导致错误。

为解决这些缺点，MySQL 8.0 引入了角色，实际上是特权的命名集合。

您可以像创建任何其他用户帐户一样创建角色。您只需不需要为其指定访问凭据。

```
mysql> `CREATE ROLE cookbook, admin;`
Query OK, 0 rows affected (0.00 sec)
```

在上面的列表中，我们创建了一个名为`cookbook`的角色，该角色将访问 cookbook 数据库，以及一个名为`admin`的角色，用于数据库管理。

下一步是为我们的新角色分配权限。

```
mysql> `GRANT SELECT, INSERT, UPDATE, DELETE, EXECUTE ON cookbook.* TO 'cookbook';`
Query OK, 0 rows affected (0.00 sec)

mysql> `GRANT GROUP_REPLICATION_ADMIN, PERSIST_RO_VARIABLES_ADMIN,` 
    -> `REPLICATION_SLAVE_ADMIN, RESOURCE_GROUP_ADMIN,` 
    -> `ROLE_ADMIN, SYSTEM_VARIABLES_ADMIN, SYSTEM_USER,`
    -> `RELOAD, SHUTDOWN ON *.* to 'admin';`
Query OK, 0 rows affected (0.01 sec)
```

设置角色后，我们可以将其分配给不同的用户。例如，要将对`cookbook`数据库的访问权限分配给用户`cbuser`、`sveta`和`alkin`，请使用以下命令：

```
mysql> `GRANT cookbook TO cbuser;`
Query OK, 0 rows affected (0.01 sec)

mysql> `GRANT cookbook TO sveta;`
Query OK, 0 rows affected (0.00 sec)

mysql> `GRANT cookbook TO alkin;`
Query OK, 0 rows affected (0.00 sec)
```

要授予用户`paul`和`amelia`管理员访问权限，请使用以下命令：

```
mysql> `GRANT admin TO paul;`
Query OK, 0 rows affected (0.00 sec)

mysql> `GRANT admin TO amelia;`
Query OK, 0 rows affected (0.01 sec)
```

撤销角色访问权限与授予访问权限一样简单：

```
mysql> `REVOKE cookbook FROM sveta;`
Query OK, 0 rows affected (0.01 sec)
```

## 另请参阅

MySQL 支持其他与角色相关的操作，例如为新添加的用户设置默认角色或激活和停用角色。有关 MySQL 中角色的更多信息，请参见[MySQL 用户参考手册中的使用角色](https://dev.mysql.com/doc/refman/8.0/en/roles.html)。

# 24.12 使用视图保护数据访问

## 问题

您希望仅允许用户访问某些查询结果，但不希望他们看到存储在表中的实际数据。

## 解决方案

使用视图。

## 讨论

您可能希望某些用户能够访问查询结果，但不希望他们访问存储在表中的实际数据。例如，统计部门可能希望了解医院内的患者数量、其性别和年龄分布，以及这些数据与康复率的相关性，但不应访问患者的实际数据，例如他们的姓名、身份证号或能够关联其身份和诊断的信息。

要实现此目标，您可以创建一个视图，查询特定数据，并仅为此视图授予特定用户访问权限。

考虑一个`patients`表：

```
mysql> `SHOW CREATE TABLE patients\G`
*************************** 1\. row ***************************
       Table: patients
Create Table: CREATE TABLE `patients` (
  `id` int NOT NULL AUTO_INCREMENT,
  `national_id` char(32) DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  `surname` varchar(255) DEFAULT NULL,
  `gender` enum('F','M') DEFAULT NULL,
  `age` tinyint unsigned DEFAULT NULL,
  `additional_data` json DEFAULT NULL,
  `diagnosis` varchar(255) DEFAULT NULL,
  `result` enum('R','N','D') DEFAULT NULL ↩
    COMMENT 'R=Recovered, N=Not Recovered, D=Dead',
  `date_arrived` date NOT NULL,
  `date_departed` date DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=21 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```

如果您希望为统计部门提供对`gender`、`age`、`diagnosis`、`result`列的访问权限，但希望限制对`national_id`、`name`、`surname`和`additional_data`的访问权限，您还可以让他们了解患者在医院中停留了多少天，以及他们到达的月份和年份，但不想让他们探索实际的到达和离开日期。换句话说，限制对`date_arrived`和`date_departed`的访问，同时提供基于这些列中存储的值计算出的数据。

您可以通过创建视图来实现此目标：

```
mysql> `CREATE VIEW patients_statistics AS` 
    -> `SELECT age, gender, diagnosis, result,` 
    ->  `datediff(date_departed, date_arrived) as recovered_time,`
    -> `MONTH(date_arrived) AS month_arrived,` 
    ->  `YEAR(date_arrived) AS year_arrived`
    ->  `FROM patients;`
Query OK, 0 rows affected (0.03 sec)
```

然后为统计部门创建一个用户，该用户只能对此视图进行只读访问，并且不能访问底层表。

```
mysql> `CREATE USER statistics;`
Query OK, 0 rows affected (0.03 sec)

mysql> `GRANT SELECT ON cookbook.patients_statistics TO statistics;`
Query OK, 0 rows affected (0.02 sec)
```

现在统计部门可以登录并运行分析查询，例如找出最常见的诊断或每月有多少这样的诊断患者到达。

```
# `mysql cookbook -A -ustatistics`
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 17
...
mysql> `SELECT diagnosis, AVG(recovered_time) AS recovered_time_avg, COUNT(*) AS cases`
    -> `FROM patients_statistics WHERE year_arrived='2020'`
    -> `GROUP BY diagnosis ORDER BY cases DESC;`
+---------------+--------------------+-------+
| diagnosis     | recovered_time_avg | cases |
+---------------+--------------------+-------+
| Data Phobia   |            24.8333 |     6 |
| Diabetes      |            10.0000 |     4 |
| Asthma        |            10.3333 |     3 |
| Arthritis     |            22.0000 |     3 |
| Appendicitis  |             5.5000 |     2 |
| Breast Cancer |            75.0000 |     2 |
+---------------+--------------------+-------+
6 rows in set (0.00 sec)

mysql> `SELECT diagnosis, month_arrived, COUNT(*) AS month_cases`
    -> `FROM patients_statistics WHERE diagnosis='Data Phobia' AND year_arrived='2020'`
    -> `GROUP BY diagnosis, month_arrived ORDER BY month_arrived;`
+--------------+---------------+-------------+
| diagnosis    | month_arrived | month_cases |
+--------------+---------------+-------------+
| Data Phobia  |             4 |           1 |
| Data Phobia  |             9 |           2 |
| Data Phobia  |            10 |           2 |
| Data Phobia  |            11 |           1 |
+--------------+---------------+-------------+
4 rows in set (0.00 sec)
```

但他们将无法直接访问表数据：

```
mysql> `SELECT * FROM patients;`
ERROR 1142 (42000): SELECT command denied to user 'statistics'@'localhost' ↩
                                          for table 'patients'

mysql> `SELECT diagnosis, result, date_arrived FROM patients;`
ERROR 1142 (42000): SELECT command denied to user 'statistics'@'localhost' ↩
                                          for table 'patients'
```

###### 注意

视图支持`SQL SECURITY`子句，允许在执行视图时指定安全上下文。有关详细信息，请参见 Recipe 24.13。

## 另请参阅

获取有关使用视图的更多信息，请参阅 Recipe 5.7。

# 24.13 使用存储过程来保护数据修改

## 问题

您希望让用户修改其个人数据，但希望防止他们访问其他人的类似数据。

## 解决方案

使用存储过程。

## 讨论

您可能希望让用户查看和更改自己的个人信息。例如，患者可以结婚、改姓或决定添加关于自己的新附加信息，例如地址、体重等等。但您不希望他们看到其他患者的类似信息。在这种情况下，仅在列级别上限制访问不起作用。

存储过程支持`SQL SECURITY`子句，允许您指定是否要以创建过程的`DEFINER`用户的访问权限执行该过程，还是以当前执行该过程的`INVOKER`用户的访问权限执行。

在我们的情况下，我们不希望授予`INVOKER`访问敏感列中存储的数据的任何特权。因此，我们需要将这些特权授予过程的`DEFINER`，并指定参数`SQL SECURITY DEFINER`。

###### 提示

`SQL SECURITY`的默认值为`DEFINER`，因此此子句可以省略。

为了说明这一点，让我们从上一个示例中获取`patients`表。但现在我们将仅访问包含敏感数据的列。

```
mysql> `SHOW CREATE TABLE patients\G`
*************************** 1\. row ***************************
       Table: patients
Create Table: CREATE TABLE `patients` (
  ...
  `national_id` char(32) DEFAULT NULL,
  `name` varchar(255) DEFAULT NULL,
  `surname` varchar(255) DEFAULT NULL,
  ...
  `additional_data` json DEFAULT NULL,
  ...
```

首先，让我们为我们的存储过程准备一个用户，该用户将作为`DEFINER`。我们不希望任何人使用此帐户，除非是存储过程，所以首先让我们安装`mysql_no_login`认证插件。

```
mysql> `INSTALL PLUGIN mysql_no_login SONAME 'mysql_no_login.so';`
Query OK, 0 rows affected (0.01 sec)
```

然后让我们创建用户帐户，并授予它对`patients`表的访问权限。

```
mysql> `CREATE USER sp_access IDENTIFIED WITH mysql_no_login;`
Query OK, 0 rows affected (0.00 sec)

mysql> `GRANT SELECT, UPDATE ON cookbook.patients TO sp_access;`
Query OK, 0 rows affected (0.01 sec)
```

现在让我们创建一个过程，该过程将返回由`national_id`标识的患者的敏感数据。

```
mysql> `delimiter $$`
mysql> `CREATE DEFINER='sp_access'@'%' PROCEDURE get_patient_data(IN nat_id CHAR(32))` 
    ->  `SQL SECURITY DEFINER`
    ->  `BEGIN`
    -> `SELECT name, surname, gender, age, additional_data` 
    ->  `FROM patients WHERE national_id=nat_id;`
    ->  `END`
    -> `$$`
Query OK, 0 rows affected (0.01 sec)

mysql> `delimiter ;`
```

和一个更新记录的过程。

```
mysql> `delimiter $$`
mysql> `CREATE DEFINER='sp_access'@'%' PROCEDURE update_patient_data(`
    ->  `IN nat_id CHAR(32),`
    ->  `IN new_name varchar(255),`
    ->  `IN new_surname varchar(255),`
    ->  `IN new_additional_data JSON)`
    ->  `SQL SECURITY DEFINER`
    ->  `BEGIN`
    -> `UPDATE patients` 
    ->  `SET name=COALESCE(new_name, name),`
    ->  `surname=COALESCE(new_surname, surname),`
    ->  `additional_data=JSON_MERGE_PATCH(COALESCE(additional_data, '{}'),`
    ->  `COALESCE(new_additional_data, '{}'))`
    ->  `WHERE national_id=nat_id;`
    ->  `END`
    -> `$$`
Query OK, 0 rows affected (0.01 sec)

mysql> `delimiter ;`
```

然后，为我们的`DEFINER`添加执行这些过程的权限。

```
mysql> `GRANT EXECUTE ON PROCEDURE cookbook.get_patient_data TO sp_access;`
Query OK, 0 rows affected (0.01 sec)

mysql> `GRANT EXECUTE ON PROCEDURE cookbook.update_patient_data TO sp_access;`
Query OK, 0 rows affected (0.00 sec)
```

最后，让我们创建一个用户，该用户将在没有任何其他特权的情况下使用这些过程。

```
mysql> `CREATE USER patient;`
Query OK, 0 rows affected (0.01 sec)

mysql> `GRANT EXECUTE ON PROCEDURE cookbook.get_patient_data TO patient;`
Query OK, 0 rows affected (0.02 sec)

mysql> `GRANT EXECUTE ON PROCEDURE cookbook.update_patient_data TO patient;`
Query OK, 0 rows affected (0.01 sec)
```

现在让我们登录为这个用户，并检查我们的过程如何工作。

```
mysql> `SHOW GRANTS;`
+------------------------------------------------------------------------------+
| Grants for patient@%                                                         |
+------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `patient`@`%`                                          |
| GRANT EXECUTE ON PROCEDURE `cookbook`.`get_patient_data` TO `patient`@`%`    |
| GRANT EXECUTE ON PROCEDURE `cookbook`.`update_patient_data` TO `patient`@`%` |
+------------------------------------------------------------------------------+
3 rows in set (0.00 sec)

mysql> `CALL update_patient_data('89AR642465', NULL, 'Johnson',`
    -> `' {"Height": 165, "Weight": 55, "Hair color": "Blonde"}');`
Query OK, 1 row affected (0.00 sec)

mysql> `CALL get_patient_data('89AR642465')\G`
*************************** 1\. row ***************************
           name: Mary
        surname: Johnson
         gender: F
            age: 24
additional_data: {"Height": 165, "Weight": 55, "Hair color": "Blonde"}
1 row in set (0.00 sec)

Query OK, 0 rows affected (0.00 sec)
```

正如您所见，我们可以通过国家身份证号更改特定患者的详细信息，而无需访问其他患者的数据。

## 参见

有关存储过程的更多信息，请参阅第十一章。
