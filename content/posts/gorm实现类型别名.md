---
title: "Gorm实现类型别名"
date: 2023-06-09T21:28:07+08:00
draft: false
categories:
    - gorm
    - golang
tags:
    - gorm
    - golang
---

一个项目中使用到了gorm，但是gorm的类型别名功能在项目中并没有使用，所以在这里记录一下。

<!--more-->

## 业务描述

数据库中存在一个类似这样的表：
``` Go
type User struct {

    /***************省略非必要字段*************/

	CardId      string `db:"card_id"`   // 身份证号 
    CardType    int    `db:"card_type"` // 证件类型

    /***************省略非必要字段*************/
}
```
业务中有这样的需求：根据证件类型做不同的处理，我很自然地就想到给CardType绑定各种方法，来获取证件类型等。为了实现他，我对CardType进行了封装。

## 封装

``` Go
type CardType int

const (
	China CardType = iota
	HongKong
	Macao
	Taiwan
	Expatriate
	Err
)
```
如此可以为CardType类型添加方法，实现业务需求。

## 存在的问题
由于使用了类型别名，对于gorm来说，可能是一个全新的自定义类型，可能无法自动将数据映射到数据库中（没试过，也许会报错吧？）。

通过查阅[文档](https://gorm.io/zh_CN/docs/data_types.html)，了解到gorm是支持自定义类型的。需要实现gorm的Scanner和Valuer接口。
``` Go
func (c *CardType) Value() (driver.Value, error) {
	return int(*c), nil
}

func (c *CardType) Scan(value interface{}) (err error) {
	if value == nil {
		return nil
	}

	var intValue int
	switch value.(type) {
	case int64:
		intValue = int(value.(int64))
	case []byte:
		intValue, err = strconv.Atoi(string(value.([]byte)))
		if err != nil {
			return err
		}
	case string:
		intValue, err = strconv.Atoi(value.(string))
	case float64:
		intValue = int(value.(float64))
	default:
		return errors.New(fmt.Sprint("Failed to unmarshal CardType value:", value))
	}

	*c = CardType(intValue)
	return nil
}
```
为了兼容更多的数据库，用switch来枚举数据库可能返回的类型，再转成int。“聪明”的Chat同学跟我说MySQL返回的是[]byte😅（实际上是int64），我能理解，这个被大家追捧的人工智能也不是第一次正经地在我面前胡说了。

经过测试，没有问题。