---
title: "MySQL删除重复记录的几种方法"
date: 2022-02-10T17:47:05+08:00
draft: false
categories: ["技术"]
tags: ["MySQL"]
---

## 问题
这里有这么一张学生信息表，表结构如下：
```mysql
CREATE TABLE `students`  (
  `id` int UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` varchar(30) NOT NULL,
  `gender` tinyint(1) UNSIGNED NOT NULL,
  `age` int UNSIGNED NOT NULL,
  PRIMARY KEY (`id`)
);
```
现`name`,`gender`,`age`三个字段均相同，则可认定这是同一个学生；但是由于表没有加联合唯一索引，所以表里面有很多重复的记录，怎么样使相同的记录只保留一条？

## 解答

若是查询，我们大可用`group by`然后`having count`或者`distinct`来合并查询一条，但是删除好似没有这样方便的语句。所以总结了一些比较好的方法：

### 创建联合唯一索引

这是最简单方便的一条了
```mysql
ALTER IGNORE TABLE students ADD UNIQUE INDEX idx_name_gender_age (name, gender, age);
```
注意这个`IGNORE`，由于表里面已经有重复记录了，所以直接创建会出现错误。执行之后，所有的重复记录都会被自动删除（只保留一条），而且以后也无法插入重复记录了。

### 自关联

如果你没有修改数据库结构的权限，那下面这条肯定适合你
```mysql
DELETE `a`
FROM
    `students` AS `a`,
    `students` AS `b`
WHERE
    -- IMPORTANT: Ensures one version remains
    -- Change "ID" to your unique column's name
    `a`.`id` > `b`.`id`

    -- Any duplicates you want to check for
    AND (`a`.`name` = `b`.`name` OR `a`.`name` IS NULL AND `b`.`name` IS NULL)
    AND (`a`.`gender` = `b`.`gender` OR `a`.`gender` IS NULL AND `b`.`gender` IS NULL)
    AND (`a`.`age` = `b`.`age` OR `a`.`age` IS NULL AND `b`.`age` IS NULL);
```
这样，就会只保留id最小的那一条记录了。

### 复制表

创建一个表结构完全一样的新表`students_copy`，取出原表中不重复的记录，塞入这个新表中，将旧表改名或删除，将新表改为`students`。


### 总结

能改表结构的话我选第一种，不能的话我选第二种😅，不管哪一种，操作前务必做好数据备份🤣。
