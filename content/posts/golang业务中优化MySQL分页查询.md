---
title: "golang业务中优化MySQL分页查询"
date: 2023-09-18T14:12:10+08:00
draft: fasle
categories:
    - gorm
    - golang
tags:
    - gorm
    - golang
    - mysql
    - 分页查询
---

当在golang业务中使用MySQL分页查询时，LIMIT子句的效率会随着数据量的增加而降低。在本文中，我将记录上次查询结束位置的方法来优化MySQL分页查询。

<!--more-->

## 背景

后端开发时，经常会遇到需要分页查询的场景，当使用**LIMIT**和**OFFSET**来进行分页查询时，MySQL需要跳过一定数量的行，这会在大数据集上导致性能问题。网上有不错的文章对其进行分析和优化，如[MySQL优化之超大分页查询](https://zhuanlan.zhihu.com/p/279863859)，但是本文的重点是在业务中通过代码实现**记录上次查询结束的位置**的优化方法。

## 需求分析

一张student表，需要根据外键：school_id进行分页查询，每页数据不固定，查询结果需要按照id升序排列。

## 思路

由于使用**LIMIT**和**OFFSET**来进行分页查询时，MySQL需要跳过一定数量的行，这会在大数据集上导致性能问题。因此，我们可以通过记录上次查询结束的位置来优化MySQL分页查询。考虑使用**迭代器模式**进行封装。

## 代码实现
### 迭代器接口
```go
type Iterator interface {
	Next() bool       // 是否还有下一个元素
	CurrentItem() any // 返回当前元素
	Error() error     // 返回错误信息
}
```

### 数据结构
```go
type Student struct {
    Id        int    `gorm:"column:id;type:int(11);primary_key;AUTO_INCREMENT" json:"id"`
    Name      string `gorm:"column:name;type:varchar(255);NOT NULL" json:"name"`
    SchoolId  int    `gorm:"column:school_id;type:int(11);NOT NULL" json:"school_id"`
    // 省略不重要字段
}

type StudentIterator struct {
	curIdx     int       // 当前索引
	limit      int       // 每页数量
	schoolId   int       // 学校id
	curPage    []Student // 当前页数据
	isFinished bool      // 是否已经查询完毕
	err        error     // 错误信息
}

// @param schoolId 学校id
// @param startIdx 起始索引
// @param limit 每页数量
// @return Iterator 迭代器
func NewStudentIterator(schoolId, startIdx, limit int) Iterator {
	return &StudentIterator{
		schoolId:   schoolId,
		curIdx:     startIdx,
		limit:      limit,
		isFinished: false,
	}
}
```

### db查询
这个函数是每次查询的核心，通过**curIdx**和**limit**来进行分页查询，查询结果按照id升序排列。
```go
func InfoStudentWithLimit(schoolId, idx, limit int) (ret []Student, err error) {
	ret = make([]Student, 0, limit)

	err = gdb.Where("id >= ?", idx).
		Where("school_id = ?", schoolId).
		Order("id").Limit(limit).Find(&ret).Error
	return
}
```

### 迭代器实现
当**Next**被调用时，首先检查**isFinished**是否为**true**，如果已经迭代完成，则返回**false**表示没有更多数据。

否则，它调用**InfoStudentWithLimit()**函数来获取下一页的学生数据，该函数会返回当前页学生数据和可能的错误。

如果出现错误，它将设置**err**并将**isFinished**设置为**true**，然后返回**false**。

如果获取到数据，它会检查当前页的数据数量是否小于**limit**或是否为空，如果是，则表示没有更多数据，将**isFinished**设置为**true**。

最后，更新**curIdx**为当前页最后一个学生的ID加1，以便下次调用**Next**时获取下一页。
```go
func (s *StudentIterator) Next() bool {
	if s.isFinished {
        s.curPage = nil
		return false
	}

	s.curPage, s.err = InfoStudentWithLimit(s.schoolId, s.curIdx, s.limit)
	if s.err != nil {
		s.isFinished = true
        s.curPage = nil
		return false
	}

	if len(s.curPage) < s.limit || len(s.curPage) == 0 {
		s.isFinished = true
	}

	s.curIdx = s.curPage[len(s.curPage)-1].Id + 1
	return true
}

func (s *StudentIterator) CurrentItem() any { return s.curPage }

func (s *StudentIterator) Error() error { return s.err }
```