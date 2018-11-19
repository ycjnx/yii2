查询构建器
=============

查询构建器是基于[数据存取对象DAO](db-dao.md)实现的，它以编程的方式构建SQL查询语句，与具体使用哪种DBMS（数据库管理系统）无关。与编写原生SQL语句相比，使用查询构建器书写的SQL代码具有更好的可读性和安全性。



使用查询构建器通常只需两步：

1、创建一个[yii\db\Query]对象，它包含了查询SQL语句的各个部分（例如`SELECT`、`FROM`）。
2、执行[yii\db\Query]的查询方法（例如`all()`）从数据库中获取数据。

使用查询构建器的示例代码如下：

```php
$rows = (new \yii\db\Query())
    ->select(['id', 'email'])
    ->from('user')
    ->where(['last_name' => 'Smith'])
    ->limit(10)
    ->all();
```

上例代码将生成和执行以下的SQL查询语句，参数`:last_name`绑定到了字符串`'Smith'`：


```sql
SELECT `id`, `email` 
FROM `user`
WHERE `last_name` = :last_name
LIMIT 10
```

> 信息：通常我们用的最多的是[[yii\db\Query]]，而不是[[yii\db\QueryBuiler]]，前者在执行查询方法时会自动调用后者。[[yii\db\QueryBuiler]]负责将DBMS无关的[[yii\db\Query]]对象转换成DBMS的SQL语句（例如引用不同的表名和列名）。





## 构建查询 <span id="building-queries"></span>

在构建一个[[yii\db\Query]]对象时，我们需要为SQL查询语句的每一部分调用各自的构建方法。这些方法名称与SQL语句中对应部分的关键字相似。例如，要定义SQL查询语句的`FROM`部分，我们可以调用[[yii\db\Query::from()|from()]]方法。


这些查询构建方法将返回查询对象本身，这样你就可以对多个部分通过链式操作统一调用了。

接下来我们将叙述一下每个查询构建方法的用法。


### [[yii\db\Query::select()|select()]] <span id="select"></span>

[[yii\db\Query::select()|select()]]方法用来定义SQL语句的`SELECT`部分。如下例所示，我们可以使用数组或字符串来定义将要查询的字段。从查询对象生成SQL语句时这些字段名称将被自动引用。
 


```php
$query->select(['id', 'email']);

// 等同于：

$query->select('id, email');
```

和编写原生SQL查询一样，查询的字段名可以包含表前缀，也可以为字段起个别名。
例如：

```php
$query->select(['user.id AS user_id', 'email']);

// 等同于：

$query->select('user.id AS user_id, email');
```

如果使用数组格式来指定字段，你可以使用数组的键值来表示字段的别名。
例如，上面的代码可以被重写为如下形式：

```php
$query->select(['user_id' => 'user.id', 'email']);
```

在构建查询语句时如果没有调用[[yii\db\Query::select()|select()]]方法，将查询`*`，也就是查询所有字段。


除了查询字段名，我们也可以查询数据库表达式。当要查询的数据库表达式包含逗号时，我们只能使用数组格式以避免自动引用名称时产生错误。例如：


```php
$query->select(["CONCAT(first_name, ' ', last_name) AS full_name", 'email']); 
```

