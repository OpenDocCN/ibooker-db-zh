# 第十九章：使用 JSON

# 19.0 介绍

几十年来，关系数据库已被证明非常有效。它们防止了数据的重复和丢失，并且能够快速访问存储的值。然而，业务不断创造新的情景，其中数据需要比关系模型允许的更加灵活。

例如，让我们考虑一个可以访问仅订阅数字内容并留下评论的用户记录。对于这样的用户，只有基本信息 - 姓名、电子邮件地址和密码 - 就足以开始使用。然而，一旦用户开始探索更多选项（例如，需要交付物品），他们可能需要存储其邮寄地址。邮寄地址可能与账单地址不同。用户可能还希望添加一个或几个社交网络账号。

在关系数据库中存储灵活数据的一种方法是将附加数据片段存储在共享每个用户详细信息的引用表中。我们在配方 16.5 和配方 16.6 中讨论了这种技术。

然而，在以下情况下，这种技术可能不是最佳选择。

当主表中只有少数项目在引用表中具有详细信息时

如果您仍然需要了解这些详细信息，当您查询主表中所需字段时，您将需要每次将其与引用表进行连接。这将使查询复杂化并影响性能。

当大多数具体细节可能被忽略时

用户的区域或楼号等详细信息仅适用于请求物品实体交付的用户。对于其他用户，这些字段可以为空，但仍然需要在数据库中保留这些空字段的空间。随着数据库的增长，这将增加显著成本。

当您可能不知道未来需要哪些额外数据时

业务可能需要向数据集合添加附加详细信息。在关系模型中追加这些详细信息意味着在现有表格中创建新的表格和列。这需要模式重新设计和实施更改的维护窗口。这并不总是可行且成本/空间高效。

要解决这些问题，灵活的数据结构（如 JSON）是最佳选择。MySQL 始终允许使用字符串数据类型在文本字段中存储 JSON 值。自 MySQL 版本 5.7 以来，还支持 JSON 数据类型和函数，可以有效地操作 JSON 值。MySQL 结合了 SQL 和 NoSQL 世界的优势。

# 19.1 选择正确的数据类型

## 问题

您希望存储 JSON 值但不知道选择哪种数据类型。

## 解决方案

使用 JSON 数据类型。

## 讨论

JSON 数据可以存储在任何文本或二进制列中。JSON 函数可以正常工作，但是 JSON 特殊数据类型具有许多优点，特别是：

优化性能

JSON 数据被转换为一种格式，允许在文档中快速查找值。

部分更新

对 JSON 元素的更新在原地进行，无需重新编写完整文档。

自动数据验证

当将一个值插入 JSON 数据类型的列时，MySQL 会自动验证它，并在文档无效时产生错误。

以下代码将创建一个具有 JSON 列`author`的表。

```
CREATE TABLE book_authors (
  id     INT NOT NULL AUTO_INCREMENT,
  author JSON NOT NULL,
  PRIMARY KEY (id)
);
```

## 另请参阅

