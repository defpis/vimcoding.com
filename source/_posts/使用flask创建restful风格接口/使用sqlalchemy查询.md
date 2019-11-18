---
title: "4. 使用sqlalchemy查询"
date: 2019-11-09 00:00:00
categories:
- 使用flask创建restful风格接口
---

使用jwt认证时，我们通过flask_sqlalchemy简单的操作了user表：注册用户-插入一条数据到user表；用户认证-从数据库查询username相匹配的一条用户记录，然后验证密码。实际生产中，可能会有更多的需求：对结果排序，更复杂的过滤条件，多表联结查询等等。本节我们将一起探讨使用orm工具sqlalchemy和flask_sqlalchemy来实现复杂的数据库操作。

<!-- more -->

首先按照以下关系创建表，我们不使用外键关系约束，因为对于高性能的业务，使用关系在级联操作的时候可能会带来性能的损耗。
