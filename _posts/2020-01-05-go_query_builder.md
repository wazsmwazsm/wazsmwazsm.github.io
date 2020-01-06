---
layout: post
title:  "解放数据库查询, 写一个 go 的查询构造器"
date:   2020-01-05 09:25:08
categories: golang
excerpt: 针对直接编写 SQL 不灵活, 使用 ORM 语法繁琐语义不明确, 提供一种 orm 和 raw sql 之间的中间方案, 查询构造器 (Query Builder)
---

## 前言

### 数据库查询
在 go 开发中, 查询数据库一般有两种选择:
- 使用 orm (gorm\xorm 等)
- 直接写 SQL

直接编写 SQL 语义清晰, 不易出错, 但是遇到多个可变条件时显得不灵活

ORM 有模型关系, 记录预加载 (sql 生成优化) 等功能, 但是 sql 语句对开发人员相对透明, 管了太多数据库相关的东西, 相对封闭, 语法晦涩语义不明确, 想要操作 db 连接、构造复杂 SQL 很繁琐

### 查询构造器

对于查询场景少、查询条件相对固定的系统, 直接写 SQL 无疑是一种好的选择。那么, 对于 SQL 多变的场景而又不想使用 orm 的开发者, 如何能快速开发数据层呢? 

go 的官方包已经提供了好用的 database/sql 工具, 也有各个数据库的驱动包, 屏蔽了底层驱动差异, 使数据库查询变得简单, 只需提供 SQL 语句和占位符参数即可快速查询, 也无需考虑 SQL 注入等问题。那么, 只要解决了 SQL 语句和占位符参数的构造问题, 就解决了直接写 SQL 的灵活性问题。

为了解决 SQL 语句和占位符参数的构造问题, 我们需要查询构造器 (Query Builder)。简而言之, 查询构造器就是利用 database/sql 的优势, 提供了一种 orm 和 raw sql 之间的中间方案。有了查询构造器, 你可以在遇到不定 SQL 时动态构造 SQL, 遇到复杂确定 SQL 时直接写原生 SQL, 使数据查询更加灵活可控。


## 思路

### 做什么
查询构造器, 顾名思义, 最主要的就是构造。构造什么? 查询语句。查询语句本身就是一个满足标准 SQL 规范的字符串, 所以我们要做查询构造器, 主要的任务就是构造字符串。

### 拆解 SQL

在构造一条 SQL 之前, 不妨看看一条 SQL 是什么样的吧。

```sql
SELECT `name`,`age`,`school` FROM `test` WHERE `name` = 'jack'
```

复杂点的, 带联合查询、分组、排序、分页
```sql
SELECT `t1`.`name`,`t1`.`age`,`t2`.`teacher`,`t3`.`address` FROM `test` as t1 LEFT JOIN `test2` as `t2` ON `t1`.`class` = `t2`.`class` INNER JOIN `test3` as t3 ON `t1`.`school` = `t3`.`school` WHERE `t1`.`age` >= 20 GROUP BY `t1`.`age` HAVING COUNT(`t1`.`age`) > 2 ORDER BY `t1`.`age` DESC LIMIT 10 OFFSET 0
```

当然, 标准 SQL 还有很多语法规定, 这里就不一一举例。而对于规范中最常用的语法, 我们的查询构造器必须要有构造它们的能力

一个标准的查询语句结构如下:
```sql
SELECT [字段] FROM [表名] [JOIN 子句] [WHERE 子句] [GROUP BY 子句] [HAVING 子句] [ORDER BY 子句] [LIMIT 子句]
```

其中 JOIN 子句、WHERE 子句、 HAVING 子句和 LIMIT 子句会用到占位符参数

再看 INSERT、UPDATE、DELETE 操作的结构:

INSERT
```sql
INSERT INTO [表名] ([字段名]) VALUES ([要插入的值])
```

要插入的值会用到占位符参数

UPDATE
```sql
UPDATE [表名] [SET 子句] [WHERE 子句] 
```

SET 子句和 WHERE 子句会用到占位符参数


DELETE
```sql
DELETE FROM [表名] [WHERE 子句] 
```
WHERE 子句会用到占位符参数

OK, 拆解后是不是觉得 SQL 语句的基本结构很简单? 要实现查询构造器, 只需按照这些语句的结构构造出相应的字符串, 并保存需要的占位符参数即可。


