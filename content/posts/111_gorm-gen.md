---
title: "GEN 自动生成 GORM 模型结构体文件及使用示例"
date: 2022-09-16T21:12:00+08:00 
draft: false 
tags: ["Golang", "GORM", "GEN"]
series: ["Golang"]
categories: ["后端"]
---


## 背景

GEN 是一个基于 GORM 的安全 ORM 框架, 由字节跳动无恒实验室与 GORM 作者联合研发，主要功能说白了就是帮助生成数据表对应的模型文件和更安全方便地执行SQL。

> 直接使用 GORM 与 GEN 工具的对比：
>
> | 直接使用GORM                                                                                    | 使用GEN                                                  |
> | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
> | 需手动创建与数据表各列一一对应的结构体                                                          | 指定表名后自动读取并生成对应结构体                       |
> | 需手动实现具体的 go 代码查询逻辑                                                                | 描述 SQL 查询逻辑即可，工具自动转换成安全稳定的代码      |
> | 查询接口十分灵活，但不能保持查询的 SQL 不发生语法错误，<br />只能通过测试保证部分场景的正常运行 | 查询接口使用类型安全，编译可通过，查询逻辑即是正常合理的 |
> | 需人工评经验保证业务不存在安全问题，<br />一旦出错往往在上线前才能发现，影响上线流程            | 提供的安全可靠的查询 API，开发时能用的就是安全的         |

## 本文目标

GEN 提供的功能是强大而丰富的，本文相当一个上手指引只挑些常见操作进行示例说明：

* 表字段的整型类型无论大小和有无符号，结构体中统一使用 int64 （为了简化后续使用时的操作）。
* 个别结构体字段 json 序列化时由数字类型转成字符串类型，如：余额字段 `balance` 在表中是 DECIMAL 类型，结构体中是 int64，业务需要的 json 字段类型为字符串。
* 使用非默认字段名实现自动更新、创建时间戳和软删除。
* 模型关联

示例环境：

* go 1.18
* gen v0.3.16  注意:当前GEN的最新版主版本是0, 也就是 API 在未来可能会有较大的变更, 使用时务必注意版本变更问题
* MySQL 8.0

目标表有3个,分别是 `user`、`address`和 `hobby`，`user`与 `address`是一对多关系，如下所示:

```sql
CREATE TABLE IF NOT EXISTS `user` (
    `id` int unsigned NOT NULL AUTO_INCREMENT COMMENT 'ID',
    `name` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '' COMMENT '用户名',
    `age` tinyint unsigned NOT NULL DEFAULT '0' COMMENT '年龄',
    `balance` decimal(11,2) unsigned NOT NULL DEFAULT '0.00' COMMENT '余额',
    `updated_at` datetime NOT NULL COMMENT '更新时间',
    `created_at` datetime NOT NULL COMMENT '创建时间',
    `deleted_at` datetime DEFAULT NULL COMMENT '删除时间',
    PRIMARY KEY (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;

CREATE TABLE IF NOT EXISTS `address` (
    `id` int unsigned NOT NULL AUTO_INCREMENT,
    `uid` int unsigned NOT NULL,
    `province` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
    `city` varchar(20) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
    `update_time` int unsigned NOT NULL,
    `create_time` int unsigned NOT NULL,
    `delete_time` int unsigned NOT NULL DEFAULT '0',
    PRIMARY KEY (`id`) USING BTREE,
    KEY `uid` (`uid`)
    ) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=DYNAMIC;

CREATE TABLE IF NOT EXISTS `hobby` (
    `id` int unsigned NOT NULL AUTO_INCREMENT,
    `name` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
    `updated_at` int unsigned NOT NULL,
    `created_at` int unsigned NOT NULL,
    `deleted_at` int unsigned NOT NULL DEFAULT '0',
    PRIMARY KEY (`id`) USING BTREE
    ) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci ROW_FORMAT=DYNAMIC;

```

## 配置 GEN 并生成模型结构体

创建并初始化项目,再引入 GEN 包

```shell
mkdir gormgendemo && cd gormgendemo
go mod init
go get -u gorm.io/gen@v0.3.16
```

继续创建用于生成模型文件的 `./cmd/generate/main.go`文件和用于执行 SQL 示例的 `./main.go`文件，目录如下所示：

```.
├── cmd
│   └── generate
│       └── main.go
├── go.mod
├── go.sum
└── main.go
```

`./cmd/generate/main.go`代码：

