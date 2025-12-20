# Git 提交代次

## 期望

为每个 Commit 保存它的 generation 信息到数据库，
代次（深度）信息是「祖先跳表」方法的基础。

## 做法

设置根提交的 generation 为 0，后续子代依次 +1

```sql
UPDATE
    commits
SET
    generation = parent.generation + 1
FROM
    commits AS parent
WHERE
    commits.generation IS NULL
    AND parent.generation IS NOT NULL
    AND commits.repository_id = parent.repository_id
    AND commits.parent_hash = parent.commit_hash
;

CREATE INDEX IF NOT EXISTS idx_commits_null_generation
    ON commits(repository_id, parent_hash)
    WHERE generation IS NULL;
CREATE INDEX IF NOT EXISTS idx_commits_parent_lookup
    ON commits(repository_id, commit_hash, generation)
    WHERE generation IS NOT NULL;
```

## 循环

上述 SQL 只能更新一个层面的 Commit，因此丢在 do-while 循环中反复执行

```c
int rows_affected = 0;
sqlite3_stmt *stmt = stmts[STMT_UPDATE_GENERATION];
do {
    sqlite3_exec(connection, "BEGIN TRANSACTION;", NULL, NULL, NULL);

    sqlite3_reset(stmt);
    sqlite3_step(stmt);
    rows_affected = sqlite3_changes(connection);

    sqlite3_exec(connection, "COMMIT;", NULL, NULL, NULL);
} while (rows_affected > 0);
```

## 障碍

速度还是太慢了，即使有（缺省）索引依然需要全表扫描。

```
$ cat update-generation.sql | sqlite3 stage.db
QUERY PLAN
|--SCAN commits USING INDEX idx_commits_null_generation
`--SEARCH parent USING COVERING INDEX idx_commits_parent_lookup (repository_id=? AND commit_hash=? AND generation>?)
```

对于有 26 万个提交的 aports 测试用例来说，循环完成需要 4 个多小时。

## 投降

之前在检测 Commit 会修改哪些文件的时候已经投降过一次了，现在只能继续投降。

抱住 Git 二进制的大腿，尝试逆序输出解析。

```shell
$ git log --pretty=format:%n%H --name-only --first-parent --reverse
```

- `%H` 显示完整 CommitHash
- `%n` 显示一个 `\n`
- `name-only` 输出当前提交涉及修改的文件名
- `first-parent` 单线遍历
- `reverse` 逆序输出

## 成本

投降之前评估一下成本，理论上来说逆序应该更耗时，因为正序是 FIFO，逆序需要状态。
但是实际测试区别不明显，甚至逆序耗时更短？

```
$ hyperfine "git log --pretty=format:%H --name-only --first-parent" \
            "git log --pretty=format:%H --name-only --first-parent --reverse"
Benchmark 1: git log --pretty=format:%H --name-only --first-parent
  Time (mean ± σ):     233.308 s ±  1.055 s    [User: 209.537 s, System: 22.922 s]
  Range (min … max):   232.254 s … 235.719 s    10 runs

Benchmark 2: git log --pretty=format:%H --name-only --first-parent --reverse
  Time (mean ± σ):     232.894 s ±  0.565 s    [User: 208.993 s, System: 23.075 s]
  Range (min … max):   232.023 s … 233.930 s    10 runs

Summary
  git log --pretty=format:%H --name-only --first-parent --reverse ran
    1.00 ± 0.01 times faster than git log --pretty=format:%H --name-only --first-parent
```

```
$ /usr/bin/time -v git log --pretty=format:%H --name-only --first-parent > /dev/null
	Command being timed: "git log --pretty=format:%H --name-only --first-parent"
	User time (seconds): 208.61
	System time (seconds): 22.89
	Percent of CPU this job got: 99%                <<< 单线程进程，CPU 满载
	Elapsed (wall clock) time (h:mm:ss or m:ss): 3m 52.31s
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 669828      <<< 最大内存占用 600M
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 12839624
	Voluntary context switches: 1
	Involuntary context switches: 367               <<< 别的都看不懂
	Swaps: 0
	File system inputs: 0
	File system outputs: 0
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

```
$ /usr/bin/time -v git log --pretty=format:%H --name-only --first-parent --reverse > /dev/null
	Command being timed: "git log --pretty=format:%H --name-only --first-parent --reverse"
	User time (seconds): 210.01
	System time (seconds): 23.14
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 3m 53.98s
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 612564
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 12762389
	Voluntary context switches: 2
	Involuntary context switches: 475
	Swaps: 0
	File system inputs: 0
	File system outputs: 0
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

## 结果

```
qaq^n5105 bushi/bushi-index main*
$ ./build/bushi-index -t stage.db -p ~/aports/.git > log

qaq^n5105 bushi/bushi-index main* 4m52s
```

不到 5 分钟！满意，收工！

## 链接

- <https://git-scm.com/docs/git-log>
- <https://sqlite.org/partialindex.html>
- <https://github.com/sharkdp/hyperfine>