## 实现

有了思路, 实现起来就简单了。

参考其他语言的查询构造器, 方法名直接体现 SQL 语法, 多为链式调用:
```php
$db.table("`test`").
	where("a", ">", 20).
	where("b", "=", "aaa").
	get()
```
要实现查询构造器, 这是一个好的示范。

话不多说, 开写!

首先定义我们的 SQLBuilder 类型:

```go
type SQLBuilder struct {
	_select       string // select 子句字符串
	_insert       string // insert 子句字符串
	_update       string // update 子句字符串
	_delete       string // delete 子句字符串
	_table        string // 表名
	_join         string // join 子句字符串
	_where        string // where 子句字符串
	_groupBy      string // group by 子句字符串
	_having       string // having 子句字符串
	_orderBy      string // order by 子句字符串
	_limit        string // limit 子句字符串
	_insertParams []interface{} // insert 插入值需要的占位符参数
	_updateParams []interface{} // update SET 子句需要的占位符参数
	_whereParams  []interface{} // where 子句需要的占位符参数
	_havingParams []interface{} // having 子句需要的占位符参数
	_limitParams  []interface{} // limit 子句需要的占位符参数
	_joinParams   []interface{} // join 子句需要的占位符参数
}
```

SQLBuilder 的构造函数:
```go
func NewSQLBuilder() *SQLBuilder {
	return &SQLBuilder{}
}
```

### 获取 SQL 字符串

获取字符串很简单, 只要按照 SQL 的规定将各个子句组合即可。

获取 QuerySQL:
```go
var ErrTableEmpty = errors.New("table empty")

func (sb *SQLBuilder) GetQuerySQL() (string, error) {
	if sb._table == "" {
		return "", ErrTableEmpty
	}
	var buf strings.Builder

	buf.WriteString("SELECT ")
	if sb._select != "" {
		buf.WriteString(sb._select)
	} else {
		buf.WriteString("*")
	}
	buf.WriteString(" FROM ")
	buf.WriteString(sb._table)
	if sb._join != "" {
		buf.WriteString(" ")
		buf.WriteString(sb._join)
	}
	if sb._where != "" {
		buf.WriteString(" ")
		buf.WriteString(sb._where)
	}
	if sb._groupBy != "" {
		buf.WriteString(" ")
		buf.WriteString(sb._groupBy)
	}
	if sb._having != "" {
		buf.WriteString(" ")
		buf.WriteString(sb._having)
	}
	if sb._orderBy != "" {
		buf.WriteString(" ")
		buf.WriteString(sb._orderBy)
	}
	if sb._limit != "" {
		buf.WriteString(" ")
		buf.WriteString(sb._limit)
	}

	return buf.String(), nil
}
```

> tips: 上述代码使用 strings.Builder 包来拼接字符串。当然构造查询语句本身不是一个高频操作, 不考虑效率使用 + 来拼接也是可以的

获取 InsertSQL:
```go
var ErrInsertEmpty = errors.New("insert content empty")

func (sb *SQLBuilder) GetInsertSQL() (string, error) {
	if sb._table == "" {
		return "", ErrTableEmpty
	}
	if sb._insert == "" {
		return "", ErrInsertEmpty
	}

	var buf strings.Builder

	buf.WriteString("INSERT INTO ")
	buf.WriteString(sb._table)
	buf.WriteString(" ")
	buf.WriteString(sb._insert)

	return buf.String(), nil
}
```

获取 UpdateSQL:
```go
var ErrUpdateEmpty = errors.New("update content empty")

func (sb *SQLBuilder) GetUpdateSQL() (string, error) {
	if sb._table == "" {
		return "", ErrTableEmpty
	}

	if sb._update == "" {
		return "", ErrUpdateEmpty
	}

	var buf strings.Builder

	buf.WriteString("UPDATE ")
	buf.WriteString(sb._table)
	buf.WriteString(" ")
	buf.WriteString(sb._update)
	if sb._where != "" {
		buf.WriteString(" ")
		buf.WriteString(sb._where)
	}

	return buf.String(), nil
}
```

