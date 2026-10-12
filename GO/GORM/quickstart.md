```go
package main

import (
	"fmt"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

type Product struct {
	gorm.Model
	Code  string
	Price uint
}

func main() {
	db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
	if err != nil {
		panic("failed to connect database")
	}

	// 迁移 schema
	db.AutoMigrate(&Product{})

	// Create
	db.Create(&Product{Code: "D42", Price: 100})

	// Read
	var product Product
	db.First(&product, "code = ?", "D42") // 查找 code 字段值为 D42 的记录
	fmt.Println(product)
	// Update - 将 product 的 price 更新为 200
	db.Model(&product).Update("Price", 200)
	fmt.Println(product)
	// Update - 更新多个字段
	db.Model(&product).Updates(Product{Price: 200, Code: "F42"}) // 仅更新非零值字段
	db.Model(&product).Updates(map[string]interface{}{"Price": 200, "Code": "F42"})
	fmt.Println(product)
	// Delete - 删除 product
	db.Delete(&product, 1)

	fmt.Println("success")
}
```

output:

```shell
➜  myDemoRepo git:(feature) ✗ go run quickstart.go
{{4 2022-04-27 17:38:56.770739 +0800 +0800 2022-04-27 17:38:56.770739 +0800 +0800 {0001-01-01 00:00:00 +0000 UTC false}} D42 100}
{{4 2022-04-27 17:38:56.770739 +0800 +0800 2022-04-27 17:38:56.772481 +0800 CST {0001-01-01 00:00:00 +0000 UTC false}} D42 200}
{{4 2022-04-27 17:38:56.770739 +0800 +0800 2022-04-27 17:38:56.774322 +0800 CST {0001-01-01 00:00:00 +0000 UTC false}} F42 200}
success
```