```go
package main

import (
	"fmt"
	"strings"

	"gorm.io/driver/mysql"
	"gorm.io/gen"
	"gorm.io/gen/field"
	"gorm.io/gorm"
)

const MySQLDSN = "root:123456@(localhost:3306)/gormgendemo?charset=utf8mb4&parseTime=True&loc=Local"

func main() {

	// 连接数据库
	db, err := gorm.Open(mysql.Open(MySQLDSN))
	if err != nil {
		panic(fmt.Errorf("cannot establish db connection: %w", err))
	}

	// 生成实例
	g := gen.NewGenerator(gen.Config{
		// 相对执行`go run`时的路径, 会自动创建目录
		OutPath: "dal/query",

		// WithDefaultQuery 生成默认查询结构体(作为全局变量使用), 即`Q`结构体和其字段(各表模型)
		// WithoutContext 生成没有context调用限制的代码供查询
		// WithQueryInterface 生成interface形式的查询代码(可导出), 如`Where()`方法返回的就是一个可导出的接口类型
		Mode: gen.WithDefaultQuery | gen.WithQueryInterface,

		// 表字段可为 null 值时, 对应结体字段使用指针类型
		FieldNullable: true, // generate pointer when field is nullable

		// 表字段默认值与模型结构体字段零值不一致的字段, 在插入数据时需要赋值该字段值为零值的, 结构体字段须是指针类型才能成功, 即`FieldCoverable:true`配置下生成的结构体字段.
		// 因为在插入时遇到字段为零值的会被GORM赋予默认值. 如字段`age`表默认值为10, 即使你显式设置为0最后也会被GORM设为10提交.
		// 如果该字段没有上面提到的插入时赋零值的特殊需要, 则字段为非指针类型使用起来会比较方便.
		FieldCoverable: false, // generate pointer when field has default value, to fix problem zero value cannot be assign: https://gorm.io/docs/create.html#Default-Values

		// 模型结构体字段的数字类型的符号表示是否与表字段的一致, `false`指示都用有符号类型
		FieldSignable: false, // detect integer field's unsigned type, adjust generated data type
		// 生成 gorm 标签的字段索引属性
		FieldWithIndexTag: false, // generate with gorm index tag
		// 生成 gorm 标签的字段类型属性
		FieldWithTypeTag: true, // generate with gorm column type tag
	})
	// 设置目标 db
	g.UseDB(db)

	// 自定义字段的数据类型
	// 统一数字类型为int64,兼容protobuf
	dataMap := map[string]func(detailType string) (dataType string){
		"tinyint":   func(detailType string) (dataType string) { return "int64" },
		"smallint":  func(detailType string) (dataType string) { return "int64" },
		"mediumint": func(detailType string) (dataType string) { return "int64" },
		"bigint":    func(detailType string) (dataType string) { return "int64" },
		"int":       func(detailType string) (dataType string) { return "int64" },
	}
	// 要先于`ApplyBasic`执行
	g.WithDataTypeMap(dataMap)

	// 自定义模型结体字段的标签
	// 将特定字段名的 json 标签加上`string`属性,即 MarshalJSON 时该字段由数字类型转成字符串类型
	jsonField := gen.FieldJSONTagWithNS(func(columnName string) (tagContent string) {
		toStringField := `balance, `
		if strings.Contains(toStringField, columnName) {
			return columnName + ",string"
		}
		return columnName
	})
	// 将非默认字段名的字段定义为自动时间戳和软删除字段;
	// 自动时间戳默认字段名为:`updated_at`、`created_at, 表字段数据类型为: INT 或 DATETIME
	// 软删除默认字段名为:`deleted_at`, 表字段数据类型为: DATETIME
	autoUpdateTimeField := gen.FieldGORMTag("update_time", "column:update_time;type:int unsigned;autoUpdateTime")
	autoCreateTimeField := gen.FieldGORMTag("create_time", "column:create_time;type:int unsigned;autoCreateTime")
	softDeleteField := gen.FieldType("delete_time", "soft_delete.DeletedAt")
	// 模型自定义选项组
	fieldOpts := []gen.ModelOpt{jsonField, autoCreateTimeField, autoUpdateTimeField, softDeleteField}

	// 创建模型的结构体,生成文件在 model 目录; 先创建的结果会被后面创建的覆盖
	// 这里创建个别模型仅仅是为了拿到`*generate.QueryStructMeta`类型对象用于后面的模型关联操作中
	Address := g.GenerateModel("address")
	// 创建全部模型文件, 并覆盖前面创建的同名模型
	allModel := g.GenerateAllTable(fieldOpts...)

	// 创建有关联关系的模型文件
	User := g.GenerateModel("user",
		append(
			fieldOpts,
			// user 一对多 address 关联, 外键`uid`在 address 表中
			gen.FieldRelate(field.HasMany, "Address", Address, &field.RelateConfig{GORMTag: "foreignKey:UID"}),
		)...,
	)
	Address = g.GenerateModel("address",
		append(
			fieldOpts,
			gen.FieldRelate(field.BelongsTo, "User", User, &field.RelateConfig{GORMTag: "foreignKey:UID"}),
		)...,
	)

	// 创建模型的方法,生成文件在 query 目录; 先创建结果不会被后创建的覆盖
	g.ApplyBasic(User, Address)
	g.ApplyBasic(allModel...)

	g.Execute()
}