获取 DeteleSQL:
```go
func (sb *SQLBuilder) GetDeleteSQL() (string, error) {
	if sb._table == "" {
		return "", ErrTableEmpty
	}

	var buf strings.Builder

	buf.WriteString("DELETE FROM ")
	buf.WriteString(sb._table)
	if sb._where != "" {
		buf.WriteString(" ")
		buf.WriteString(sb._where)
	}

	return buf.String(), nil
}
```

### 获取占位符参数

同样, 我们要填充占位符 "?" 的参数也需要获得, query、insert、update、delete 拥有的参数类型都有差别, 也都有着不同的顺序

```go

func (sb *SQLBuilder) GetQueryParams() []interface{} {
	params := []interface{}{}
	params = append(params, sb._joinParams...)
	params = append(params, sb._whereParams...)
	params = append(params, sb._havingParams...)
	params = append(params, sb._limitParams...)
	return params
}

func (sb *SQLBuilder) GetInsertParams() []interface{} {
	params := []interface{}{}
	params = append(params, sb._insertParams...)
	return params
}

func (sb *SQLBuilder) GetUpdateParams() []interface{} {
	params := []interface{}{}
	params = append(params, sb._updateParams...)
	params = append(params, sb._whereParams...)
	return params
}

func (sb *SQLBuilder) GetDeleteParams() []interface{} {
	params := []interface{}{}
	params = append(params, sb._whereParams...)
	return params
}
```

### 表名设置

设置表名, 这里我们设置完成后返回 SQLBuilder 指针自己, 可以完成链式调用。之后大部分方法都会使用这种方式。

```go
func (sb *SQLBuilder) Table(table string) *SQLBuilder {

	sb._table = table

	return sb
}
```

用例:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
        Select("*").
        GetQuerySQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetQueryParams()

	log.Println(sql)    // SELECT * FROM `test`
	log.Println(params) // []
}
```

### select 子句

设置 select 子句, 支持多个参数用逗号隔开, 注意最后一个逗号要去掉

```go
func (sb *SQLBuilder) Select(cols ...string) *SQLBuilder {
	var buf strings.Builder

	for k, col := range cols {

		buf.WriteString(col)

		if k != len(cols)-1 {
			buf.WriteString(",")
		}
	}

	sb._select = buf.String()

	return sb
}
```

用例:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
		Select("`age`", "COUNT(age)").
		GetQuerySQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetQueryParams()

	log.Println(sql)    // SELECT `age`,COUNT(age) FROM `test`
	log.Println(params) // []
}
```

### where 子句

#### where

对于 where 子句, 第一个 where 条件需要 WHERE 关键字, 再有其它条件, 会通过 AND 和 OR 来连接, 那么我们可以增加 Where() 和 OrWhere() 方法, 两个方法公共逻辑可以提出来:

```go
func (sb *SQLBuilder) Where(field string, condition string, value interface{}) *SQLBuilder {
	return sb.where("AND", condition, field, value)
}

func (sb *SQLBuilder) OrWhere(field string, condition string, value interface{}) *SQLBuilder {
	return sb.where("OR", condition, field, value)
}

func (sb *SQLBuilder) where(operator string, condition string, field string, value interface{}) *SQLBuilder {
	var buf strings.Builder

	buf.WriteString(sb._where) // 载入之前的 where 子句

	if buf.Len() == 0 { // where 子句还没设置
		buf.WriteString("WHERE ")
	} else { // 已经设置, 拼接 OR 或 AND 操作符
		buf.WriteString(" ")
		buf.WriteString(operator)
		buf.WriteString(" ")
	}

	buf.WriteString(field) // 拼接字段

	buf.WriteString(" ")
	buf.WriteString(condition) // 拼接条件 =、!=、<、>、like 等
	buf.WriteString(" ")
	buf.WriteString("?") // 拼接占位符

	sb._where = buf.String() // 写字符串

	sb._whereParams = append(sb._whereParams, value) // push 占位符参数到数组

	return sb
}
```

用例:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
		Select("`name`", "`age`", "`school`").
		Where("`name`", "=", "jack").
		Where("`age`", ">=", 18).
		OrWhere("`name`", "like", "%admin%").
		GetQuerySQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetQueryParams()

	log.Println(sql)    // SELECT `name`,`age`,`school` FROM `test` WHERE `name` = ? AND `age` >= ? OR `name` like ?
	log.Println(params) // [jack 18 %admin%]
}
```

上述代码可以解决简单的条件子句, 如果遇到 WHERE a = ? AND (b = ? OR c = ?) 这样的复杂子句, 该如何构造呢? 面对这种场景, 我们需要提供书写原生 where 子句的能力, 增加 WhereRaw() 和 OrWhereRaw() 方法:

```go
func (sb *SQLBuilder) WhereRaw(s string, values ...interface{}) *SQLBuilder {
	return sb.whereRaw("AND", s, values)
}

