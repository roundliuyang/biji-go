# 为什么go 数据表的orm，很多地方用指针呢

## 例子

```go
package models

import (
    "time"
    "github.com/jinzhu/gorm"
)

type YyMedicineSaleOrder struct {
    ID                    uint   `gorm:"primary_key;AUTO_INCREMENT;column:id" json:"id"` // 匹配自增ID
    HospitalID            *int   `gorm:"column:hospital_id" json:"hospital_id"` // 医院id
    OrderNo               *string `gorm:"size:100;column:order_no" json:"order_no"` // 进货单号
    Path                  *string `gorm:"size:255;column:path" json:"path"` // 文件路径
    Bz                    *string `gorm:"size:100;column:bz" json:"bz"` // 备注
    IsMedicineMatchUpdate int    `gorm:"NOT NULL;DEFAULT:2;column:is_medicine_match_update" json:"is_medicine_match_update"` // 是否有药物匹配更新,1是，2否
    Status                int    `gorm:"NOT NULL;DEFAULT:1;column:status" json:"status"` // 1,未匹配完成。2，匹配完成（无需匹配）3，生成应收中，4，生成应收成功，5，生成应收失败。6,匹配过(未匹配完)
    Error                 *string `gorm:"size:255;column:error" json:"error"` // job执行错误信息
    CreatedAt             *time.Time `gorm:"column:created_at" json:"created_at"` 
    Createtime            uint64 `gorm:"column:createtime" json:"createtime"` // 创建时间
    JobStatus             *int   `gorm:"DEFAULT:2;column:job_status" json:"job_status"` // job是否正在执行，1,是，2否。
    DateMonth             *time.Time `gorm:"type:date;column:date_month" json:"date_month"` // 单据生成的年月
    Type                  int8   `gorm:"NOT NULL;column:type" json:"type"` // 类型：1外采,2内部
}

// TableName sets the insert table name for this struct type
func (y *YyMedicineSaleOrder) TableName() string {
    return "yy_medicine_sale_order"
}
```

**解析：GORM 鼓励使用指针类型**
在 Go 语言的 GORM 中，指针用来表示数据库中的 NULL 值。如果我们在结构体中定义字段类型为非指针的基本类型，例如 int 或者 string，那么在 Go 语言中这些类型是无法表示 NULL 的。如果数据库字段有可能出现 NULL 情况，GORM 查询出来对于 Go 语言来说都会是类型的零值，例如 int 类型字段的零值是 0，string 类型字段的零值是””。

但是在一些场景中，数据库字段的 NULL 和 零值 在业务含义上可能是两种完全不同的状态，所以为了解决这个问题，GORM 鼓励使用指针类型。当数据库字段为 NULL 时，对应的将是 nil；当数据库字段有具体值时，将对应具体的值。

当然，如果你确定这个字段一定不会是 NULL 值，你也可以直接使用非指针类型。例如我看到你的部分字段有 NOT NULL 约束，那么这些字段在 Go 语言的结构体定义上可以不用指针。





在数据库中，**布尔类型字段是否允许为 `NULL`** 取决于你的数据库设计需求和具体的业务含义。以下是对布尔类型是否允许为 `NULL` 的常见实践说明：

## 布尔类型允许为 NULL（用指针表示）

在有些场景下，布尔值不仅有“是”（`true`）和“否”（`false`），还需要表示**“未设置”或“未知”**，这时候就应该允许为 `NULL`。在 GORM 中，可以使用 `*bool`（布尔指针类型）来表示这种三态逻辑：

```go
IsActive *bool `gorm:"column:is_active" json:"is_active"` // true, false, or NULL
```

- `true`：表示“是”
- `false`：表示“否”
- `nil`：表示“未设置”或“未知”

这在某些业务中非常常见，例如：

- 用户是否启用双因素认证（可能还没设置过）
- 某项设置是否开启（可能还未初始化）



## **布尔类型不允许为 NULL（用值类型）**

如果你确定这个布尔字段一定会被设置为 `true` 或 `false`，数据库字段可以加上 `NOT NULL` 约束，同时在 GORM 中使用 `bool` 作为字段类型即可：

```go
IsActive bool `gorm:"NOT NULL;DEFAULT:false;column:is_active" json:"is_active"`
```

这种场景适用于：

- 系统在创建记录时就会明确初始化该字段
- NULL 没有业务意义，所有状态都应明确



##  总结

| 需求                       | 数据库字段是否为 NULL | GORM 字段类型 |
| -------------------------- | --------------------- | ------------- |
| 三态逻辑（是 / 否 / 未知） | 是                    | `*bool`       |
| 二态逻辑（是 / 否）        | 否（使用 NOT NULL）   | `bool`        |

------

你当前的表结构中如果有布尔逻辑但也可能为 NULL（比如 job 是否正在执行但尚未开始），使用 `*bool` 是更稳妥的选择。