想要关于 JSON 数据类型的更多信息，请参阅 MySQL 用户参考手册中的[JSON 数据类型](https://dev.mysql.com/doc/refman/8.0/en/json.html)。

# 19.2 插入 JSON 值

## 问题

你希望在 MySQL 中存储 JSON 文档。

## 解决方案

使用常规的`INSERT`语句。

## 讨论

JSON 与其他数据类型无异。使用常规的`INSERT`语句将文档添加到表中。

```
mysql> `INSERT` `INTO` `` ` ```书籍作者``` ` `` `VALUES` 
    -> `(``1``,``'{"id": 1, "name": "Paul",` 
    '>       `"books"``:` `[`
    '>  `"Software Portability with imake: Practical Software Engineering",` '>         `"Mysql: The Definitive Guide to Using, Programming,` ↩ `and Administering Mysql 4 (Developer\'s Library)"``,` 
    '> `"MySQL Certification Study Guide",` 
    '> `"MySQL (OTHER NEW RIDERS)"``,` 
    '> `"MySQL Cookbook",` 
    '> `"MySQL 5.0 Certification Study Guide"``,` 
    '> `"Using csh & tcsh: Type Less, Accomplish More` ↩ `(Nutshell Handbooks)",` 
    '> `"MySQL (Developer\'s Library)"``]``,` 
    '>  `"lastname": "DuBois"}'``)``,`
    -> `(``2``,``'{"id": 2, "name": "Alkin",` 
    '> `"books"``:` `[``"MySQL Cookbook"``]``,` 
    '>  `"lastname": "Tezuysal"}'``)``,`
    -> `(``3``,``'{"id": 3, "name": "Sveta",` 
    '> `"books"``:` `[``"MySQL Troubleshooting"``,` `"MySQL Cookbook"``]``,` 
    '>  `"lastname": "Smirnova"}'``)``;`
Query OK, 3 rows affected (0,01 sec)
Records: 3  Duplicates: 0  Warnings: 0
```

# 19.3 验证 JSON

## 问题

你希望确保一个给定的字符串是有效的 JSON。

## 解决方案

使用 JSON 数据类型执行自动验证。使用函数`JSON_VALID`验证字符串。使用 JSON 模式定义 JSON 文档的模式。

## 讨论

函数`JSON_VALID`检查给定文档是否为有效的 JSON。

```
mysql> `SELECT` `JSON_VALID``(``'"name": "Sveta"'``)``;`
+-------------------------------+ | JSON_VALID('"name": "Sveta"') |
+-------------------------------+ |                             0 |
+-------------------------------+ 1 row in set (0,00 sec)

mysql> `SELECT` `JSON_VALID``(``'{"name": "Sveta"}'``)``;`
+---------------------------------+ | JSON_VALID('{"name": "Sveta"}') |
+---------------------------------+ |                               1 |
+---------------------------------+ 1 row in set (0,00 sec)
```

如果一个列定义为 JSON，MySQL 将不允许你插入无效值。此外，错误消息将定位到第一个错误，因此你可以更快地修复它。

```
mysql> `INSERT` `INTO` `book_authors``(``author``)` 
    -> `VALUES` `(``'{"name": "Sveta" "lastname": "Smirnova"'``)``;`
ERROR 3140 (22032): Invalid JSON text: "Missing a comma or '}' after an object↩
member." at position 17 in value for column 'book_authors.author'.
```

如果你不仅想验证一个 JSON 文档，还想使其满足一个模式，请使用函数`JSON_SCHEMA_VALID`。该函数支持 JSON 模式，如[JSON 模式规范草案第 4 版](https://json-schema.org/specification-links.html#draft-4)所述。使用它之前，你需要先定义一个模式，然后将 JSON 值与之进行比较。

函数`JSON_SCHEMA_VALIDATION_REPORT`不仅检查给定的文档是否符合模式，还会报告违反模式的具体部分。

对于表`book_authors`，我们可以定义一个模式，要求必须包含`name`和`lastname`作为必填字段，并且可以选择包含书籍标题数组作为可选元素`books`。我们可以使用以下代码来定义模式：

```
{
"id": "http://www.oreilly.com/mysqlcookbook", ![1](img/1.png)
"$schema": "http://json-schema.org/draft-04/schema#", ![2](img/2.png)
"description": "Schema for the table book_authors", ![3](img/3.png)
"type": "object", ![4](img/4.png)
"properties": { ![5](img/5.png)
  "name": {"type": "string"}, 
  "lastname": {"type": "string"}, 
  "books": {"type": "array"}
}, 
"required":["name", "lastname"] ![6](img/6.png)
}
```

![1](img/#co_nch-json-json-validation-id_co)

模式的唯一标识符

![2](img/#co_nch-json-json-validation-schema_co)

JSON 模式规范。应始终为`http://json-schema.org/draft-04/schema#`

![3](img/#co_nch-json-json-validation-desc_co)

模式的描述

![4](img/#co_nch-json-json-validation-type_co)

根元素的类型

![5](img/#co_nch-json-json-validation-props_co)

属性列表。每个属性都应该被描述。它们应该有一个定义的类型，并可以指定其他验证，如最小和最大值。

![6](img/#co_nch-json-json-validation-req_co)

必填字段列表

如果我们将刚刚定义的模式赋给一个变量，比如`@schema`，我们可以根据这个模式检查 JSON 数据。

```
mysql> `SET` `@``schema` `=` `'{` '> `"id"``:` `"http://www.oreilly.com/mysqlcookbook"``,`
    '> `"$schema": "http://json-schema.org/draft-04/schema#",` '> `"description"``:` `"Schema for the table book_authors"``,`
    '> `"type": "object",` '> `"properties"``:` `{`
    '> `"name": {"type": "string"},` '> `"lastname"``:` `{``"type"``:` `"string"``}``,`
    '> `"books": {"type": "array"}` '> `}``,`
    '> `"required":["name", "lastname"]` '> `}``';` Query OK, 0 rows affected (0,00 sec)

mysql> `SELECT JSON_SCHEMA_VALIDATION_REPORT(@schema,` -> `'``{``"name"``:` `"Sveta"``}``')  AS '``Valid``?``'\G` *************************** 1\. row ***************************
Valid?: {"valid": false, "reason": "The JSON document location '#' failed requirement ↩
'required' at JSON Schema location '#'", "schema-location": "#", ↩
"document-location": "#", "schema-failed-keyword": "required"}
1 row in set (0,00 sec)
```

在这种情况下，验证失败，因为文档仅包含`name`字段，而没有包含另一个必填字段`lastname`。

```
mysql> `SELECT` `JSON_SCHEMA_VALIDATION_REPORT``(``@``schema``,` 
    -> `'{"name": "Sveta", "lastname": "Smirnova"}'``)` `AS` `'Valid?'``;`
+-----------------+ | Valid?          |
+-----------------+ | {"valid": true} |
+-----------------+ 1 row in set (0,00 sec)
```

在这种情况下，文档是有效的，因为它包含所有必需的字段。字段`books`是可选的，不是必需的。

```
mysql> `SELECT` `JSON_SCHEMA_VALIDATION_REPORT``(``@``schema``,` 
    -> `'{"name": "Sveta", "lastname": "Smirnova",` -> `"books": "MySQL Cookbook"}'``)` `AS` `'Valid?'``\``G`
*************************** 1. row ***************************
Valid?: {"valid": false, "reason": "The JSON document location '#/books' failed ↩
requirement 'type' at JSON Schema location '#/properties/books'", ↩
"schema-location": "#/properties/books", "document-location": "#/books", ↩
"schema-failed-keyword": "type"}
1 row in set (0,00 sec)
```

在这种情况下，文档无效，因为成员`books`的类型为`string`而不是模式中定义的`array`。

```
mysql> `SELECT` `JSON_SCHEMA_VALIDATION_REPORT``(``@``schema``,` 
    -> `'{"name": "Sveta", "lastname": "Smirnova",` -> `"books": ["MySQL Troubleshooting", "MySQL Cookbook"]}'``)` `AS` `'Valid?'``;`
+-----------------+ | Valid?          |
+-----------------+ | {"valid": true} |
+-----------------+ 1 row in set (0,00 sec)
```

这份文件修正了元素`books`的类型错误，因此有效。

```
mysql> `SELECT` `JSON_SCHEMA_VALID``(``@``schema``,` `'{"name": "Sveta", "lastname": "Smirnova",` -> `"vehicles": ["Honda CRF 250L"]}'``)` `AS` `'Valid 1?'``,`
    -> `JSON_SCHEMA_VALID``(``@``schema``,` `'{"name": "Alkin", "lastname": "Tezuysal",` -> `"vehicles": "boat"}'``)` `AS` `'Valid 2?'``;`
+----------+----------+ | Valid 1? | Valid 2? |
+----------+----------+ |        1 |        1 |
+----------+----------+ 1 row in set (0,00 sec)
```

这些文档也是有效的，因为`vehicles`属性没有要求：它可能存在，也可能不存在，并且可以是任何类型。

如果您想自动验证表中的 JSON 字段是否符合定义的模式，请使用`CHECK`约束。

```
ALTER TABLE book_authors 
ADD CONSTRAINT CHECK(JSON_SCHEMA_VALID('
{"id": "http://www.oreilly.com/mysqlcookbook", 
 "$schema": "http://json-schema.org/draft-04/schema#", 
 "description": "Schema for the table book_authors", 
 "type": "object", 
 "properties": {
 "name": {"type": "string"}, 
 "lastname": {"type": "string"}, 
 "books": {"type": "array"}},
 "required":["name", "lastname"]} ', 
  author));
```

# 19.4 格式化 JSON 值

## 问题

您希望将 JSON 打印成漂亮的格式。

## 解决方案

使用函数`JSON_PRETTY`。

## 讨论

默认情况下，JSON 被打印为一个长字符串，可能很难阅读。如果希望 MySQL 将其以人类可读的格式打印，请使用函数`JSON_PRETTY`。

```
mysql> `SELECT` `JSON_PRETTY``(``author``)` `FROM` `book_authors``\``G`
*************************** 1. row ***************************
JSON_PRETTY(author): {
  "id": 1,
  "name": "Paul",
  "books": [
    "Software Portability with imake: Practical Software Engineering",
    "Mysql: The Definitive Guide to Using, Programming, ↩
     and Administering Mysql 4 (Developer's Library)",
    "MySQL Certification Study Guide",
    "MySQL (OTHER NEW RIDERS)",
    "MySQL Cookbook",
    "MySQL 5.0 Certification Study Guide",
    "Using csh & tcsh: Type Less, Accomplish More (Nutshell Handbooks)",
    "MySQL (Developer's Library)"
  ],
  "lastname": "DuBois"
}
*************************** 2. row ***************************
JSON_PRETTY(author): {
  "id": 2,
  "name": "Alkin",
  "books": [
    "MySQL Cookbook"
  ],
  "lastname": "Tezuysal"
}
*************************** 3. row ***************************
JSON_PRETTY(author): {
  "id": 3,
  "name": "Sveta",
  "books": [
    "MySQL Troubleshooting",
    "MySQL Cookbook"
  ],
  "lastname": "Smirnova"
}
3 rows in set (0,00 sec)
```

# 19.5 从 JSON 提取值

## 问题

您希望从 JSON 文档中提取值。

## 解决方案

使用函数`JSON_EXTRACT`，或者操作符`->`和`->>`。

## 讨论

JSON 本身没有用处，如果无法从文档中提取值。MySQL 中的 JSON 支持 JSON 路径，可以用于指向 JSON 中的特定元素。JSON 文档的根元素由`$`符号表示。对象成员通过操作符`.`访问，数组成员通过索引在方括号中访问。索引从零开始。您可以使用关键字`to`引用多个数组元素（例如`$.[3 to 5]`。`last`关键字是数组中的最后一个元素的同义词。

通配符`*`表示所有对象成员的所有值，如果在点后使用：`.*`或者所有数组元素如果在方括号内：`[*]`。

表达式`[prefix]**suffix`表示所有以`prefix`开头且以`suffix`结尾的路径。请注意，虽然`suffix`部分是必需的，但`prefix`是可选的。换句话说，JSON Path 表达式不应以双星号结束。

要访问 JSON 元素，请使用函数`JSON_EXTRACT`。

例如，要选择作者的名称，请使用以下 SQL：

```
mysql> `SELECT` `JSON_EXTRACT``(``author``,` `'$.name'``)` `AS` `author` `FROM` `book_authors``;`
+---------+ | author  |
+---------+ | "Paul"  |
| "Alkin" |
| "Sveta" |
+---------+ 3 rows in set (0,00 sec)
```

要从值中删除引号，请使用函数`JSON_UNQUOTE`。

```
mysql> `SELECT` `JSON_UNQUOTE``(``JSON_EXTRACT``(``author``,` `'$.name'``)``)` `AS` `author` `FROM` `book_authors``;`
+--------+ | author |
+--------+ | Paul   |
| Alkin  |
| Sveta  |
+--------+ 3 rows in set (0,00 sec)
```

操作符`->`是函数`JSON_EXTRACT`的别名。

```
mysql> `SELECT` `author``-``>``'$.name'` `AS` `author` `FROM` `book_authors``;`
+---------+ | author  |
+---------+ | "Paul"  |
| "Alkin" |
| "Sveta" |
+---------+ 3 rows in set (0,00 sec)
```

操作符`->>`是`JSON_UNQUOTE(JSON_EXTRACT(...))`的别名。

```
mysql> `SELECT` `author``-``>``>``'$.name'` `AS` `author` `FROM` `book_authors``;`
+--------+ | author |
+--------+ | Paul   |
| Alkin  |
| Sveta  |
+--------+ 3 rows in set (0,00 sec)
```

要分别使用数组索引`0`和`last`提取作者的第一本和最后一本书。

```
mysql> `SELECT` `CONCAT``(``author``-``>``>``'$.name'``,` `' '``,` `author``-``>``>``'$.lastname'``)` `AS` `author``,`
    -> `author``-``>``>``'$.books[0]'` `AS` `` ` ```First` `Book``` ` ```,`

    -> `author``-``>``>``'$.books[last]'` `AS` `` ` ```Last` `Book``` ` `` `FROM` `book_authors``\``G`

*************************** 1. 行 ***************************

    作者：Paul DuBois

第一本书：使用 imake 进行软件可移植性的实际软件工程

最后一本书：MySQL（开发者图书馆）

*************************** 2. 行 ***************************

    作者：Alkin Tezuysal

第一本书：MySQL Cookbook

最后一本书：MySQL Cookbook

*************************** 3. 行 ***************************

    作者：Sveta Smirnova

第一本书：MySQL 故障排除

最后一本书：MySQL Cookbook

一共 3 行（0,00 秒）

```

JSON Path `$.books[*]` will return full array of books. Same will happen if you omit a wildcard and simply refer books array as `$.books`. Expression `$.*` will return all elements of the JSON object as an array.

```

mysql> `SELECT` `author``-``>``'$.*'` `FROM` `book_authors` `WHERE` `author``-``>``>``'$.name'` `=` `'Sveta'``;`

+-----------------------------------------------------------------------+ | author->'$.*'                                                         |

+-----------------------------------------------------------------------+ | [3, "Sveta", ["MySQL 故障排除", "MySQL 食谱"], "Smirnova"] |

+-----------------------------------------------------------------------+ 1 行 (0.00 秒)

```

## See Also

For additional information about JSON Path, see [JSON Path Syntax](https://dev.mysql.com/doc/refman/8.0/en/json.html#json-path-syntax) in the MySQL User Reference Manual.

# 19.6 Searching inside JSON

## Problem

You want to search for JSON documents, containing particular values.

## Solution

Use function `JSON_SEARCH`.

## Discussion

Accessing by key works great but you may want to search for particular values in the JSON documents. MySQL allows you to do it. For example, to find all authors of the book “MySQL Cookbook”, run query:

```

mysql> `SELECT` `author``-``>``>``'$.name'` `AS` `author` `FROM` `book_authors`

    -> `WHERE` `JSON_SEARCH``(``author``,` `'one'``,` `'MySQL 食谱'``)``;`

+--------+ | 作者 |

+--------+ | Paul   |

| Alkin  |
| --- |
| Sveta  |

+--------+ 3 行, 1 警告 (0.00 秒)

```

Function `JSON_SEARCH` takes a JSON document, keyword `one` or `all` and a search string as required arguments and returns found path of the element or elements that contain the searched value. It also supports optional escape character and JSON path arguments.

Similarly to the operator `LIKE` function `JSON_SEARCH` supports wildcards `%` and `_`.

Thus, to search all books, which names start from `MySQL`, use expression:

```

mysql> `SELECT` `author``-``>``>``'$.name'` `AS` `author``,`

    -> `JSON_SEARCH``(``author``,` `'all'``,` `'MySQL%'``)` `AS` `books`

    -> `FROM` `book_authors``\``G`

*************************** 1. 行 ***************************

author: Paul

books: ["$.books[2]", "$.books[3]", "$.books[4]", "$.books[5]", "$.books[7]"]

*************************** 2. 行 ***************************

author: Alkin

books: "$.books[0]"

*************************** 3. 行 ***************************

author: Sveta

books: ["$.books[0]", "$.books[1]"]

3 行 (0.00 秒)

```

When searching for single match you can use return value of the function `JSON_SEARCH` as an argument for the function `JSON_EXTRACT`:

```

mysql> `SELECT` `author``-``>``>``'$.name'` `AS` `author``,`

    -> `JSON_EXTRACT``(``author``,`

    -> `JSON_UNQUOTE``(``JSON_SEARCH``(``author``,` `'one'``,` `'MySQL%'``)``)``)` `AS` `book`

    -> `FROM` `book_authors``;`

+--------+-----------------------------------+ | 作者 | 书籍                              |

+--------+-----------------------------------+ | Paul   | "MySQL 认证考试指南" |

| Alkin  | "MySQL 食谱"                  |
| --- | --- |
| Sveta  | "MySQL 故障排除"           |

+--------+-----------------------------------+ 3 行 (0.00 秒)

```

# 19.7 Inserting New Elements into a JSON Document

## Problem

You want to insert new elements into a JSON document.

## Solution

Use functions `JSON_INSERT`, `JSON_ARRAY_APPEND` and `JSON_ARRAY_INSERT`.

## Discussion

You may want to not only search inside JSON values but modify them. MySQL supports a number of functions which can modify JSON. The most wonderful thing about them is that they do not replace the JSON document as regular string functions do. They rather perform updates in place. This allows you to modify JSON values effectively.

MySQL functions allow you to append, remove, and replace parts of JSON as well as merge two or more documents into one. They all take the original document as an argument, a path that needs to be modified, and a new value.

To insert new value into a JSON object, use the function `JSON_INSERT`. Thus, to add information about current author’s work place call the function as follow.

```

UPDATE book_authors SET author = JSON_INSERT(author, '$.work', 'Percona')

WHERE author->>'$.name' IN ('Sveta', 'Alkin');

```

To add a book into the end of the `book` array use function `JSON_ARRAY_APPEND`.

```

UPDATE book_authors SET author = JSON_ARRAY_APPEND(author, '$.books',

'MySQL 性能模式实战') WHERE author->>'$.name' = 'Sveta';

```

This will add new book into the end of the array.

```

mysql> `SELECT` `JSON_PRETTY``(``author``)` `FROM` `book_authors`

    -> `WHERE` `author``-``>``>``'$.name'` `=` `'Sveta'``\``G`

*************************** 1. 行 ***************************

JSON_PRETTY(author): {

"id": 3,

"name": "Sveta",

"work": "Percona",

"books": [

    "MySQL 故障排除",

    "MySQL 食谱",

    "MySQL 性能模式实战"

],

"lastname": "Smirnova"

}

1 行 (0.00 秒)

```

To add an element into a specific place, use the function `JSON_ARRAY_INSERT`.

```

UPDATE book_authors SET author = JSON_ARRAY_INSERT(author, '$.books[0]',

'MySQL 初学者入门') WHERE author->>'$.name' = 'Alkin';

```

This will insert a new book into the beginning of the array.

```

mysql> `SELECT` `JSON_PRETTY``(``author``)`

    -> `FROM` `book_authors` `WHERE` `author``-``>``>``'$.name'` `=` `'Alkin'``\``G`

*************************** 1. 行 ***************************

JSON_PRETTY(author): {

"id": 2,

"name": "Alkin",

"work": "Percona",

"books": [

    "MySQL 初学者入门",

    "MySQL Cookbook"

],

"lastname": "Tezuysal"

}

1 行数据（0,00 秒）

```

# 19.8 Updating JSON

## Problem

You want to update a JSON value.

## Solution

Use the functions `JSON_REPLACE` and `JSON_SET`.

## Discussion

While we were working on this book, Alkin switched his work place. So the content of the table needs to be updated. The function `JSON_REPLACE` replaces a given path with the new value.

```

UPDATE book_authors SET author = JSON_REPLACE(author, '$.work', 'PlanetScale')

WHERE author->>'$.name' = 'Alkin';

```

However, the function `JSON_REPLACE` will do nothing if a record, that needs to be replaced, does not exist in the document.

```

mysql> `UPDATE` `book_authors` `SET` `author` `=` `JSON_REPLACE``(``author``,` `'$.work'``,` `'Oracle'``)`

    -> `WHERE` `author``-``>``>``'$.name'` `=` `'Paul'``;`

查询成功，0 行受影响（0,00 秒）

匹配行数: 1  修改行数: 0  警告数: 0

mysql> `SELECT` `author``-``>``>``'$.work'` `FROM` `book_authors` `WHERE` `author``-``>``>``'$.name'` `=` `'Paul'``;`

+-------------------+ | author->>'$.work' |

+-------------------+ | NULL              |

+-------------------+ 1 行数据（0,00 秒）

```

To resolve this problem, use the function `JSON_SET` to update the document if the path exists, or to insert a new value if the path does not.

```

mysql> `UPDATE` `book_authors` `SET` `author` `=` `JSON_SET``(``author``,` `'$.work'``,` `'MySQL'``)`

    -> `WHERE` `author``-``>``>``'$.name'` `=` `'Paul'``;`

查询成功，1 行受影响（0,01 秒）

匹配行数: 1  修改行数: 1  警告数: 0

mysql> `SELECT` `author``-``>``>``'$.work'` `FROM` `book_authors` `WHERE` `author``-``>``>``'$.name'` `=` `'Paul'``;`

+-------------------+ | author->>'$.work' |

+-------------------+ | MySQL             |

+-------------------+ 1 行数据（0,00 秒）

mysql> `UPDATE` `book_authors` `SET` `author` `=` `JSON_SET``(``author``,` `'$.work'``,` `'Oracle'``)`

    -> `WHERE` `author``-``>``>``'$.name'` `=` `'Paul'``;`

查询成功，1 行受影响（0,00 秒）

匹配行数: 1  修改行数: 1  警告数: 0

mysql> `SELECT` `author``-``>``>``'$.work'` `FROM` `book_authors` `WHERE` `author``-``>``>``'$.name'` `=` `'Paul'``;`

+-------------------+ | author->>'$.work' |

+-------------------+ | Oracle            |

+-------------------+ 1 行数据（0,00 秒）

```

# 19.9 Removing Elements from JSON

## Problem

You want to remove elements from a JSON document.

## Solution

Use the function `JSON_REMOVE`.

## Discussion

The function `JSON_REMOVE` removes specified element from JSON.

For example, to remove unpublished books from the `book_authors` table use following code.

```

UPDATE book_authors SET author = JSON_REMOVE(author, '$.books[0]')

WHERE author->>'$.name' = 'Alkin';

UPDATE book_authors SET author = JSON_REMOVE(author, '$.books[last]')

WHERE author->>'$.name' = 'Sveta';

```

# 19.10 Merging Two or More JSON Documents into One

## Problem

You want to combine two or more JSON documents into one.

## Solution

Use functions family `JSON_MERGE_*`.

## Discussion

Two functions: `JSON_MERGE_PATCH` and `JSON_MERGE_PRESERVE` are available for combining multiple JSON documents into one. `JSON_MERGE_PATCH` removes duplicates when merging two documents while `JSON_MERGE_PRESERVE` keeps them. Both functions take two or more arguments that should be valid JSON text.

For examples in this recipe we will store values of the `author` column in the table `book_authors` into user-defined variables: one for each author. Additionally we will store arrays of books for Sveta in a variable `sveta_books`.

```

SELECT author INTO @paul FROM book_authors WHERE author->>'$.name'='Paul';

SELECT author INTO @sveta FROM book_authors WHERE author->>'$.name'='Sveta';

SELECT author INTO @alkin FROM book_authors WHERE author->>'$.name'='Alkin';

SELECT author->>'$.books' INTO @sveta_books FROM book_authors

WHERE author->>'$.name'='Sveta';

```

`JSON_MERGE_PRESERVE` combines documents, provided by its arguments, into a single object. You can use this function to add new elements to your objects or arrays. Thus, to add array of countries where the author lived you can just provide an object, containing such an array as an argument.

```

mysql> `SELECT` `JSON_PRETTY``(``JSON_MERGE_PRESERVE``(``@``sveta``,`

    -> `'{"places lived": ["俄罗斯", "土耳其"]}'``)``)``\``G`

*************************** 1. 行 ***************************

JSON_PRETTY(JSON_MERGE_PRESERVE(@sveta, '{"places lived": ["俄罗斯", "土耳其"]}')): {

"id": 3,

"name": "Sveta",

"work": "Percona",

"books": [

    "MySQL Troubleshooting",

    "MySQL Cookbook"

],

"lastname": "Smirnova",

"places lived": [

    "俄罗斯",

    "土耳其"

]

}

1 行数据（0,00 秒）

```

To add a new book into the array `books` pass it as a part of the `books` array in the object as a second argument.

```

mysql> `SELECT` `JSON_PRETTY``(``JSON_MERGE_PRESERVE``(``@``sveta``,`

    -> `'{"books": ["MySQL Performance Schema in Action"]}'``)``)``\``G`

*************************** 1. 行 ***************************

JSON_PRETTY(JSON_MERGE_PRESERVE(@sveta, ↩

'{"books": ["MySQL Performance Schema in Action"]}')): {

"id": 3，

"name": "Sveta",

"work": "Percona",

"books": [

    "MySQL Troubleshooting",

    "MySQL Cookbook",

    "MySQL Performance Schema in Action"

],

"lastname": "Smirnova"

}

1 row in set (0,00 sec)

```

Content of the array `books` in the second argument will be added to the end of the array with the same name in the first argument.

If two objects have scalar values with the same key they will be merged into an array.

```

mysql> `SELECT` `JSON_PRETTY``(``JSON_MERGE_PRESERVE``(``@``paul``,` `@``sveta``,` `@``alkin``)``)` `AS` `authors``\``G`

*************************** 1. row ***************************

authors: {

"id": [

    1,

    3,

    2

],

"name": [

    "Paul",

    "Sveta",

    "Alkin"

],

"work": [

    "Oracle",

    "Percona",

    "PlanetScale"

],

"books": [

    "Software Portability with imake: Practical Software Engineering",

    "Mysql: The Definitive Guide to Using, Programming, ↩

    and Administering Mysql 4 (Developer's Library)",

    "MySQL Certification Study Guide",

    "MySQL (OTHER NEW RIDERS)",

    "MySQL Cookbook",

    "MySQL 5.0 Certification Study Guide",

    "Using csh & tcsh: Type Less, Accomplish More (Nutshell Handbooks)",

    "MySQL (Developer's Library)",

    "MySQL Troubleshooting",

    "MySQL Cookbook",

    "MySQL Cookbook"

],

"lastname": [

    "DuBois",

    "Smirnova",

    "Tezuysal"

]

}

1 row in set (0,00 sec)

```

###### Note

Note that function `JSON_MERGE_PRESERVE` does not try to handle duplicates anyhow, so book title `MySQL Cookbook` repeats in the resulting array three times.

Function `JSON_MERGE_PATCH`, instead, removes duplicates in favor of its latest argument. Same combination of merging three authors will just return the one, specified as the last argument.

```

mysql> `SELECT` `JSON_PRETTY``(``JSON_MERGE_PATCH``(``@``paul``,` `@``sveta``,` `@``alkin``)``)` `AS` `authors``\``G`

*************************** 1. row ***************************

authors: {

"id": 2,

"name": "Alkin",

"work": "PlanetScale",

"books": [

    "MySQL Cookbook"

],

"lastname": "Tezuysal"

}

1 row in set (0,00 sec)

```

This feature could be used to remove unneeded elements from JSON. For example, if we decide that it does not matter in which company the author works, we can remove `work` element by passing it as an object with value `null`:

```

mysql> `SELECT` `JSON_PRETTY``(``JSON_MERGE_PATCH``(``@``sveta``,` `'{"work": null}'``)``)``\``G`

*************************** 1. row ***************************

JSON_PRETTY(JSON_MERGE_PATCH(@sveta, '{"work": null}')): {

"id": 3,

"name": "Sveta",

"books": [

    "MySQL Troubleshooting",

    "MySQL Cookbook"

],

"lastname": "Smirnova"

}

1 row in set (0,00 sec)

```

When the latest document of the function is not an object `JSON_MERGE_PRESERVE` will add it as a latest element of an array. For example, to add a new book to the array of books by Sveta you may use following code:

```

mysql> `SELECT` `JSON_PRETTY``(``JSON_MERGE_PRESERVE``(``@``sveta_books``,`

    -> `'"MySQL Performance Schema in Action"'``)``)` `AS` `'Books by Sveta'``\``G`

*************************** 1. row ***************************

Books by Sveta: [

"MySQL Troubleshooting",

"MySQL Cookbook",

"MySQL Performance Schema in Action"

]

1 row in set (0,00 sec)

```

`JSON_MERGE_PATCH`, instead, will replace elements in the original document with the new one:

```

mysql> `SELECT` `JSON_PRETTY``(``JSON_MERGE_PATCH``(``@``sveta_books``,`

    -> `'"MySQL Performance Schema in Action"'``)``)` `AS` `'Books by Sveta'``;`

+--------------------------------------+ | Books by Sveta                       |

+--------------------------------------+ | "MySQL Performance Schema in Action" |

+--------------------------------------+ 1 row in set (0,00 sec)

```

# 19.11 Creating JSON from Relational Data

## Problem

You have relational data and want to create JSON from it.

## Solution

Use the function `JSON_OBJECT`, the function `JSON_ARRAY` and their aggregate variants `JSON_OBJECTAGG`, `JSON_ARRAYAGG`.

## Discussion

It could be useful to create JSON out of relational data. MySQL provides the function `JSON_OBJECT` that combines pairs of values into JSON object.

```

mysql> `SELECT` `JSON_PRETTY``(`

    -> `JSON_OBJECT``(``"string"``,` `"Some String"``,` `"number"``,` `42``,` `"null"``,` `NULL``)``)` `AS` `my_object``\``G`

*************************** 1. row ***************************

my_object: {

"null": null,

"number": 42,

"string": "Some String"

}

1 row in set (0,00 sec)

```

The function `JSON_ARRAY` creates a JSON array from its arguments.

```

mysql> `SELECT` `JSON_PRETTY``(``JSON_ARRAY``(``"one"``,` `"two"``,` `"three"``,` `4``,` `5``)``)` `AS` `my_array``\``G`

*************************** 1. row ***************************

my_array: [

"one",

"two",

"three",

4,

5

]

1 row in set (0,00 sec)

```

You can combine both functions, make nesting objects and arrays.

```

mysql> `选择` `JSON_PRETTY``(``JSON_OBJECT``(``"示例"``,` `"嵌套对象和数组"``,`

    -> `"人类"``,` `JSON_OBJECT``(``"名称"``,` `"斯韦塔"``,` `"姓"``,` `"斯米尔诺娃"``)``,`

    -> `"数字"``,` `JSON_ARRAY``(``"一个"``,` `"两"``,` `"三"``)``)``)` `AS` `my_object``\``G`

*************************** 1. row ***************************

my_object: {

"人类": {

    "名称": "斯韦塔",

    "姓": "斯米尔诺娃"

},

"示例": "嵌套对象和数组",

"数字": [

    "一个",

    "两",

    "三"

]

}

1 行设置（0,00 秒）

```

The functions `JSON_OBJECTAGG` and `JSON_ARRAYAGG` are aggregate versions of `JSON_OBJECT` and `JSON_ARRAY` that allow you to create JSON objects and arrays out of data, returned by `GROUP BY` queries.

The database `cookbook` has a table `movies_actors` that contains a list of movies and actors that were starred in them. The table has few rows for the single movie and few others for the single actor.

If you want to have a JSON object that will list a movie and all actors who starred in that movie in as an array, combine the functions `JSON_OBJECT` and `JSON_ARRAYAGG`.

```

mysql> `选择` `JSON_PRETTY``(``JSON_OBJECT``(``'电影'``,` `电影``,`

    -> `'主演'``,` `JSON_ARRAYAGG``(``演员``)``)``)` `AS` `主演`

    -> `从` `电影演员` `组` `BY` `电影``\``G`

*************************** 1. row ***************************

starred: {

"电影": "天国王朝",

"主演": [

    "连姆·尼森",

    "奥兰多·布鲁姆"

]

}

*************************** 2. row ***************************

starred: {

"电影": "红色",

"主演": [

    "海伦·米伦",

    "布鲁斯·威利斯"

]

}

*************************** 3. row ***************************

starred: {

"电影": "指环王",

"主演": [

    "伊恩·麦克莱恩",

    "伊恩·霍尔姆",

    "奥兰多·布鲁姆",

    "伊莱贾·伍德"

]

}

*************************** 4. row ***************************

starred: {

"电影": "第五元素",

"主演": [

    "布鲁斯·威利斯",

    "加里·奥德曼",

    "伊恩·霍尔姆"

]

}

*************************** 5. row ***************************

starred: {

"电影": "幽灵危机",

"主演": [

    "伊万·麦克格雷格",

    "连姆·尼森"

]

}

*************************** 6. row ***************************

starred: {

"电影": "未知",

"主演": [

    "黛安·克鲁格",

    "连姆·尼森"

]

}

6 行设置（0,00 秒）

```

Function `JSON_OBJECTAGG` can take table values in one column as member names and values in another column as their arguments.

```

mysql> `选择` `JSON_PRETTY``(``JSON_OBJECTAGG``(``名称``,` `网站``)``)` `AS` `网站`

    -> `从` `书籍供应商``\``G`

*************************** 1. row ***************************

网站：{

"亚马逊": "www.amazon.com",

"巴恩斯与贵族": "www.barnesandnoble.com",

"O'Reilly Media": "www.oreilly.com"

}

1 行设置（0,00 秒）

```

# 19.12 Converting JSON into Relational Format

## Problem

You have JSON data and want to work with it in the same way as you do with relational structure.

## Solution

Use function `JSON_TABLE`.

## Discussion

In the previous recipe we converted relational data into JSON. You may need to do the opposite: convert JSON into relational format. In this case, the function `JSON_TABLE` will help.

The function `JSON_TABLE` takes a JSON document and a list of columns with paths as its arguments. It returns a table as a result.

For example, for a document

```

{

"空": null,

"号码": 42,

"字符串": "一些字符串"

}

```

`JSON_TABLE` can be called as:

```

mysql> `选择` `*`  ![1](img/1.png)

    -> `从` `JSON_TABLE``(` ![2](img/2.png)

    -> `'{"空": null, "号码": 42, "字符串": "一些字符串"}'``,`  ![3](img/3.png)

    -> `'$'`  ![4](img/4.png)

    -> `COLUMNS``(`  ![5](img/5.png)

    -> `number` `INT` `PATH` `'$.number'``,`  ![6](img/6.png)

    -> `字符串` `VARCHAR``(``255``)` `PATH` `'$.string'` `错误` `ON` `错误` ![7](img/7.png)

    -> `)``)` `AS` `jt``;` ![8](img/8.png)

+--------+-------------+ | number | string      |

+--------+-------------+ |     42 | 一些字符串 |

+--------+-------------+ 1 行设置（0,00 秒）

```

![1](img/#co_nch-json-json-table_select_co)

Start the query by selecting everything from the resulting table.

![2](img/#co_nch-json-json-table_from_co)

The function `JSON_TABLE` can be used only in the `FROM` clause.

![3](img/#co_nch-json-json-table_json_co)

The first argument to the function is a JSON document. In this example, the function takes a string. If you want to pass a column name in another table, you need to specify this table prior the `JSON_TABLE` call:

```

选择* FROM mytable, JSON_TABLE(mytable.mycolumn ...

```

![4](img/#co_nch-json-json-table_path_co)

A path that will be used as a document root. In this example we are using the whole document but you can simplify expressions for the columns if you specify the path to the part of the JSON document here.

![5](img/#co_nch-json-json-table_columns_co)

Definition of columns.

![6](img/#co_nch-json-json-table_col_co)

Column `number` has type `INT` and default error handling: the column set to `NULL` in case an error happens. We use JSON path `$.number` to set a value for this column.

![7](img/#co_nch-json-json-table_error_co)

For the column `string` we decided to raise an error, therefore we used clause `ERROR ON ERROR`.

![8](img/#co_nch-json-json-table_alias_co)

Any function in the `FROM` clause should have an alias, so we used `jt` as an alias.

To call the function `JSON_TABLE` on an existing table add it to the query prior calling the function. Practically, perform a `CROSS JOIN` of two tables. The `COLUMNS` clause also supports nested paths, so you can expand arrays into multiple rows.

The column `author` in the table `book_authors` contains list of books in the array `books`. To expand each row into the own row use clause `NESTED PATH`.

```

mysql> `选择` `jt``.``*` `从` `书籍作者` `ba``,`  ![1](img/1.png)

    -> `JSON_TABLE``(``ba``.``作者``,`

    -> `'$'` `COLUMNS` `(`

    -> `名称` `VARCHAR``(``255``)` `PATH` `'$.name'``,`

    -> `lastname` `VARCHAR``(``255``)` `PATH` `'$.lastname'``,`

    -> `NESTED` `PATH` `'$.books[*]'` `COLUMNS` `(` ![2](img/2.png)

    -> `book` `VARCHAR``(``255``)` `PATH` `'$'` `)` ![3](img/3.png)

    -> `)``)` `AS` `jt``;`

+-------+----------+-----------------------------------------------------------------+ | 名称  | 姓氏 | 书籍                                                            |

+-------+----------+-----------------------------------------------------------------+ | 保罗  | 杜博伊斯   | 使用 imake 进行软件可移植性: 实用软件工程 |

| 保罗  | 杜博伊斯   | Mysql: 详细使用指南, 编程, ↩

                    和管理 Mysql 4 (开发者图书馆)                 |

| 保罗  | 杜博伊斯   | MySQL 认证学习指南                                 |
| --- | --- | --- |
| 保罗  | 杜博伊斯   | MySQL (OTHER NEW RIDERS)                                        |
| 保罗  | 杜博伊斯   | MySQL Cookbook                                                  |
| 保罗  | 杜博伊斯   | MySQL 5.0 认证学习指南                             |

| 保罗  | 杜博伊斯   | 使用 csh & tcsh: 更少打字, 更多成就 ↩

                    (Nutshell Handbooks)                                            |

| 保罗  | 杜博伊斯   | MySQL (Developer's Library)                                     |
| --- | --- | --- |
| 阿尔金 | 特祖萨尔 | MySQL Cookbook                                                  |
| 斯维塔 | 斯米尔诺娃 | MySQL 故障排除                                           |
| 斯维塔 | 斯米尔诺娃 | MySQL Cookbook                                                  |

+-------+----------+-----------------------------------------------------------------+ 11 行结果 (0.01 秒)

```

![1](img/#co_nch-json-json-table_exttable_co)

To use a column in the existing table put the table name before the `JSON_TABLE` call.

![2](img/#co_nch-json-json-table_nested_co)

The clause `NESTED PATH` expands the following path pattern into several columns. In our case the path is `$.books[*]` that points to the each element of the array `books`.

![3](img/#co_nch-json-json-table_nestedcol_co)

Define the nested column as any other column. Note that `PATH` should be relative to the `NESTED PATH`.

## See Also

For additional information about function `JSON_TABLE`, see [JSON Table Functions in the MySQL User Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/json-table-functions.html).

# 19.13 Investigating JSON

## Problem

You want to know details about your JSON data structure, such as how deep the value is, how many children a particular element has, and so on.

## Solution

Use JSON attribute functions.

## Discussion

The function `JSON_LENGTH` returns a number of elements in the JSON document or a path, if specified. For scalars it is always 1, for objects and arrays this is the number of elements. You can use this function to perform such tasks as calculating number of books, written by a particular author:

```

mysql> `SELECT` `CONCAT``(``author``-``>``>``'$.name'``,` `' '``,` `author``-``>``>``'$.lastname'``)` `AS` `'author'``,`

    -> `JSON_LENGTH``(``author``-``>``>``'$.books'``)` `AS` `'书籍数量'` `FROM` `book_authors``;`

+----------------+-----------------+ | 作者         | 书籍数量 |

+----------------+-----------------+ | 保罗 杜博伊斯    |               8 |

| 阿尔金·特祖萨尔 |               1 |
| --- | --- |
| 斯维塔 斯米尔诺娃 |               2 |

+----------------+-----------------+ 3 行结果 (0.01 秒)

```

Function `JSON_DEPTH` returns maximum depth of the JSON document. It returns one for a scalar, empty object or empty array. For objects and arrays with inner elements it counts all nested levels. For the `author` column in the `book_authors` table it returns three:

```

mysql> `SELECT` `JSON_DEPTH``(``author``)` `FROM` `book_authors` `WHERE` `author``-``>``>``'$.name'` `=` `'斯维塔'``;`

+--------------------+ | JSON_DEPTH(author) |

+--------------------+ |                  3 |

+--------------------+ 1 行结果 (0.00 秒)

```

To understand why so let’s examine example value in detail:

```

{ ![1](img/1.png)

"id": 3,

"name": "斯维塔",

"work": "Percona", ![2](img/2.png)

"books": 

    "MySQL 故障排除", ![3

    "MySQL Cookbook"

],

"lastname": "斯米尔诺娃"

}

```

![1](img/#co_nch-json-json-attributes_obj_co)

Level one: the object that contains all the elements.

![2](img/#co_nch-json-json-attributes_el_co)

Level two: object element.

![3](img/#co_nch-json-json-attributes_array_el_co)

Level three: element of the nested array.

Function `JSON_DEPTH` is useful when you need to understand how complex your JSON data is.

Function `JSON_STORAGE_SIZE` returns the number of bytes that the JSON data takes. It is useful to plan storage use for your data.

```

mysql> `SELECT` `JSON_STORAGE_SIZE``(``author``)` `FROM` `book_authors``;`

+---------------------------+ | JSON_STORAGE_SIZE(author) |

+---------------------------+ |                       475 |

|                       144 |
| --- |
|                       171 |

+---------------------------+ 3 行结果 (0.00 秒)

```

The function `JSON_TYPE` returns the type of the JSON element. Thus, for the `author` column in the table `book_authors` the types are as shown:

```

mysql> `SELECT` `JSON_TYPE``(``author``)``,` `JSON_TYPE``(``author``-``>``'$.id'``)``,`

    -> `JSON_TYPE``(``author``-``>``'$.name'``)``,` `JSON_TYPE``(``author``-``>``'$.books'``)`

    -> `FROM` `book_authors` `WHERE` `author``-``>``>``'$.name'` `=` `'Sveta'``\``G`

*************************** 1. row ***************************

        JSON_TYPE(author): OBJECT

JSON_TYPE(author->'$.id'): INTEGER

JSON_TYPE(author->'$.name'): STRING

JSON_TYPE(author->'$.books'): ARRAY

1 row in set (0,00 sec)

```

###### Warning

Note that we used operator `->` instead of `->>` to preserve quotes in scalar values.

# 19.14 Working with JSON in MySQL as a Document Store

## Problem

You want to work with JSON in MySQL in the same way as NoSQL databases do.

## Solution

Use X DevAPI. The following clients and connectors support X DevAPI and can work with JSON as a Document Store.

*   MySQL Shell in JavaScript and Python mode
*   Connector/C++
*   Connector/J
*   Connector/Node.js
*   Connector/NET
*   Connector/Python

## Discussion

We will use MySQL Shell for examples in this recipe. We assume that you are connected to the MySQL server, thus have the default objects available. See Recipe 2.1 for instructions on how to connect to MySQL Server via MySQL Shell.

MySQL Document Store is a collection, stored in a table, defined as

```

CREATE TABLE `MyCollection` (

`doc` json DEFAULT NULL,

`_id` varbinary(32) GENERATED ALWAYS AS (json_unquote(

        json_extract(`doc`,_utf8mb4'$._id'))) STORED NOT NULL,

`_json_schema` json GENERATED ALWAYS AS (_utf8mb4'{"type":"object"}') VIRTUAL,

PRIMARY KEY (`_id`),

CONSTRAINT `$val_strict_2190F99D7C6BE98E2C1EFE4E110B46A3D43C9751`

CHECK (json_schema_valid(`_json_schema`,`doc`)) /*!80016 NOT ENFORCED */

) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

```

where `doc` is a JSON column, storing the document. `_id` is unique identifier, generated by extracting value of the member `_id` and optional `_json_schema` is a schema which you can enforce when creating collection. See Recipe 2.9 for the details and example.

X DevAPI will create such a table when you call `createCollection` method.

```

MySQL  cookbook  JS > `session.getDefaultSchema().createCollection('MyCollection')`

<Collection:MyCollection>

```

###### Tip

We use syntax which MySQL Shell in JavaScript mode understands for examples in this recipe. Syntax for different languages slightly differs. Refer to your implementation documentation for details.

Once the collection is created you can insert documents into it, update, remove and search them.

It is handy to store collection object in a variable.

```

MySQL  cookbook  JS > `MyCollection = session.getDefaultSchema().

                -> `getCollection('MyCollection')`

<Collection:MyCollection>

```

`Collection` class in X DevAPI supports four basic CRUD (Create, Read, Update, Delete) operations:

*   `add`
*   `find`
*   `modify`
*   `remove`

We already showed them in action when we discussed MySQL Shell in Recipe 2.9 and Recipe 2.10. In this recipe we will cover details, missed there.

### Adding documents to the collection

To add documents into the collection, use a method `add` that accepts either a JSON object or an array of JSON objects, or a `mysqlx.expr` as an argument. The following code snippet demonstrates all three flavours of the syntax.

```

MySQL  cookbook  JS > `MyCollection.add({"document": "one"}).`

                    -> `add([{"document": "two"}, {"document": "three"}]).`

                    -> `add(mysqlx.expr('{"document": "four"}'))`

                    ->

Query OK, 4 items affected (0.0083 sec)

Records: 4  Duplicates: 0  Warnings: 0

```

### Searching for documents

To search for documents, use the method `find`. If called without arguments it will return a list of all documents in the collection.

```

MySQL  cookbook  JS > `MyCollection.find()`

{

    "_id": "000060d5ab750000000000000012",

    "document": "one"

}

{

    "_id": "000060d5ab750000000000000013",

    "document": "two"

}

{

    "_id": "000060d5ab750000000000000014",

    "document": "three"

}

{

    "_id": "000060d5ab750000000000000015",

    "document": "four"

}

4 documents in set (0.0007 sec)

```

Each of the documents contains an automatically generated `_id` that is also a primary key for the collection.

The method `find` narrows a result set by using search conditions, limiting the number of the documents, and grouping, sorting, and modifying the resulting values. These are basic methods, available to modify result of any SQL `SELECT` operation. However, it is not possible to join two collections like you can do with SQL tables.

To search for a particular document pass a condition as an argument of the method `find`. You can use the operator `LIKE` and others to perform creative comparisons.

```

MySQL  cookbook  JS > `MyCollection.find("document LIKE 't%'")`

{

    "_id": "000060d5ab750000000000000013",

    "document": "two"

}

{

    "_id": "000060d5ab750000000000000014",

    "document": "three"

}

2 documents in set (0.0009 sec)

```

To modify the result, pass the expression to the method `fields`:

```

MySQL  cookbook  JS > `MyCollection.find("document LIKE 't%'").`

                    -> `fields(mysqlx.expr('{"Document": upper(document)}'))`

                    ->

{

    "Document": "TWO"

}

{

    "Document": "THREE"

}

2 documents in set (0.0009 sec)

```

To group documents, use the method `groupBy` and narrow the result with the method `having`. To illustrate how they work we will use collection `CollectionLimbs`.

```

MySQL  cookbook  JS > `limbs = session.getDefaultSchema().getCollection('CollectionLimbs')`

<Collection:CollectionLimbs>

MySQL  cookbook  JS > `limbs.find().fields('arms', 'COUNT(thing)').groupBy('arms')`

{

    "arms": 2,

    "COUNT(thing)": 3

}

{

    "arms": 0,

    "COUNT(thing)": 5

}

{

    "arms": 10,

    "COUNT(thing)": 1

}

{

    "arms": 1,

    "COUNT(thing)": 1

}

{

    "arms": null,

    "COUNT(thing)": 1

}

5 documents in set (0.0010 sec)

```

The code above prints the number of things with a specific number of arms. To limit this list to only things that have both arms and legs, we can use the method `having`:

```

MySQL  cookbook  JS > `limbs.find().fields('arms', 'COUNT(thing)').`

                -> `groupBy('arms').having('MIN(legs) > 0')`

{

    "arms": 2,

    "COUNT(thing)": 3

}

1 document in set (0.0006 sec)

```

To print the three things with the highest number of legs, use the method `sort` with the keyword `DESC` and `limit`:

```

MySQL  cookbook  JS > `limbs.find().sort('legs DESC').limit(3)`

{

    "_id": "000060d5ab75000000000000001a",

    "arms": 0,

    "legs": 99,

    "thing": "centipede"

}

{

    "_id": "000060d5ab750000000000000017",

    "arms": 0,

    "legs": 6,

    "thing": "insect"

}

{

    "_id": "000060d5ab75000000000000001b",

    "arms": 0,

    "legs": 4,

    "thing": "table"

}

3 documents in set (0.0006 sec)

```

You may also bind values if you pass the parameter name after a colon sign in the method `find` and pass values in the method `bind`. You may bind as many arguments as you want.

```

MySQL  cookbook  JS > `limbs.find('legs = :legs').bind('legs', 4)`

{

    "_id": "000060d5ab75000000000000001b",

    "arms": 0,

    "legs": 4,

    "thing": "table"

}

{

    "_id": "000060d5ab75000000000000001c",

    "arms": 2,

    "legs": 4,

    "thing": "armchair"

}

2 documents in set (0.0008 sec)

MySQL  cookbook  JS > `limbs.find('legs = :legs and arms = :arms').`

                    -> `bind('legs', 4).bind('arms', 2)`

{

    "_id": "000060d5ab75000000000000001c",

    "arms": 2,

    "legs": 4,

    "thing": "armchair"

}

1 document in set (0.0005 sec)

```

### Modifying documents

To modify documents in the collection, use the method `modify`. It accepts search condition and allows you to bind parameters similarly to the method `find`. To modify found elements use the methods `set` and `unset` to set or unset values of the object member. Use the methods `arrayInsert`, `arrayAppend` and `arrayDelete` to modify arrays, and use the method `patch` to merge JSON documents.

For the illustrations we will use the collection `MyCollection`.

```

MySQL  cookbook  JS > `MyCollection``.``find``(``'document = "one"'``)` ![1](img/1.png)

{

    "_id": "000060d5ab750000000000000012",

    "document": "one"

}

1 document in set (0.0005 sec)

MySQL  cookbook  JS > `MyCollection``.``modify``(``'document = "one"'``)``.`

                    -> `set``(``'array'``,` `[``2``,` `3``,` `4``]``)`![2](img/2.png)

Query OK, 1 item affected (0.0054 sec)

Rows matched: 1  Changed: 1  Warnings: 0

MySQL  cookbook  JS > `MyCollection``.``find``(``'document = "one"'``)`

{

    "_id": "000060d5ab750000000000000012",

    "array": [

        2,

        3,

        4

    ],

    "document": "one"

}

1 document in set (0.0005 sec)

MySQL  cookbook  JS > `MyCollection``.``modify``(``'document = "one"'``)``.``arrayAppend``(``'array'``,` `5``)`![3](img/3.png)

Query OK, 1 item affected (0.0073 sec)

Rows matched: 1  Changed: 1  Warnings: 0

MySQL  cookbook  JS > `MyCollection``.``find``(``'document = "one"'``)`

{

    "_id": "000060d5ab750000000000000012",

    "array": [

        2,

        3,

        4,

        5

    ],

    "document": "one"

}

1 document in set (0.0007 sec)

MySQL  cookbook  JS > `MyCollection``.``modify``(``'document = "one"'``)``.`

                    -> `arrayInsert``(``'array[0]'``,` `1``)`![4](img/4.png)

Query OK, 1 item affected (0.0072 sec)

Rows matched: 1  Changed: 1  Warnings: 0

MySQL  cookbook  JS > `MyCollection``.``find``(``'document = "one"'``)`

{

    "_id": "000060d5ab750000000000000012",

    "array": [

        1,

        2,

        3,

        4,

        5

    ],

    "document": "one"

}

1 document in set (0.0008 sec)

MySQL  cookbook  JS > `MyCollection``.``modify``(``'document = "one"'``)``.``arrayDelete``(``'array[2]'``)`![5](img/5.png)

Query OK, 1 item affected (0.0059 sec)

Rows matched: 1  Changed: 1  Warnings: 0

MySQL  cookbook  JS > `MyCollection``.``find``(``'document = "one"'``)`

{

    "_id": "000060d5ab750000000000000012",

    "array": [

        1,

        2,

        4

        5

    ],

    "document": "one"

}

1 document in set (0.0009 sec)

MySQL  cookbook  JS > `MyCollection``.``modify``(``'document = "one"'``)``.``unset``(``'array'``)`![6](img/6.png)

Query OK, 1 item affected (0.0080 sec)

Rows matched: 1  Changed: 1  Warnings: 0

MySQL  cookbook  JS > `MyCollection``.``find``(``'document = "one"'``)`

{

    "_id": "000060d5ab750000000000000012",

    "document": "one"

}

1 document in set (0.0007 sec)

MySQL  cookbook  JS > `MyCollection``.``modify``(``'document = "one"'``)``.`

                    -> `patch``(``{``'number'``:` `42``,` `'array'``:` `[``1``,``2``,``3``]``}``)``.`

                    -> `patch``(``{``'array'``:` `[``4``,``5``]``}``)`![7](img/7.png)

Query OK, 1 item affected (0.0063 sec)

Rows matched: 1  Changed: 1  Warnings: 0

MySQL  cookbook  JS > `MyCollection``.``find``(``'document = "one"'``)`

{

    "_id": "000060d5ab750000000000000012",

    "array": [

        4,

        5

    ],

    "number": 42,

    "document": "one"

}

1 document in set (0.0007 sec)

```

![1](img/#co_nch-json-json-nosql-modify_init_co)

We will experiment with this document from the collection.

![2](img/#co_nch-json-json-nosql-modify_set_co)

The mehtod `set` adds or changes an element in the object.

![3](img/#co_nch-json-json-nosql-modify_arrayAppend_co)

The method `arrayAppend` adds new element to the end of the array.

![4](img/#co_nch-json-json-nosql-modify_arrayInsert_co)

For the method `arrayInsert` you can specify the position in the array where you want to add new element.

![5](img/#co_nch-json-json-nosql-modify_arrayDelete_co)

The method `arrayDelete` removes an element from the specified position.

![6](img/#co_nch-json-json-nosql-modify_unset_co)

The method `unset` removes an element from the object.

![7](img/#co_nch-json-json-nosql-modify_patch_co)

The method `patch` works similarly to the JSON function `JSON_MERGE_PATCH`. In our case it first added two elements: `number` and `array` to the original document, then replaced content of the element `array` with the content of the element with the same name in the object, passed as a parameter to the second invocation of the method `patch`.

### Removing documents and collections.

To remove documents use the method `remove`.

```

MyCollection.remove('document = :number').bind('number', 'one')

```

To drop a collection, use the method `dropCollection` in your API.

```

session.getSchema('cookbook').dropCollection('MyCollection')

```

## 查看也可以

关于 X DevAPI 的更多信息，请参见[X DevAPI 参考手册](https://dev.mysql.com/doc/dev/mysqlsh-api-javascript/8.0/group___x_dev_a_p_i.html)。
