---
layout:     post
title:      "drop、delete与truncate的区别"
subtitle:   ""
date:       2019-05-22
author:     "CHuiL"
header-img: "img/database-bg.png"
tags:
    - 数据库
---

### Delete 与 Truncate
- Delete与Truncate都是删除表中的数据
- Delete执行后会在事务日志中保存记录以便回滚，而Truncate则不会记录。所以是不能恢复的
- 表和索引所占空间。当表被Truncate后，`这个表和索引所占用的空间会恢复到初始大小`。而Delete操作不会减少表或索引所占用的空间
- `Truncate与不带where的delete：都是删除表中所有的行数据，不删除表中的结构。使用Truncate后自增序列将重置，索引也会重新设置。`
- `Truncate 表名速度快，效率高`，使用系统和事务日志资源少。Delete每删除一行就要记录一项。Truncate通过释放存储表数据所用的数据页来删除数据，并且只在事务日志中记录页的释放。

### Drop
- 删除表结构，包括所有数据。
- 立即执行，不能回滚。
- 将删除表的约束，触发器，索引。
 
### 总结
- 速度上：drop> truncate > delete。
- 在使用drop和truncate时一定要注意，虽然可以恢复，但为了减少麻烦，还是要慎重。