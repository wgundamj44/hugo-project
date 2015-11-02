---
layout: post
title: NULL or NOT NULL
category: [tech]
tags: [mysql]
---

In mysql table definiations, how NULL and NOT NULL effects behavior?

**AFTER** MySQL 5.0.2. If there's no NOT NULL in the column defination, MySQL add a DEFAULT NULL to defination. If there's NOT NULL in the defination, in strict mode, an insertion that lacks the column will raise an error, while implict default value will be inserted in non-strict mode.


So for portability, always add DEFAULT for NOT NULL column.

Detail can be found [here](http://dev.mysql.com/doc/refman/5.0/en/data-type-defaults.html).