func (sb *SQLBuilder) OrWhereRaw(s string, values ...interface{}) *SQLBuilder {
	return sb.whereRaw("OR", s, values)
}

func (sb *SQLBuilder) whereRaw(operator string, s string, values []interface{}) *SQLBuilder {
	var buf strings.Builder

	buf.WriteString(sb._where) // append

	if buf.Len() == 0 {
		buf.WriteString("WHERE ")
	} else {
		buf.WriteString(" ")
		buf.WriteString(operator)
		buf.WriteString(" ")
	}

	buf.WriteString(s) // 直接使用 raw SQL 字符串
	sb._where = buf.String()

	for _, value := range values {
		sb._whereParams = append(sb._whereParams, value)
	}

	return sb
}
```

用例:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
		Select("`name`", "`age`", "`school`").
		WhereRaw("`title` = ?", "hello").
		Where("`name`", "=", "jack").
		OrWhereRaw("(`age` = ? OR `age` = ?) AND `class` = ?", 22, 25, "2-3").
		GetQuerySQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetQueryParams()

	log.Println(sql)    // SELECT `name`,`age`,`school` FROM `test` WHERE `title` = ? AND `name` = ? OR (`age` = ? OR `age` = ?) AND `class` = ?
	log.Println(params) // [hello jack 22 25 2-3]
}

```


#### where in

where in 也是常见的 where 子句, where in 子句分为 where in、or where in、where not in、or where not in 四种模式, 占位符数量等于 where in 的集合数量。

我们希望构造 where in 子句的方法入参是一个 slice, 占位符的数量等于 slice 的长度, 那么我们需要封装一个生成占位符的函数:
```go
func GenPlaceholders(n int) string {
	var buf strings.Builder

	for i := 0; i < n-1; i++ {
		buf.WriteString("?,") // 生成 n-1 个 "?" 占位符
	}

	if n > 0 {
		buf.WriteString("?") // 生成最后一个占位符, 如果 n <= 0 则不生成任何占位符
	}

	return buf.String()
}

```

按照 where in 子句的四种模式, 增加 WhereIn() OrWhereIn() WhereNotIn() OrWhereNotIn() 方法:
```go
func (sb *SQLBuilder) WhereIn(field string, values ...interface{}) *SQLBuilder {
	return sb.whereIn("AND", "IN", field, values)
}

func (sb *SQLBuilder) OrWhereIn(field string, values ...interface{}) *SQLBuilder {
	return sb.whereIn("OR", "IN", field, values)
}

func (sb *SQLBuilder) WhereNotIn(field string, values ...interface{}) *SQLBuilder {
	return sb.whereIn("AND", "NOT IN", field, values)
}

func (sb *SQLBuilder) OrWhereNotIn(field string, values ...interface{}) *SQLBuilder {
	return sb.whereIn("OR", "NOT IN", field, values)
}

func (sb *SQLBuilder) whereIn(operator string, condition string, field string, values []interface{}) *SQLBuilder {
	var buf strings.Builder

	buf.WriteString(sb._where) // append

	if buf.Len() == 0 {
		buf.WriteString("WHERE ")
	} else {
		buf.WriteString(" ")
		buf.WriteString(operator)
		buf.WriteString(" ")
	}

	buf.WriteString(field)

	plhs := GenPlaceholders(len(values)) // 生成占位符
	buf.WriteString(" ")
	buf.WriteString(condition)
	buf.WriteString(" ")
	buf.WriteString("(")
	buf.WriteString(plhs) // 拼接占位符
	buf.WriteString(")")

	sb._where = buf.String()

	for _, value := range values { 
		sb._whereParams = append(sb._whereParams, value) // push 占位符参数
	}

	return sb
}
```

用例:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
		Select("`name`", "`age`", "`school`").
		WhereIn("`id`", 1, 2, 3).
		OrWhereNotIn("`uid`", 2, 4).
		GetQuerySQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetQueryParams()

	log.Println(sql)    // SELECT `name`,`age`,`school` FROM `test` WHERE `id` IN (?,?,?) OR `uid` NOT IN (?,?)
	log.Println(params) // [1 2 3 2 4]
}