```

执行生成程序:

```shell
go mod tidy
go run ./cmd/generate/main.go
```

结果目录:

```
.
├── cmd
│   └── generate
│       └── main.go
├── dal
│   ├── model
│   │   ├── address.gen.go
│   │   ├── hobby.gen.go
│   │   └── user.gen.go
│   └── query
│       ├── address.gen.go
│       ├── gen.go
│       ├── hobby.gen.go
│       └── user.gen.go
├── go.mod
├── go.sum
└── main.go
```

生成的模型结构体:

```go
// User mapped from table <user>
type User struct {
    ID        int64          `gorm:"column:id;type:int unsigned;primaryKey;autoIncrement:true" json:"id"`                    // ID
    Name      string         `gorm:"column:name;type:varchar(20);not null" json:"name"`                                      // 用户名
    Age       int64          `gorm:"column:age;type:tinyint unsigned;not null" json:"age"`                                   // 年龄
    Balance   float64        `gorm:"column:balance;type:decimal(11,2) unsigned;not null;default:0.00" json:"balance,string"` // 余额
    UpdatedAt time.Time      `gorm:"column:updated_at;type:datetime;not null" json:"updated_at"`                             // 更新时间
    CreatedAt time.Time      `gorm:"column:created_at;type:datetime;not null" json:"created_at"`                             // 创建时间
    DeletedAt gorm.DeletedAt `gorm:"column:deleted_at;type:datetime" json:"deleted_at"`                                      // 删除时间
    Address   []Address      `gorm:"foreignKey:UID" json:"address"`
}

// Address mapped from table <address>
type Address struct {
	ID         int64                 `gorm:"column:id;type:int unsigned;primaryKey;autoIncrement:true" json:"id"`
	UID        int64                 `gorm:"column:uid;type:int unsigned;not null" json:"uid"`
	Province   string                `gorm:"column:province;type:varchar(20);not null" json:"province"`
	City       string                `gorm:"column:city;type:varchar(20);not null" json:"city"`
	UpdateTime int64                 `gorm:"column:update_time;type:int unsigned;autoUpdateTime" json:"update_time"`
	CreateTime int64                 `gorm:"column:create_time;type:int unsigned;autoCreateTime" json:"create_time"`
	DeleteTime soft_delete.DeletedAt `gorm:"column:delete_time;type:int unsigned;not null" json:"delete_time"`
	User       User                  `gorm:"foreignKey:UID" json:"user"`
}

// Hobby mapped from table <hobby>
type Hobby struct {
    ID        int64  `gorm:"column:id;type:int unsigned;primaryKey;autoIncrement:true" json:"id"`
    Name      string `gorm:"column:name;type:varchar(20);not null" json:"name"`
    UpdatedAt int64  `gorm:"column:updated_at;type:int unsigned;not null" json:"updated_at"`
    CreatedAt int64  `gorm:"column:created_at;type:int unsigned;not null" json:"created_at"`
    DeletedAt int64  `gorm:"column:deleted_at;type:int unsigned;not null" json:"deleted_at"`
}
```

上面3个模型都实现了更新、创建字段的自动时间戳功能，前面2个实现了软删除功能，表字段的整型类型都以 int64表示。

## 使用 GEN 进行数据库操作

`./main.go`代码：

```go
package main

