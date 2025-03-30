# rofig cron

## 代码

```go
package main

import (
	"fmt"
	"log"
	"time"

	"github.com/robfig/cron"
)

func main() {
	log.Println("Starting...")

	c := cron.New()
	if err := c.AddFunc("* * * * * *", func() {
		CronFunc()
	}); err != nil {
		fmt.Errorf("err:%v", err)
		return
	}

	c.Start()

	for {
		select {}
	}
}

func CronFunc() {
	fmt.Println("CronFunc at:", time.Now())
}
```

结果

```none
go run main.go
2023/01/15 14:03:21 Starting...
CronFunc at: 2023-01-15 14:03:22.000246 +0800 CST m=+0.477580184
CronFunc at: 2023-01-15 14:03:23.000721 +0800 CST m=+1.478031795
CronFunc at: 2023-01-15 14:03:24.000335 +0800 CST m=+2.477621763
CronFunc at: 2023-01-15 14:03:25.000217 +0800 CST m=+3.477479751
CronFunc at: 2023-01-15 14:03:26.001094 +0800 CST m=+4.478333060
CronFunc at: 2023-01-15 14:03:27.001083 +0800 CST m=+5.478298348
CronFunc at: 2023-01-15 14:03:28.001084 +0800 CST m=+6.478275654
CronFunc at: 2023-01-15 14:03:29.001085 +0800 CST m=+7.478252830
CronFunc at: 2023-01-15 14:03:30.00095 +0800 CST m=+8.478094350
...
```

上例很简单，就是每秒执行一下。



## cron表达式

这个包的定时是到秒级别的，下面就详细介绍一下表达式规则。

| 字段名                         | 是否必填 | 允许的值        | 允许的特殊字符 |
| :----------------------------- | :------- | :-------------- | :------------- |
| 秒（Seconds）                  | Yes      | 0-59            | * / , -        |
| 分（Minutes）                  | Yes      | 0-59            | * / , -        |
| 时（Hours）                    | Yes      | 0-23            | * / , -        |
| 一个月中的某天（Day of month） | Yes      | 1-31            | * / , - ?      |
| 月（Month）                    | Yes      | 1-12 or JAN-DEC | * / , -        |
| 星期几（Day of week）          | Yes      | 0-6 or SUN-SAT  | * / , - ?      |

## 特殊字符



### 星号 ( * )

星号表示将匹配字段的所有值



### 斜线 ( / )

描述范围的增量，表现为 “N-M/x” 的形式，例如 3-59/15 表示此时的第三分钟和此后的每 15 分钟，到59分钟为止。即从 N 开始，使用增量x直到M结束。它不会重复



### 逗号 ( , )

逗号用于分隔列表中的项目。例如，在 Day of week 使用“MON，WED，FRI”将意味着星期一，星期三和星期五

### 连字符 ( - )

连字符用于定义范围。例如，在Hours使用 “9 - 17” 表示从上午 9 点到下午 5 点的每个小时



### 问号 ( ? )

不指定值，用于代替 “ * ”，类似 “ _ ” 的存在，不难理解



## 重要函数

```none
cron.New()
```

会根据本地时间创建一个新（空白）的 Cron job runner

```none
c.AddFunc
```

向 Cron job runner 添加一个 func ，以按给定的时间表运行。首先解析时间表，如果填写有问题会直接 err，无误则将 func 添加到 Schedule 队列中等待执行

```none
c.Start()
```

在当前执行的程序中启动 Cron 调度程序



## 缺点

1.依赖当前服务运行环境的时间，如果时间被改了，会发生错乱。

2.它是单机的，线上服务一般是多实例，如果需要多示例运行，需要加上全局锁。