AutoMigrate 用于自动迁移您的 schema，保持您的 schema 是最新的。

注意： AutoMigrate 会创建表、缺失的外键、约束、列和索引。 如果大小、精度、是否为空可以更改，则 AutoMigrate 会改变列的类型。 出于保护您数据的目的，它不会删除未使用的列

```
db.AutoMigrate(&User{})

db.AutoMigrate(&User{}, &Product{}, &Order{})

// 创建表时添加后缀
db.Set("gorm:table_options", "ENGINE=InnoDB").AutoMigrate(&User{})
```

注意 AutoMigrate 会自动创建数据库外键约束，您可以在初始化时禁用此功能，例如：

```
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  DisableForeignKeyConstraintWhenMigrating: true,
})
```