import (
	"context"
	"fmt"
	"log"

	"gormgendemo/dal/model"
	"gormgendemo/dal/query"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

const MySQLDSN = "root:123456@(localhost:3306)/gormgendemo?charset=utf8mb4&parseTime=True&loc=Local"

func main() {

	// 连接数据库
	db, err := gorm.Open(mysql.Open(MySQLDSN))
	if err != nil {
		panic(fmt.Errorf("cannot establish db connection: %w", err))
	}

	query.SetDefault(db)
	q := query.Q
	ctx := context.Background()
	//qc := q.WithContext(ctx)

	// 增
	insert(ctx, q)
	// 删
	del(ctx, q)
	// 改
	update(ctx, q)
	// 查
	find(ctx, q)

	fmt.Println("Done!")
}

func insert(ctx context.Context, q *query.Query) {
	qc := q.WithContext(ctx)

	// 插入数据
	users := []*model.User{
		{
			Name: "张三",
			Age:  30,
			Address: []model.Address{
				{Province: "广东", City: "广州"},
				{Province: "广东", City: "深圳"},
			},
		},
		{
			Name: "李四",
			Age:  40,
			Address: []model.Address{
				{Province: "广东", City: "广州"},
				{Province: "广东", City: "深圳"},
			},
		},
		{
			Name: "王五",
			Age:  50,
		},
	}
	hobby := []*model.Hobby{
		{Name: "看电影"}, {Name: "看书"}, {Name: "跑步"},
	}

	err := qc.User.Create(users...)
	if err != nil {
		log.Fatal(err)
	}
	err = qc.Hobby.Create(hobby...)
	if err != nil {
		log.Fatal(err)
	}
}

func del(ctx context.Context, q *query.Query) {
	qc := q.WithContext(ctx)

	qc.User.Where(q.User.Name.Eq("张三")).Delete()       // 软删
	qc.Address.Where(q.Address.City.Eq("深圳")).Delete() // 软删
	qc.Hobby.Where(q.Hobby.Name.Eq("看电影")).Delete()    // 非软删
}

func update(ctx context.Context, q *query.Query) {
	qc := q.WithContext(ctx)

	qc.User.Where(q.User.Name.Eq("李四")).UpdateSimple(q.User.Age.Add(1))
}

func find(ctx context.Context, q *query.Query) {
	qc := q.WithContext(ctx)

	// First、Take、Last 方法, 如果没有找到记录则返回错误 ErrRecordNotFound。

	_, err := qc.User.Preload(q.User.Address).Where(q.User.Name.Eq("张三")).First()
	if err != nil {
		fmt.Printf("%+v \n", err)
		//record not found
	}
	user, err := qc.User.Preload(q.User.Address).Where(q.User.Name.Eq("李四")).First()
	if err == nil {
		fmt.Printf("%+v \n", *user)
		//{ID:2 Name:李四 Age:41 Balance:0 UpdatedAt:2022-09-16 15:13:39 +0800 CST CreatedAt:2022-09-16 15:13:39 +0800 CST
		//DeletedAt:{Time:0001-01-01 00:00:00 +0000 UTC Valid:false}
		//Address:[{ID:3 UID:2rovince:广东 City:广州 UpdateTime:1663312418 CreateTime:1663312418 DeleteTime:0
		//User:{ID:0 Name: Age:0 Balance:0 UpdatedAt:0001-01-01 00:00:00 +0000 UTC CreatedAt:0001-01-01 00:00:00 +0000 UTC
		//CreatedAt:{Time:0001-01-01 00:00:00 +0000 UTC Valid:false} Address:[]}}]}
	}

	addr, err := qc.Address.Preload(q.Address.User).Offset(1).First()
	if err == nil {
		fmt.Printf("%+v \n", *addr)
		//{ID:3 UID:2 Province:广东 City:广州 UpdateTime:1663312418 CreateTime:1663312418 DeleteTime:0
		//User:{ID:2 Name:李四 Age:41 Balance:0 UpdatedAt:2022-09-16 15:13:39 +0800 CST CreatedAt:2022-09-16 39 +0800 CST
		//DeletedAt:{Time:0001-01-01 00:00:00 +0000 UTC Valid:false} Address:[]}}
	}

	_, err = qc.Hobby.Where(q.Hobby.Name.Eq("看电影")).First()
	if err != nil {
		fmt.Printf("%+v \n", err)
		//record not found
	}
	hb, err := qc.Hobby.Where(q.Hobby.Name.Eq("看书")).First()
	if err == nil {
		fmt.Printf("%+v \n", *hb)
		//{ID:2 Name:看书 UpdatedAt:1663312418 CreatedAt:1663312418 DeletedAt:0}
	}

}

```

执行使用程序可以看到效果

```shell
go run main.go
```

## 结语

示例的相关代码已上传[github仓库](https://github.com/jeffid/gormgendemo)，有需要的可以自取。如果文章中有什么错漏或改善的地方也欢迎给我留言交流。

上面示例仅作抛砖引玉之用，更多使用方法还得参考官方文档和源码。

最后在这里对 GORM 和 GEN 的开发者们给我们带来了这么好用的工具表示感谢！

## 参考

[GORM/GEN中文文档](https://github.com/go-gorm/gen/blob/master/README.ZH_CN.md)
[无恒实验室联合GORM推出安全好用的ORM框架-GEN](https://www.modb.pro/db/156460)
