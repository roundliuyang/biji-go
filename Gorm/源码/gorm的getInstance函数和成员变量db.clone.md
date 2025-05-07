# gorm的getInstance函数和成员变量db.clone

>db.clone的值控制函数getInstance是否返回新实例，一级新实列复制原实例的哪些数据。

## 问题

看下面一段代码，其中的三次查询为何不会相会影响，查询一的Where不会带到查询二和查询三。

```go
package main

import (
	"fmt"
	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

// 定义模型
type Product struct {
	gorm.Model
	Code  string
	Price uint
}

func main() {
	// 连接数据库
	db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}

	// 自动迁移模式
	db.AutoMigrate(&Product{})

	// 插入3条示例数据
	db.Create(&Product{Code: "D42", Price: 100})
	db.Create(&Product{Code: "D43", Price: 200})
	db.Create(&Product{Code: "D44", Price: 300})

	// 根据不同条件查询
	var products []Product
    
    // 查询一
	db.Where("price > ?", 150).Find(&products) // 查询价格大于150的产品
	fmt.Println("Products with price > 150:", products)

    // 查询二
	db.Where("code LIKE ?", "%42%").Find(&products) // 查询代码包含42的产品
	fmt.Println("Products with code containing 42:", products)

    // 查询三
	db.Where("price BETWEEN ? AND ?", 100, 200).Find(&products) // 查询价格在100到200之间的产品
	fmt.Println("Products with price between 100 and 200:", products)
}
```



## 变量db.clone

```go
// DB GORM DB definition
type DB struct {
	*Config
	Error        error
	RowsAffected int64
	Statement    *Statement
	clone        int
}
```



### db.clone取值

截止版本，db.clone 只有三个取值，分别为 0、1 和 2 。

- 取值0：

是默认值，表示当前实例不是克隆的，也没有开启事务。在这种情况下，所有的数据库操作都会直接应用到原始的数据库连接上。

- 取值1：

创建一个新的 Statement 实例，并将当前数据库连接的一些属性（如连接池、上下文、语句片段、变量和跳过的钩子）复制到新的 Statement 实例中。

- 取值2：

克隆当前数据库连接的 Statement 实例，并将新的 DB 实例设置为 Statement 的 DB 属性。

## 函数db.getInstance

```go
func (db *DB) getInstance() *DB {
    // 当 db.clone > 0 时，返回一个新的 *DB 实例（tx）
	if db.clone > 0 {
		tx := &DB{Config: db.Config, Error: db.Error}

		if db.clone == 1 {
			// clone with new statement
			tx.Statement = &Statement{
				DB:        tx,
				ConnPool:  db.Statement.ConnPool,
				Context:   db.Statement.Context,
				Clauses:   map[string]clause.Clause{}, // 清空了，已设置的 where、join 等清空了
				Vars:      make([]interface{}, 0, 8), // 清空了
				SkipHooks: db.Statement.SkipHooks,
			}
		} else { // 必然是 db.clone == 2
			// with clone statement
			tx.Statement = db.Statement.clone() // 复制 Clauses、Vars、Joins 等，已设置的 where、join 等继续保留
			tx.Statement.DB = tx
		}

		return tx // 没有设置 tx.clone，因此 tx.clone 值为 0
	}

    // 当 db.clone <= 0 时，直接返回原始的 db 实例
	return db // 没对 db 做任何修改，因此 db.clone 值保持不变
}
```



## Clauses

```go
type Clause struct {
	Name                string // WHERE
	BeforeExpression    Expression
	AfterNameExpression Expression
	AfterExpression     Expression
	Expression          Expression
	Builder             ClauseBuilder
}

type Statement struct {
    Clauses              map[string]clause.Clause
    Vars                 []interface{}
    Joins                []join
    Omits                []string
    Selects              []string
}
```