```

### group by 子句

group by 子句可以根据多个字段分组:
```go
func (sb *SQLBuilder) GroupBy(fields ...string) *SQLBuilder {
	var buf strings.Builder

	buf.WriteString("GROUP BY ")

	for k, field := range fields {

		buf.WriteString(field)

		if k != len(fields)-1 {
			buf.WriteString(",")
		}
	}

	sb._groupBy = buf.String()

	return sb
}
```

having 子句和 where 子句基本相同, 这里就不费篇幅说明了, 详细见 [QueryBuilder/builder/builder.go](https://github.com/wazsmwazsm/QueryBuilder/blob/master/builder/builder.go)

用例:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
		Select("`school`", "`class`", "COUNT(*) as `ct`").
		GroupBy("`school`", "`class`").
		Having("`ct`", ">", "2").
		GetQuerySQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetQueryParams()

	log.Println(sql)    // SELECT `school`,`class`,COUNT(*) as `ct` FROM `test` GROUP BY `school`,`class` HAVING `ct` > ?
	log.Println(params) // [2]
}

```


### order by 子句和 limit 子句

order by 子句可以根据多个字段来排序:
```go
func (sb *SQLBuilder) OrderBy(operator string, fields ...string) *SQLBuilder {
	var buf strings.Builder

	buf.WriteString("ORDER BY ")

	for k, field := range fields {

		buf.WriteString(field)

		if k != len(fields)-1 {
			buf.WriteString(",")
		}
	}

	buf.WriteString(" ")
	buf.WriteString(operator) // DESC 或 ASC

	sb._orderBy = buf.String()

	return sb
}
```

limit 来限制查询的结果, 这里我们使用 LIMIT OFFSET 语法, 这个语法是标准 SQL 规定的, LIMIT x,x 这个形式只有 mysql 支持

```go
func (sb *SQLBuilder) Limit(offset, num interface{}) *SQLBuilder {
	var buf strings.Builder

	buf.WriteString("LIMIT ? OFFSET ?")

	sb._limit = buf.String()

	sb._limitParams = append(sb._limitParams, num, offset)

	return sb
}

```

用例:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
		Select("`name`", "`age`", "`school`").
		Where("`name`", "=", "jack").
		Where("`age`", ">=", 18).
		OrderBy("DESC", "`age`", "`class`").
		Limit(1, 10).
		GetQuerySQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetQueryParams()

	log.Println(sql)    // SELECT `name`,`age`,`school` FROM `test` WHERE `name` = ? AND `age` >= ? ORDER BY `age`,`class` DESC LIMIT ? OFFSET ?
	log.Println(params) // [jack 18 10 1]
}

```

### join 子句

使用 join 子句后, SQL 变得复杂。标准 SQL join 有 left join、right join、inner join、full join 几种模式 join 子句的 on 条件类似 where 子句, 连表后需要给表起别名用来区分字段所属...面对这样灵活多变的语法, 我们这里较好的方式就是提供 raw sql 的形式来处理 join 操作:
```go
func (sb *SQLBuilder) JoinRaw(join string, values ...interface{}) *SQLBuilder {
	var buf strings.Builder

	buf.WriteString(sb._join)
	if buf.Len() != 0 {
		buf.WriteString(" ")
	}
	buf.WriteString(join) // 拼接 raw join sql

	sb._join = buf.String()

	for _, value := range values {
		sb._joinParams = append(sb._joinParams, value)
	}

	return sb
}
```

用例 (构造一个复杂的查询):
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test` as t1").
		Select("`t1`.`name`", "`t1`.`age`", "`t2`.`teacher`", "`t3`.`address`").
		JoinRaw("LEFT JOIN `test2` as `t2` ON `t1`.`class` = `t2`.`class`").
		JoinRaw("INNER JOIN `test3` as t3 ON `t1`.`school` = `t3`.`school`").
		Where("`t1`.`age`", ">=", 18).
		GroupBy("`t1`.`age`").
		Having("COUNT(`t1`.`age`)", ">", 2).
		OrderBy("DESC", "`t1`.`age`").
		Limit(1, 10).
		GetQuerySQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetQueryParams()

	log.Println(sql)    // SELECT `t1`.`name`,`t1`.`age`,`t2`.`teacher`,`t3`.`address` FROM `test` as t1 LEFT JOIN `test2` as `t2` ON `t1`.`class` = `t2`.`class` INNER JOIN `test3` as t3 ON `t1`.`school` = `t3`.`school` WHERE `t1`.`age` >= ? GROUP BY `t1`.`age` HAVING COUNT(`t1`.`age`) > ? ORDER BY `t1`.`age` DESC LIMIT ? OFFSET ?
	log.Println(params) // [18 2 10 1]
}

