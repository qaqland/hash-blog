# Git 祖先跳表

2025年5月4日 介绍一下 Bushi 所使用的祖先跳表

## 目标

快速判断 Commit A 是否是 Commit B 的祖先

- git-ref-contains-commit
- git-log-path-filter

## 限制

本项目仅考虑提交的第一个 Parent Commit

## 原理

## 判断

## 验证

选择 Aports 仓库中 Commit-ID 分别为 100 和 20000 的两个提交，对应的 Commit-OID 如下

```sqlite
sqlite> SELECT * FROM commits WHERE commit_id = 100 OR commit_id = 20000;
commit_id  commit_hash                               commit_mark  parent_id  depth  repo_id
---------  ----------------------------------------  -----------  ---------  -----  -------
100        44a369d15ac69464584099d339a0e1ec1ec7fa66  100          99         99     1
20000      89d3e577e9a10d8f75b87d5ec44d801efbb7f3c7  20000        19999      13920  1
```

查看网页确实在同一分支中，两者 depth 之差为 13821，转为二进制数得到 2 的 N 次幂

```python
>>> bin(13821)
'0b11010111111101'
```

即为 13、12、10、8、7、6、5、4、3、2、0 这些序列，下面开始在祖先表中查找

```
sqlite> SELECT * FROM ancestors WHERE commit_id = 20000 AND level = 13;
commit_id  level  ancestor_id
---------  -----  -----------
20000      13     7618
sqlite> SELECT * FROM ancestors WHERE commit_id = 7618 AND level = 12;
commit_id  level  ancestor_id
---------  -----  -----------
7618       12     2133
sqlite> SELECT * FROM ancestors WHERE commit_id = 2133 AND level = 10;
commit_id  level  ancestor_id
---------  -----  -----------
2133       10     777
sqlite> SELECT * FROM ancestors WHERE commit_id = 777 AND level = 8;
commit_id  level  ancestor_id
---------  -----  -----------
777        8      372
sqlite> SELECT * FROM ancestors WHERE commit_id = 372 AND level = 7;
commit_id  level  ancestor_id
---------  -----  -----------
372        7      244
sqlite> SELECT * FROM ancestors WHERE commit_id = 244 AND level = 6;
commit_id  level  ancestor_id
---------  -----  -----------
244        6      161
sqlite> SELECT * FROM ancestors WHERE commit_id = 161 AND level = 5;
commit_id  level  ancestor_id
---------  -----  -----------
161        5      129
sqlite> SELECT * FROM ancestors WHERE commit_id = 129 AND level = 4;
commit_id  level  ancestor_id
---------  -----  -----------
129        4      113
sqlite> SELECT * FROM ancestors WHERE commit_id = 113 AND level = 3;
commit_id  level  ancestor_id
---------  -----  -----------
113        3      105
sqlite> SELECT * FROM ancestors WHERE commit_id = 105 AND level = 2;
commit_id  level  ancestor_id
---------  -----  -----------
105        2      101
sqlite> SELECT * FROM ancestors WHERE commit_id = 101 AND level = 0;
commit_id  level  ancestor_id
---------  -----  -----------
101        0      100
```

最终得到 Commit-ID 与目标 ID 相同，两者位于同一分支中。
