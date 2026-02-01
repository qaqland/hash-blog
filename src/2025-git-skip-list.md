# Git 祖先跳表

- 2025-05-04 介绍一下 bushi 所使用的祖先跳表
- 2025-12-19 更新了表结构，刷新字符串并补充

## 场景

在 GitWeb 中指定 Commit 的详情页面显示在远端的存在情况

- Branches containing commit
- Tags containing commit

这种请求需要判断 Reference 对应的 Commit 是否可达指定 Commit

## 限制

仅考虑 Commit 的第一个 Parent Commit，否则从链表退化到树，复杂程度翻倍

> [!NOTE]
> With the --changed-paths option, compute and write information about the
> paths changed between a commit and its first parent.
>
> 原版 git-commit-graph 同样设计如此

## 原理

如果什么都不做，判断只有两步：

1. 对比 generation 是否合理
2. 向上「依次」遍历到 generation 相等，检查是否相遇

对于可能超过 10 万提交的成熟项目，CTE 的耗时到秒级。
如果有 100 个分支，请求处理时间超出 1 分钟。

bushi 在表中以跳表形式额外存储了每个 Commit 的「祖先」信息，新判断为：

1. 对比 generation 是否合理，计算得到差值
2. 对 generation 的差值转二进制 BIN，1 跳 0 不跳，得到跳表节点
3. 向上按照「跳表」遍历到 generation 相等，检查是否相遇

时间复杂度从 `O(n)` 降低到了 `O(log n)`

## 验证

选择 Alpine Linux 的软件包构建脚本仓库深度距离为 100K 的两个提交，对应的 Commit 如下

```
sqlite> SELECT * FROM commits WHERE generation in (100, 100000);
commit_id  commit_hash                               parent_hash                               generation  repository_id
---------  ----------------------------------------  ----------------------------------------  ----------  -------------
237494     3845839a16f3162c2362e9271f59fe52cef7bf83  44a369d15ac69464584099d339a0e1ec1ec7fa66  100         1
137594     73a0fc8c219239f2df973722cf1bd75ce9aa1bf7  d435959ada011bdf44a535aa1297ad86d0f0f235  100000      1
```

两者 generation 之差转为二进制数得到 2 的 N 次幂

```python
>>> bin(99900)
'0b11000011000111100'
```

顺着 BIN 的二进制数，位数表示 exponent，1 跳 0 不跳

```
16  15                  10  9               5   4   3   2
1   1   0   0   0   0   1   1   0   0   0   1   1   1   1   0   0
```

假设有下面的 STMT 然后以绑定参数的形式执行

```sql
SELECT
    ancestors.commit_id,
    exponent,
    ancestor_id,
    generation
FROM
    ancestors
JOIN
    commits
ON
    commits.commit_id = ancestors.commit_id
WHERE
    commits.commit_id = ?1
    AND exponent = ?2;
```

从左跳也是一样的，但是右边开始可能有利于 SQLite 缓存命中

```
STMT(137594,  2) => 137598
STMT(137598,  3) => 137606
STMT(137606,  4) => 137622
STMT(137622,  5) => 137654
STMT(137654,  9) => 138166
STMT(138166, 10) => 139190
STMT(139190, 15) => 171958
STMT(171958, 16) => 237494
```

最终得到 `commit_id` 与目标相同，两者位于同一分支中。

## 友商

GitLab 家的 Gitaly 提供了 RPC-Git 接口 ListBranchNamesContainingCommit

> ListBranchNamesContainingCommit finds all branches under `refs/heads/` that
> contain the specified commit. The response is streamed back to the client to
> divide the list of branches into chunks.

不过只有对 Git 的封装

```go
type containingRequest interface {
	GetCommitId() string
	GetLimit() uint32
}


func containingArgs(...) []string {
	args := []string{fmt.Sprintf("--contains=%s", req.GetCommitId())}
	if limit := req.GetLimit(); limit != 0 {
		args = append(args, fmt.Sprintf("--count=%d", limit))
	}
	return args
}

func listRefNames(...) error {
	flags := []gitcmd.Option{
		gitcmd.Flag{Name: "--format=%(refname)"},
	}

	for _, arg := range extraArgs {
		flags = append(flags, gitcmd.Flag{Name: arg})
	}

	cmd, err := repo.Exec(ctx, gitcmd.Command{
		Name:  "for-each-ref",
		Flags: flags,
		Args:  []string{prefix},
	}, gitcmd.WithSetupStdout())
	...
}
```