```

### insert

insert SQL 构建:
```go
func (sb *SQLBuilder) Insert(cols []string, values ...interface{}) *SQLBuilder {
	var buf strings.Builder

    // 拼接字段
	buf.WriteString("(")
	for k, col := range cols {

		buf.WriteString(col)

		if k != len(cols)-1 {
			buf.WriteString(",")
		}
	}
	buf.WriteString(") VALUES (")

    // 拼接占位符
	for k := range cols {
		buf.WriteString("?")
		if k != len(cols)-1 {
			buf.WriteString(",")
		}
	}
	buf.WriteString(")")

	sb._insert = buf.String()

	for _, value := range values { // push 占位符参数
		sb._insertParams = append(sb._insertParams, value)
	}

	return sb
}
```

用例:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
		Insert([]string{"`name`", "`age`"}, "jack", 18).
		GetInsertSQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetInsertParams()

	log.Println(sql)    // INSERT INTO `test` (`name`,`age`) VALUES (?,?)
	log.Println(params) // [jack 18]
}

```

### update
update SQL 构建:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
		Update([]string{"`name`", "`age`"}, "jack", 18).
		Where("`id`", "=", 11).
		GetUpdateSQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetUpdateParams()

	log.Println(sql)    // UPDATE `test` SET `name` = ?,`age` = ? WHERE `id` = ?
	log.Println(params) // [jack 18 11]
}

```

### delete
delete SQL 构建:
```go
package main

import (
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

func main() {
	sb := builder.NewSQLBuilder()

	sql, err := sb.Table("`test`").
		Where("`id`", "=", 11).
		GetDeleteSQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetDeleteParams()

	log.Println(sql)    // DELETE FROM `test` WHERE `id` = ?
	log.Println(params) // [11]
}

```

OK, 查询构造器的实现到此结束, 是不是很简单呢?

## 使用

查询构造器实现了, 那么就结合 database/sql 用用吧!

以 mysql 为例:
```go
package main

import (
	"database/sql"
	"fmt"
	_ "github.com/go-sql-driver/mysql"
	"github.com/wazsmwazsm/QueryBuilder/builder"
	"log"
)

// Info 定义一个数据模型, 用于接收查询数据
type Info struct {
	Age      int
	AgeCount int
}

func main() {
    // 创建 mysql 连接
	dataSource := fmt.Sprintf("%s:%s@tcp(%s:%v)/%s?charset=utf8",
		"test", "test", "127.0.0.1", 3306, "test")
	mysqlConn, err := sql.Open("mysql", dataSource)
	if err != nil {
		log.Panic("Db connect failed!" + err.Error())
	}

    // 创建查询构造器实例
	sb := builder.NewSQLBuilder()

	querySQL, err := sb.Table("`test`").
		Select("`age`", "COUNT(age)").
		GroupBy("`age`").
		GetQuerySQL()
	if err != nil {
		log.Fatal(err)
	}

	params := sb.GetQueryParams()

    // 执行查询
	rows, err := mysqlConn.Query(querySQL, params...)
	if err != nil {
		log.Panic(err)
	}
	defer rows.Close()

    // 查询数据绑定到 info 结构中
	infos := []*Info{}
	for rows.Next() {
		info := new(Info)
		if err := rows.Scan(
			&info.Age,
			&info.AgeCount,
		); err != nil {
			log.Panicln(err)
		}
		infos = append(infos, info)
	}

	for _, info := range infos {
		fmt.Println(info)
	}

}

```

## 源码地址

该项目的全部源码详见 [QueryBuilder](https://github.com/wazsmwazsm/QueryBuilder "QueryBuilder"), 单元测试已 100% 覆盖