Statement 中的 Clauses 是一个 map 类型，key 是字符串，value 是 clause.Clause 接口。它用于存储 SQL 查询中的各种子句（如 SELECT、FROM、WHERE、ORDER BY 等）。在构建 SQL 查询时，GORM 会根据这些子句生成对应的 SQL 语句。通过向 Clauses 添加或修改子句，可以灵活地自定义查询的行为。



## 设置db.clone值为1的函数

### gorm.open函数

```go
// Open initialize db session based on dialector
func Open(dialector Dialector, opts ...Option) (db *DB, err error) {
    db = &DB{Config: config, clone: 1}
}
```



### db.Session函数

```go
// Session create new db session
func (db *DB) Session(config *Session) *DB {
    var (
        txConfig = *db.Config
        tx       = &DB{
            Config:    &txConfig,
            Statement: db.Statement,
            Error:     db.Error,
            clone:     1,
        }
    )
}
```



## 设置db.clone值为2的函数

### db.Session函数

```go
// Session create new db session
func (db *DB) Session(config *Session) *DB {
	if !config.NewDB {
		tx.clone = 2
	}
}
```



### 调用db.Session函数的函数

- db.WithContext

```scss
// WithContext change current instance db's context to ctx
func (db *DB) WithContext(ctx context.Context) *DB {
	return db.Session(&Session{Context: ctx})
}
```



## 问题答案

看 Where 的实现，调用 db.getInstance 创建了新的实例。但因为 db.clone 的值为 1，因此新的实例会清空之前设置的 Where 等。

```go
func (db *DB) Where(query interface{}, args ...interface{}) (tx *DB) {
	tx = db.getInstance()
	if conds := tx.Statement.BuildCondition(query, args...); len(conds) > 0 {
		tx.Statement.AddClause(clause.Where{Exprs: conds})
	}
	return
}
```

对于示例，因为每个 Where 都会创建新的实例，所以 Where 作用彼此间完全隔离的。Where 值作用在无关联的不同实例上，因此不会有任何相互影响。



## 附：观察db.cone的变化

函数 gorm.Open 会将 db.clone 值设置为 1，对于下列代码，db.cone、db2.cone、db3.cone、db4.cone、db5.cone、db6.cone、db7.cone 的值分别为多少？

```go
//
db := gorm.Open(dsn)       // db.clone 值为 1
db1 := db                  // db1.clone 值为 1，db1 和 db 指向同一个地方，两者就是同一个对象
db2 := db.WithContext(ctx) // db2.clone 值为 2，db2 和 db 指向不同的地方，两者不是同一个对象
db3 := db.Table("t_test")  // db3.clone 值为 0，db3 和 db 指向不同的地方，两者不是同一个对象
db4 := db.Model(&User{})   // db4.clone 值为 0，db3 和 db 指向不同的地方，两者不是同一个对象
db5 := db.Find(&user)      // db5.clone 值为 0，db3 和 db 指向不同的地方，两者不是同一个对象
db6 := db.Begin()          // db6.clone 值为 1，db3 和 db 指向不同的地方，两者不是同一个对象
(dlv) p db.clone
1
(dlv) p db.clone
1
(dlv) p db1.clone
1
(dlv) p db2.clone
2
(dlv) p db3.clone
0
(dlv) p db4.clone
0
(dlv) p db5.clone
0
(dlv) p db6.clone
1

(dlv) p &(*db)
("*gorm.io/gorm.DB")(0xc0001fe630)
(dlv) p &(*db1)
("*gorm.io/gorm.DB")(0xc0001fe630)
(dlv) p &(*db2)
("*gorm.io/gorm.DB")(0xc00028e1b0)
(dlv) p &(*db3)
("*gorm.io/gorm.DB")(0xc00028e330)
(dlv) p &(*db4)
("*gorm.io/gorm.DB")(0xc00028e420)
(dlv) p &(*db5)
("*gorm.io/gorm.DB")(0xc00028e4e0)
(dlv) p &(*db6)
("*gorm.io/gorm.DB")(0xc00028e7b0)
```