- <https://gitlab.com/gitlab-org/gitaly/-/issues/1744>
- <https://gitlab.com/gitlab-org/gitaly/-/merge_requests/537>
- <https://gitlab.com/gitlab-org/gitaly/-/blob/v18.7.0/internal/gitaly/service/ref/refnames_containing.go>

再去看 Gitea 家，也是 Git 命令的封装

```go
func (repo *Repository) ListOccurrences(ctx context.Context, refType, commitSHA string) ([]string, error) {
	cmd := gitcmd.NewCommand()
	switch refType {
	case "branch":
		cmd.AddArguments("branch")
	case "tag":
		cmd.AddArguments("tag")
	}
	stdout, _, err := cmd.AddArguments("--no-color", "--sort=-creatordate", "--contains").AddDynamicArguments(commitSHA).RunStdString(ctx, &gitcmd.RunOpts{Dir: repo.Path})
	...
}
```

- <https://github.com/go-gitea/gitea/blob/v1.25.3/modules/git/repo_ref.go>

## 实现

我们使用的是根正苗红的 SQLite 数据，因此只需要根据表结构放置触发器即可

```sql
CREATE TABLE IF NOT EXISTS commits(
    commit_id     INTEGER PRIMARY KEY AUTOINCREMENT,
    commit_hash   TEXT NOT NULL,
    parent_hash   TEXT,               -- only first parent
    generation    INTEGER,            -- NOT NULL after stage2
    repository_id INTEGER NOT NULL
) STRICT;

-- 索引略

CREATE TABLE IF NOT EXISTS ancestors(
    commit_id   INTEGER NOT NULL,
    exponent    INTEGER NOT NULL,     -- 2^n generation
    ancestor_id INTEGER NOT NULL,     -- aka. commit_id
    PRIMARY KEY(commit_id, exponent)
) WITHOUT ROWID, STRICT;

CREATE TRIGGER IF NOT EXISTS tgr_ancestor
AFTER UPDATE OF generation ON commits
FOR EACH ROW
WHEN NEW.parent_hash IS NOT NULL
BEGIN
    -- contents
END;
```

触发器绑定在 commits 表中的 generation 字段中，
因为这个字段的更新意味着父代已填充完毕、数据完整。

```sql
INSERT INTO ancestors(commit_id, exponent, ancestor_id)

WITH RECURSIVE skip_list_cte(commit_id, exponent, ancestor_id) AS(
    SELECT
        NEW.commit_id,
        0 AS exponent,              -- 亲爹记录是从 commits 表直接查来的
        c.commit_id AS ancestor_id
    FROM
        commits AS c
    WHERE
        repository_id = NEW.repository_id
        AND commit_hash = NEW.parent_hash

    UNION ALL                       -- 标准 CTE 语法、没有爆炸风险

    SELECT
        s.commit_id,
        s.exponent + 1,             -- 后续记录是把亲爹的记录复制，辈分 +1
        a.ancestor_id
    FROM
        skip_list_cte AS s
    INNER JOIN
        ancestors AS a
    ON
        a.commit_id = s.ancestor_id
        AND a.exponent = s.exponent
)

SELECT
    commit_id, exponent, ancestor_id
FROM
    skip_list_cte
WHERE
    ancestor_id IS NOT NULL;
```

## 结语

如果分支数量不多而提交数量很大，这样做没什么问题，时间换空间同时不用考虑一致性问题

但是对于那种一下子 23 万个标签的奇怪仓库，还需要进一步考虑：

- 每个提交携带布隆过滤器缓存 Reference？每次全量重刷？
- 非 First Parent Commit 怎么办？
- 能不能把上面的业务操作封装为扩展插件？