与原生SQL语句的其他部分一样，在为查询字段编写数据库表达式时，我们可以使用[DBMS无关的引用格式](db-dao.md#quoting-table-and-column-names)的表名和列名。


Yii从2.0.1版本起，我们就可以编写select子查询语句了，最好将每个子查询也定义为[[yii\db\Query]]对象，例如：
 

```php
$subQuery = (new Query())->select('COUNT(*)')->from('user');

// SELECT `id`, (SELECT COUNT(*) FROM `user`) AS `count` FROM `post`
$query = (new Query())->select(['id', 'count' => $subQuery])->from('post');
```

如果要去除记录中的重复项，我们可以调用[[yii\db\Query::distinct()|distinct()]]，代码如下：

```php
// SELECT DISTINCT `user_id` ...
$query->select('user_id')->distinct();
```

我们可以调用[[yii\db\Query::addSelect()|addSelect()]]来添加查询字段，例如：

```php
$query->select(['id', 'username'])
    ->addSelect(['email']);
```


### [[yii\db\Query::from()|from()]] <span id="from"></span>

[[yii\db\Query::from()|from()]]方法用来定义SQL语句的`FROM`部分，例如：

```php
// SELECT * FROM `user`
$query->from('user');
```

我们可以使用字符串或数组来定义要查询的表。与编写原生SQL语句相似，表名可以带库前缀，也可以为表定义别名。例如：


```php
$query->from(['public.user u', 'public.post p']);

// 等同于：

$query->from('public.user u, public.post p');
```

如果是数组格式，我们可以使用键名来定义表的别名，示例如下：

```php
$query->from(['u' => 'public.user', 'p' => 'public.post']);
```

除了查询表名外，我们还可以从子语句中查询，子语句是通过[[yii\db\Query]]对象来定义的。
例如：

```php
$subQuery = (new Query())->select('id')->from('user')->where('status=1');

// SELECT * FROM (SELECT `id` FROM `user` WHERE status=1) u 
$query->from(['u' => $subQuery]);
```
#### 前缀

可以通过[[yii\db\Connection::$tablePrefix|tablePrefix]]定义默认的表前缀，完整指令请参见["数据存取对象DAO"章节的"引用表"部分](guide-db-dao.html#quoting-table-and-column-names)


### [[yii\db\Query::where()|where()]] <span id="where"></span>

[[yii\db\Query::where()|where()]]方法用来定义SQL语句的`WHERE`部分，我们有四种方法来定义`WHERE`条件：


- 字符串格式，例如：`'status=1'`
- 哈希格式，例如： `['status' => 1, 'type' => 2]`
- 操作符格式，例如：`['like', 'name', 'test']`
- 对象格式，例如：`new LikeCondition('name', 'LIKE', 'test')`

#### 字符串格式 <span id="string-format"></span>

当查询条件很简单或者需要使用DBMS的内置函数时可以使用字符串格式，它就象是在编写原生SQL语句。例如：


```php
$query->where('status=1');

//或者使用参数绑定来绑定变量值
$query->where('status=:status', [':status' => $status]);

//原生SQL语句，在date字段使用MySQL的YEAR()函数
$query->where('YEAR(somedate) = 2015');
```

如下例所示，不要在查询条件中直接引用变量，尤其是变量值来自于用户输入的情况下，它会将你的应用置于SQL注入攻击之下。
 

```php
//危险！不要这样做，除非你能确保$status是一个整数。
$query->where("status=$status");
```

当使用`参数绑定`时，我们可以调用[[yii\db\Query::params()|params()]]或[[yii\db\Query::addParams()|addParams()]]方法来单独定义参数：


```php
$query->where('status=:status')
    ->addParams([':status' => $status]);
```

与原生SQL语句的其他部分一样，以字符串格式编写条件语句时，我们可以使用[DBMS无关的引用格式](db-dao.md#quoting-table-and-column-names)的表名和字段名。


#### 哈希格式 <span id="hash-format"></span>

如果要定义的查询条件是`与逻辑`关联的多个子查询，且每个子查询都是简单的等式判断时，哈希格式将是最好的选择。哈希格式就是一个数组，键名是字段名，键值是要查询的值。
例如：


```php
// ...WHERE (`status` = 10) AND (`type` IS NULL) AND (`id` IN (4, 8, 15))
$query->where([
    'status' => 10,
    'type' => null,
    'id' => [4, 8, 15],
]);
```

如上所示，无论是空值(null)，还是数组，查询构建器都可以自动处理。

我们也可以使用哈希格式的子查询，示例如下：

```php
$userQuery = (new Query())->select('id')->from('user');

// ...WHERE `id` IN (SELECT `id` FROM `user`)
$query->where(['id' => $userQuery]);
```

使用哈希格式时，Yii自动将变量绑定到参数上，而不是象[字符串格式](#string-format)那样手动添加变量。需注意的是，Yii不会对字段名做限制，我们如果传递一个变量作为字段名，而这个变量是从用户端获取的未检测值，我们的应用将在SQL注入攻击下变得异常脆弱。从应用的安全考虑，我们可以使用一个白名单来过滤变量，或者干脆不要使用变量作为字段名。如果确实需要从用户那里获取字段名，请参考文章[过滤数据](output-data-widgets.md#filtering-data)。以下就是易受攻击的代码：






```php
// 易受攻击的代码:
$column = $request->get('column');
$value = $request->get('value');
$query->where([$column => $value]);
// $value是安全的， 但是$column名称不能保证安全！
```

#### 操作符格式<span id="operator-format"></span>

使用操作符格式我们能以可编程方式定义任何查询条件，它的书写格式如下：

```php
//[操作符,操作数1,操作数2,...]
[operator, operand1, operand2, ...]
```
每个操作数（operand）可以是字符串格式、哈希格式，或是嵌套的操作符格式；而操作符（operator）可以是下列操作符之一：


-`and`：所有的操作数将使用`AND`连接起来。例如：`['and', 'id=1', 'id=2']`将生产成`id=1 AND id=2`。如果其中一个操作数是数组，据此规则它也将被转为字符串。例如，`['and', 'type=1', ['or', 'id=1', 'id=2']]`将生成 `type=1 AND (id=1 OR id=2)`。此方法不会进行任何引用或转义。





- `or`：与`and`操作符相似，但它使用`OR`操作符来连接所有操作数。

- `not`：只需要操作数1一个参数，它将被包含在`NOT()`函数中。例如，`['not', 'id=1']`将生成`NOT (id=1)`语句。这个操作数也可以是一个数组，使用它来描述多个表达式。例如，`['not', ['status' => 'draft', 'name' => 'example']]`将生成`NOT ((status='draft') AND (name='example'))`语句。

- `between`：操作数1是字段名，操作数2和操作数3分别是字段所在范围的开始值和结束值。例如， `['between', 'id', 1, 10]`将生成`id BETWEEN 1 AND 10`语句。如果我们要构建值位于两个列之间的查询条件（如`11 BETWEEN min_id AND max_id`），可以使用[[yii\db\conditions\BetweenColumnsCondition|BetweenColumnsCondition]]，请参见[查询条件 – 对象格式](#object-format)章节来学习更多关于条件对象定义的内容。






- `not between`：与`BETWEEN`操作符相似，但它使用`NOT BETWEEN`操作符来生成查询条件。


- `in`：操作数1应是一个字段或数据库表达式，操作数2可以是一个数组或是一个`Query`对象。它将生成一个`IN`查询条件。如果操作数2是数组，它表示的是列或数据库表达式值的范围；如果操作数2是`Query`对象，将会生成一个了查询，由它来确定列或数据库表达式的取值范围。例如，`['in', 'id', [1, 2, 3]]` 将生成`id IN (1, 2, 3)`语句。此方法将正确引用字段名，并转义范围中的值。
`in`操作符也支持综合列，有时操作数1是多个字段的数组，操作数2是一个有多个数组元素的数组或表示字段范围的`Query`对象。







- `not in`：与`in`操作符相似，但它使用`NOT IN`操作符来生成查询条件。

- `like`：操作数1可以是字段或数据库表达式，操作数2是要查找的字符串或数组。例如，`['like', 'name', 'tester']`会生成`name LIKE '%tester%'`语句。当值范围是使用数组表示时，将会生成使用`AND`连接的多个`LIKE`语句，例如，`['like', 'name', ['test', 'sample']]`将生成
  `name LIKE '%test%' AND name LIKE '%sample%'`语句。
  你也可以提供第3个操作数来定义如何转义变量值中的特殊字符。
  此操作数是一个数组，它将特殊字符映射到相应的转义字符上。如果此操作数未定义，将使用默认的映射。
  你可以使用`false`或一个空数组来说明此值已被转义过或无需使用转义。注意，当使用转义映射（或第三个操作数未提供）时，
  此值将自动被一对百分号括起来。







 > 注意：当使用PostgreSQL时你也可以使用[`ilike`](http://www.postgresql.org/docs/8.3/static/functions-matching.html#FUNCTIONS-LIKE)来进行忽略大小写的查找。


- `or like`: 与`like`操作符相似，当操作数2是数组时，它使用`OR`连接`LIKE`子句以生成查询条件。


- `not like`: 与`like`操作符相似，但它使用`NOT LIKE`操作符来生成查询条件。


- `or not like`: 与`not like`操作符相似，但它使用`OR`连接`NOT LIKE`子句以生成查询条件。


- `exists`: 只需一个操作数，此操作数必须是[[yii\db\Query]]表示的子查询(sub-query)。
  它将构建一个`EXISTS (sub-query)`表达式

- `not exists`: 与`exists`操作符相似并`NOT EXISTS (sub-query)`表达式。

- `>`, `<=`, 或其他有效的两操作数的DB操作符：第一操作数必须是字段名，第二操作数是一个值，例如，`['>', 'age', 10]`将生成` will generate `age>10`语句。

使用操作符格式，Yii自动将变量值绑定到参数上，这样对比[字符串格式](#string-format)你无需手动添加参数。须注意的是，Yii不会转义列名，如果你传递一个变量作为字段名，应用将在SQL注入攻击下变得异常脆弱。为了保证应用的安全性，请通过白名单来过滤变量，或者干脆不要使用变量作为字段名。
如果你确实需要从用户处获取字段名时，请阅读[Filtering Data](output-data-widgets.md#filtering-data)指南文章。例如，以下代码是易受攻击的：






```php
//易受攻击的代码：
$column = $request->get('column');
$value = $request->get('value);
$query->where(['=', $column, $value]);
//$value是安全的，$column名称是未编码的
```

#### 对象格式 <span id="object-format"></span>

从 2.0.14 版本起对象格式有效，它是定义查询条件最强大、最复杂的方式。如果你想要在查询构建器上构建自己的抽象对象，或者实现更复杂的查询条件时，你可以使用对象格式。



条件类的实例是不可改变的，它们用于存储条件数据并为条件构建器提供获取器。条件构建器是一个类，它为条件数据转换为SQL表达式提供了转换逻辑。



在构建成原生SQL语句之前，Yii内部会将此前描述的条件格式隐式转为对象格式，所以在一个条件格式中可以将各种条件格式进行组合使用：


```php
$query->andWhere(new OrCondition([
    new InCondition('type', 'in', $types),
    ['like', 'name', '%good%'],
    'disabled=false'
]))
```

将操作符格式转为对象格式是根据[[yii\db\QueryBuilder::conditionClasses|QueryBuilder::conditionClasses]]属性，此属性定义了操作符名称与实现类名称之间的映射：



- `AND`, `OR` -> `yii\db\conditions\ConjunctionCondition`
- `NOT` -> `yii\db\conditions\NotCondition`
- `IN`, `NOT IN` -> `yii\db\conditions\InCondition`
- `BETWEEN`, `NOT BETWEEN` -> `yii\db\conditions\BetweenCondition`

诸如此类。

使用对象格式可以创建自定义条件，也可以改变默认的构建方式。
请参见[添加自定义条件和表达式](#adding-custom-conditions-and-expressions) 章节以了解更多。


#### 追加查询条件 <span id="appending-conditions"></span>

你可以使用[[yii\db\Query::andWhere()|andWhere()]]或[[yii\db\Query::orWhere()|orWhere()]]为已存在的查询条件追加其他查询条件，你可以多次调用以追加多个查询条件，例如：



```php
$status = 10;
$search = 'yii';

$query->where(['status' => $status]);

if (!empty($search)) {
    $query->andWhere(['like', 'title', $search]);
}
```

如果`$search`非空，将生成以下`WHERE`查询条件语句：

```sql
WHERE (`status` = 10) AND (`title` LIKE '%yii%')
```


#### 过滤查询条件 <span id="filter-conditions"></span>

当基于用户输入构建`WHERE`查询条件时，通常我们会忽略为空的输入值。例如，一个搜索表单可以根据用户名和电子邮箱进行搜索，当用户在用户名/电子邮箱输入框中没有输入任何内容时，你想要忽略用户名/电子邮箱的查询条件，此时可以使用[[yii\db\Query::filterWhere()|filterWhere()]]方法：




```php
//$username和$email是用户输入的值
$query->filterWhere([
    'username' => $username,
    'email' => $email,
]);
```

[[yii\db\Query::filterWhere()|filterWhere()]]和[[yii\db\Query::where()|where()]]之间的唯一区别是前者将忽略为空的[哈希格式](#hash-format)查询条件。所以当`$email`为空而`$username`不为空时，上例生成的SQL语句将是`WHERE username=:username`。



> 信息：如果一个值是`null`、一个空数组、一个空字符串、或只有空格的字符串时，它将被视为空。

如同[[yii\db\Query::andWhere()|andWhere()]]和[[yii\db\Query::orWhere()|orWhere()]]一样，你可以使用[[yii\db\Query::andFilterWhere()|andFilterWhere()]]和[[yii\db\Query::orFilterWhere()|orFilterWhere()]]为已有的查询条件添加其他过滤查询条件。



另外，[[yii\db\Query::andFilterCompare()]]可以根据参数值智能确定操作符。


```php
$query->andFilterCompare('name', 'John Doe');
$query->andFilterCompare('rating', '>9');
$query->andFilterCompare('value', '<=100');
```

我们也可以显式定义操作符：

```php
$query->andFilterCompare('name', 'Doe', 'like');
```

从Yii 2.0.11版本起，对于`HAVING`查询条件有相似的方法：

- [[yii\db\Query::filterHaving()|filterHaving()]]
- [[yii\db\Query::andFilterHaving()|andFilterHaving()]]
- [[yii\db\Query::orFilterHaving()|orFilterHaving()]]

### [[yii\db\Query::orderBy()|orderBy()]] <span id="order-by"></span>

[[yii\db\Query::orderBy()|orderBy()]]方法用来定义SQL语句的`ORDER BY`部分，例如：

```php
// ... ORDER BY `id` ASC, `name` DESC
$query->orderBy([
    'id' => SORT_ASC,
    'name' => SORT_DESC,
]);
```
 
在上例代码中，数组的键名是字段名，键值是排序方式。
PHP常量`SORT_ASC`表示正序排列，`SORT_DESC`表示倒序排列。

`ORDER BY`如果只包含简单字段名，你可以使用字符串来定义它，就象编写原生SQL语句一样，例如：


```php
$query->orderBy('id ASC, name DESC');
```

> 注意：如果`ORDER BY`中有DB表达式，你应使用数组格式。

你可以调用[[yii\db\Query::addOrderBy()|addOrderBy()]]添加其他字段到`ORDER BY`部分，例如：


```php
$query->orderBy('id ASC')
    ->addOrderBy('name DESC');
```


### [[yii\db\Query::groupBy()|groupBy()]] <span id="group-by"></span>

[[yii\db\Query::groupBy()|groupBy()]]方法用来定义SQL语句的`GROUP BY`部分，例如：

```php
// ... GROUP BY `id`, `status`
$query->groupBy(['id', 'status']);
```

`GROUP BY`如果只包含简单字段名，可以使用字符串来定义它，就象编写原生SQL语句一样。
例如：

```php
$query->groupBy('id, status');
```

> 注意：如果`GROUP BY`中有DB表达式，你应使用数组格式。
 
你可以调用[[yii\db\Query::addGroupBy()|addGroupBy()]]来添加其他字段到`GROUP BY`部分。
例如：

```php
$query->groupBy(['id', 'status'])
    ->addGroupBy('age');
```


### [[yii\db\Query::having()|having()]] <span id="having"></span>

[[yii\db\Query::having()|having()]]方法用来定义SQL语句的`HAVING`部分。它有一个查询条件，此查询条件的定义方法与[where()](#where)中的一样，例如：


```php
// ... HAVING `status` = 1
$query->having(['status' => 1]);
```

如何定义一个查询条件请参考[where()](#where)的文档。

你可以调用[[yii\db\Query::andHaving()|andHaving()]]或[[yii\db\Query::orHaving()|orHaving()]]来追加查询条件到`HAVING`部分，例如：


```php
// ... HAVING (`status` = 1) AND (`age` > 30)
$query->having(['status' => 1])
    ->andHaving(['>', 'age', 30]);
```


### [[yii\db\Query::limit()|limit()]]和[[yii\db\Query::offset()|offset()]] <span id="limit-offset"></span>

[[yii\db\Query::limit()|limit()]]和[[yii\db\Query::offset()|offset()]]方法用来定义SQL语句的`LIMIT`和`OFFSET`部分，例如：


```php
// ... LIMIT 10 OFFSET 20
$query->limit(10)->offset(20);
```

如果你定义了一个无效的limit或offset（例如一个负数），它将被忽略。

> 信息：对于不支持`LIMIT` and `OFFSET`的DBMS（例如MSSQL），查询构建器将生成SQL语句来模拟`LIMIT`/`OFFSET`执行。



### [[yii\db\Query::join()|join()]] <span id="join"></span>

[[yii\db\Query::join()|join()]]方法用来定义SQL语句的`JOIN`部分。例如：

```php
// ... LEFT JOIN `post` ON `post`.`user_id` = `user`.`id`
$query->join('LEFT JOIN', 'post', 'post.user_id = user.id');
```

[[yii\db\Query::join()|join()]]方法有4个参数：

- `$type`: 连接类型，例如：`'INNER JOIN'`、`'LEFT JOIN'`。
- `$table`: 要连接的表名。
- `$on`: 可选参数，连接条件。例如，`ON`部分，关于定义查询条件请参考[where()](#where) 。注意，对字段名使用数组格式定义将**不会生效**，例如`['user.id' => 'comment.userId']`将会查询user的id等于字符串'comment.userId'的记录。此处我们应使用字符串格式来定义查询条件：`'user.id = comment.userId'`。
- `$params`: 可选参数，定义要绑定到连接条件中的参数。





你可以使用以下快捷方法来分别定义`INNER JOIN`、`LEFT JOIN`和`RIGHT JOIN`。

- [[yii\db\Query::innerJoin()|innerJoin()]]
- [[yii\db\Query::leftJoin()|leftJoin()]]
- [[yii\db\Query::rightJoin()|rightJoin()]]

例如：

```php
$query->leftJoin('post', 'post.user_id = user.id');
```

要连接多个表时，将为每个表调用一次join()方法。

除了连接表，你也可以连接子查询，此子查询需要使用[[yii\db\Query]]来定义。例如：


```php
$subQuery = (new \yii\db\Query())->from('post');
$query->leftJoin(['u' => $subQuery], 'u.id = author_id');
```

此时你需要将子查询放到数组中，并使用键名来定义一个别名。


### [[yii\db\Query::union()|union()]] <span id="union"></span>

[[yii\db\Query::union()|union()]]方法用来定义SQL语句的`UNION`部分。例如：

```php
$query1 = (new \yii\db\Query())
    ->select("id, category_id AS type, name") 
    ->from('post')
    ->limit(10);

$query2 = (new \yii\db\Query())
    ->select('id, type, name')
    ->from('user')
    ->limit(10);

$query1->union($query2);
```

你可以调用[[yii\db\Query::union()|union()]]多次以添加多个`UNION`部分。


## Query Methods <span id="query-methods"></span>

[[yii\db\Query]]为不同的查询目标提供了多种方法：

- [[yii\db\Query::all()|all()]]: 返回记录数组，每行记录都是键-值对表示的关联数组。
- [[yii\db\Query::one()|one()]]: 返回结果集的第一行记录数据。
- [[yii\db\Query::column()|column()]]: 返回结果集的第一列数据。
- [[yii\db\Query::scalar()|scalar()]]: 返回结果集中第一行记录第一列的标量值。
- [[yii\db\Query::exists()|exists()]]: 返回查询结果是否存在的一个值。
- [[yii\db\Query::count()|count()]]: 返回查询结果的数量。
- [[yii\db\Query::sum()|sum($q)]], [[yii\db\Query::average()|average($q)]],[[yii\db\Query::max()|max($q)]], [[yii\db\Query::min()|min($q)]]，这几个是聚集查询方法，`$q`是必填参数，它可以是字段名或DB表达式。



例如：

```php
// SELECT `id`, `email` FROM `user`
$rows = (new \yii\db\Query())
    ->select(['id', 'email'])
    ->from('user')
    ->all();
    
// SELECT * FROM `user` WHERE `username` LIKE `%test%`
$row = (new \yii\db\Query())
    ->from('user')
    ->where(['like', 'username', 'test'])
    ->one();
```

> 注意：[[yii\db\Query::one()|one()]]方法仅返回查询结果的第一条记录，在生成SQL语句时没有添加`LIMIT 1`。如果你知道查询将返回一条或数条数据记录（例如你使用数个主键值进行查询）这样是可以的。但如果查询结果可能会有很多数据记录时，你应显式调用`limit(1)`以提升性能。例如：`(new \yii\db\Query())->from('user')->limit(1)->one()`。





所有这些查询方法可以带一个可选参数`$db`，它是[[yii\db\Connection|DB connection]]，表示执行DB查询时将要使用的连接。如果省略此参数，`db` [应用组件](structure-application-components.md)将作为DB连接被使用。下面是使用[[yii\db\Query::count()|count()]] 查询方法的另外一个示例：



```php
//执行SQL语句: SELECT COUNT(*) FROM `user` WHERE `last_name`=:last_name
$count = (new \yii\db\Query())
    ->from('user')
    ->where(['last_name' => 'Smith'])
    ->count();
```

当你调用[[yii\db\Query]]的方法时，它实际将进行以下操作：

* 调用[[yii\db\QueryBuilder]]根据[[yii\db\Query]]当前的结构来生成SQL语句；
* 使用生成的SQL语句创建一个[[yii\db\Command]]对象；
* 调用[[yii\db\Command]]的一个查询方法（例如[[yii\db\Command::queryAll()|queryAll()]]）来执行SQL语句并获取数据。

有时，你可能想要检查或直接使用[[yii\db\Query]]对象生成的SQL语句，你可以通过以下代码实现：


```php
$command = (new \yii\db\Query())
    ->select(['id', 'email'])
    ->from('user')
    ->where(['last_name' => 'Smith'])
    ->limit(10)
    ->createCommand();
    
//显示SQL语句
echo $command->sql;
//显示要绑定的参数
print_r($command->params);

//返回查询结果的所有行
$rows = $command->queryAll();
```


### 给查询结果加索引 <span id="indexing-query-results"></span>

当你调用[[yii\db\Query::all()|all()]]时，它将返回所有行的一个数组，此数组以连续整数为索引。
有时你想要使用不同的索引方式，如使用一个特定字段或一个表达式的值来作为索引项。
在执行[[yii\db\Query::all()|all()]]前通过调用[[yii\db\Query::indexBy()|indexBy()]]即可实现。
例如：

```php
//返回[100 => ['id' => 100, 'username' => '...', ...], 101 => [...], 103 => [...], ...]
$query = (new \yii\db\Query())
    ->from('user')
    ->limit(10)
    ->indexBy('id')
    ->all();
```

要以表达式的值作为索引，传递一个匿名函数到[[yii\db\Query::indexBy()|indexBy()]]方法中即可：

```php
$query = (new \yii\db\Query())
    ->from('user')
    ->indexBy(function ($row) {
        return $row['id'] . $row['username'];
    })->all();
```

匿名函数中有一个参数`$row`，此参数表示当前行的数据；匿名函数将返回一个标量值，此值将作为当前行的索引值。


> 注意：作为对比的查询方法，如[[yii\db\Query::groupBy()|groupBy()]]或[[yii\db\Query::orderBy()|orderBy()]]，它们是已经转换成了SQL语句并作为查询语句的一部分，而indexBy()方法是从数据库中获取到数据后才进行处理的。也就是说只能使用查询中SELECT选定的列作为索引项，另外如果你选择列时使用了前缀，如`customer.id`，但在结果集中只包含了`id`，所以你只能使用没有前缀的`->indexBy('id')`来指定索引项。






### 批量查询 <span id="batch-query"></span>

如果要处理大量的数据，一些方法如[[yii\db\Query::all()]]可能就不适用了，因为它们需要加载整个查询结果集到客户端的内存中。Yii提供了批量查询来解决这个问题。服务器保存结果集，客户端每次使用游标来遍历结果集。

> 警告：MySQL实现批量查询有一定的限制条件和环境要求，请参见后文。

批量查询使用的示例：


```php
use yii\db\Query;

$query = (new Query())
    ->from('user')
    ->orderBy('id');

foreach ($query->batch() as $users) {
    //$users是来自于use表的一个100行或更少行的数组
}

//或者逐行遍历
foreach ($query->each() as $user) {
    //数据已经是服务器批处理过的100条记录，但$user表示user表中的一条数据记录。
}
```




[[yii\db\Query::batch()]]和[[yii\db\Query::each()]]返回一个[[yii\db\BatchQueryResult]]对象，它实现了`Iterator`接口，这样就可以在`foreach`循环中使用了。在第一个遍历中，SQL查询是针对数据库的，然后在其余迭代中批量获取数据。默认情况下，批处理大小是100，也就是说在每次批处理中获取100条记录。你可以向`batch()`或`each()`方法传入一个参数来修改批处理大小。





与[[yii\db\Query::all()]]相比，批量查询每次只载入100条记录到内在中。

如果你通过[[yii\db\Query::indexBy()]]指定某列来索引查询结果集，批量查询将保持正确的索引项。


例如：

```php
$query = (new \yii\db\Query())
    ->from('user')
    ->indexBy('username');

foreach ($query->batch() as $users) {
    //$users以"username"列作为索引
}

foreach ($query->each() as $username => $user) {
    // ...
}
```

#### MySQL中批量查询的限制条件 <span id="batch-query-mysql"></span>

MySQL是通过PDO驱动库实现批量查询的。默认情况下，MySQL查询是[`带缓存的`](http://php.net/manual/en/mysqlinfo.concepts.buffering.php)，这违背了使用光标获取数据的目的，因为它不阻止驱动程序将整个结果集加载到客户端的内存中。




> 注意：当使用`libmysqlclient`时（PHP5的典型配置），PHP的内存不会计算用于数据结果集的内存。看上去批量查询是正确运行的，实际上整个数据库都被加载到了客户端的内存中，而且这个使用量可能还会再增长。



要禁用缓存并减少客户端内存的需求量，PDO连接属性`PDO::MYSQL_ATTR_USE_BUFFERED_QUERY`必须设置为`false`。这样，直到整个数据集被处理完毕，通过相同连接是无法创建其他查询的。这样操作可能会阻止`ActiveRecord`执行表结构查询。如果这并不是一个问题（表结构已被缓存过），我们可以切换原始连接到非缓存模式，然后在批量查询完成后执行回滚操作。






```php
Yii::$app->db->pdo->setAttribute(\PDO::MYSQL_ATTR_USE_BUFFERED_QUERY, false);

//执行批量查询

Yii::$app->db->pdo->setAttribute(\PDO::MYSQL_ATTR_USE_BUFFERED_QUERY, true);
```

> 注意：对于MyISAM，在执行批量查询的过程中，表可能将被锁，将延迟或拒绝其他连接的写入操作。当使用非缓存查询时，尽量缩短游标打开的保持时间。



如果表结构没有被缓存，或在批量查询被处理过程中无需执行其他查询，你可以创建一个单独的到数据库的非缓存链接：


```php
$unbufferedDb = new \yii\db\Connection([
    'dsn' => Yii::$app->db->dsn,
    'username' => Yii::$app->db->username,
    'password' => Yii::$app->db->password,
    'charset' => Yii::$app->db->charset,
]);
$unbufferedDb->open();
$unbufferedDb->pdo->setAttribute(\PDO::MYSQL_ATTR_USE_BUFFERED_QUERY, false);
```

除了`PDO::MYSQL_ATTR_USE_BUFFERED_QUERY`是`false`之外，如果要确保`$unbufferedDb`拥有和原来缓存`$db`完全一样的属性，请参阅[实现`$db`的深度拷贝](https://github.com/yiisoft/yii2/issues/8420#issuecomment-301423833)，手动方法将它设置为false。




使用此连接正常创建查询即可，新连接用于运行批量查询，逐条或批量进行结果处理：


```php
//获取1000个批量处理数据
foreach ($query->batch(1000, $unbufferedDb) as $users) {
    // ...
}


//每次从服务器批量获取1000个数据，但是逐个遍历进行处理
foreach ($query->each(1000, $unbufferedDb) as $user) {
    // ...
}
```

当结果集已处理完毕不再需要连接时，可以关闭它：

```php
$unbufferedDb->close();
```

> 注意：非缓存查询在PHP端使用更少的缓存，但会增加MySQL服务器端的负载。建议您在生产实践中为大量的数据设计自己的代码。[例如，将数字键分段，使用非缓存的查询遍历](https://github.com/yiisoft/yii2/issues/8420#issuecomment-296109257).



## 添加自定义查询条件和表达式 <span id="adding-custom-conditions-and-expressions"></span>

在[查询条件-对象格式](#object-format)章节中提到过，可以创建自定义的查询条件类。例如，我们创建一个查询条件，它可以检查某些字段小于特定值的情况。
使用操作符格式，代码如下：


```php
[
    'and',
    '>', 'posts', $minLimit,
    '>', 'comments', $minLimit,
    '>', 'reactions', $minLimit,
    '>', 'subscriptions', $minLimit
]
```

当这样的查询条件被执行一次，它是很好的。当它在一个查询语句中被多次使用时可以有很多优化的地方。我们创建一个自定义查询条件对象来证实它。


Yii有一个[[yii\db\conditions\ConditionInterface|ConditionInterface]]接口，必须用它来标识这是一个表示查询条件的类。它需要实现`fromArrayDefinition()`方法，用来从数组格式创建查询条件。此时我们不需要它，抛出一个异常来完成此方法即可。



因为我们要创建自定义查询条件类，我们可以构建最适合当前任务的API。

```php
namespace app\db\conditions;

class AllGreaterCondition implements \yii\db\conditions\ConditionInterface
{
    private $columns;
    private $value;

    /**
     * @param string[] $columns 要大于$value的字段名数组
     * @param mixed $value 每个$column要比较的数值
     */
    public function __construct(array $columns, $value)
    {
        $this->columns = $columns;
        $this->value = $value;
    }
    
    public static function fromArrayDefinition($operator, $operands)
    {
        throw new InvalidArgumentException('Not implemented yet, but we will do it later');
    }
    
    public function getColumns() { return $this->columns; }
    public function getValue() { return $this->vaule; }
}
```

我们现在创建了一个查询条件对象：

```php
$conditon = new AllGreaterCondition(['col1', 'col2'], 42);
```

但是`QueryBuilder`还不知道可以从此对象生成SQL查询条件。现在我们需要为这个查询条件创建一个构建器，也就是必须实现[[yii\db\ExpressionBuilderInterface]]接口，它要实现`build()`方法。



```php
namespace app\db\conditions;

class AllGreaterConditionBuilder implements \yii\db\ExpressionBuilderInterface
{
    use \yii\db\ExpressionBuilderTrait; //包含构造喊叫和`queryBuilder`属性

    /**
     * @param ExpressionInterface $condition 要构建的查询条件
     * @param array $params 绑定的参数
     * @return AllGreaterCondition
     */ 
    public function build(ExpressionInterface $expression, array &$params = [])
    {
        $value = $condition->getValue();
        
        $conditions = [];
        foreach ($expression->getColumns() as $column) {
            $conditions[] = new SimpleCondition($column, '>', $value);
        }

        return $this->queryBuilder->buildCondition(new AndCondition($conditions), $params);
    }
}
```

接下来，让[[yii\db\QueryBuilder|QueryBuilder]]知道我们的新查询条件就简单了——添加一个映射到`expressionBuilders`数组中，在应用配置中完成即可：


```php
'db' => [
    'class' => 'yii\db\mysql\Connection',
    // ...
    'queryBuilder' => [
        'expressionBuilders' => [
            'app\db\conditions\AllGreaterCondition' => 'app\db\conditions\AllGreaterConditionBuilder',
        ],
    ],
],
```

现在我们可以在`where()`使用此查询条件了：

```php
$query->andWhere(new AllGreaterCondition(['posts', 'comments', 'reactions', 'subscriptions'], $minValue));
```

如果我们需要使用操作符格式来创建自定义查询条件，我们应在[[yii\db\QueryBuilder::conditionClasses|QueryBuilder::conditionClasses]]中声明它：


```php
'db' => [
    'class' => 'yii\db\mysql\Connection',
    // ...
    'queryBuilder' => [
        'expressionBuilders' => [
            'app\db\conditions\AllGreaterCondition' => 'app\db\conditions\AllGreaterConditionBuilder',
        ],
        'conditionClasses' => [
            'ALL>' => 'app\db\conditions\AllGreaterCondition',
        ],
    ],
],
```

并在`app\db\conditions\AllGreaterCondition`中创建一个真正的`AllGreaterCondition::fromArrayDefinition()`方法实现：


```php
namespace app\db\conditions;

class AllGreaterCondition implements \yii\db\conditions\ConditionInterface
{
    // ... see the implementation above
     
    public static function fromArrayDefinition($operator, $operands)
    {
        return new static($operands[0], $operands[1]);
    }
}
```
    
然后我们就可以使用更简短的操作符格式来创建自定义查询条件了：

```php
$query->andWhere(['ALL>', ['posts', 'comments', 'reactions', 'subscriptions'], $minValue]);
```

你可能注意到了，这里使用到了两个概念：表达式和条件。[[yii\db\ExpressionInterface]]用于标识对象，它需要一个表达式构建器类，实现要构建的[[yii\db\ExpressionBuilderInterface]]类。还有一个[[yii\db\condition\ConditionInterface]]类，它应用于对象，继承自[[yii\db\ExpressionInterface|ExpressionInterface]]，如上所示可以从数组定义中创建这些对象，但也需要构建器。





总结：

- Expression –  表达式，它是数据集的数据转换对象，可以被编译为SQL语句（操作符、字符串、数组、JSON等等）
- Condition – 是一个表达式的超集，聚集了多个表达式（或标量值），它们可以被编译为一个SQL查询条件。



你可以创建自己的类实现[[yii\db\ExpressionInterface|ExpressionInterface]]，隐藏转换数据到SQL语句的复杂过程，在[下一章节](db-active-record.md)你可以学习到更多关于表达式的实例。
