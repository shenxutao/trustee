
> 说明：本文件是 `kbs/sample_policies/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。
本目录包含用于配置 KBS 策略引擎的示例策略文件。
你可以参考这些文件编写自己的策略。

另外还有一些[用于测试](../../deps/policy-engine/test_data/)的示例策略。

| 文件 | 说明 |
| ---  | ---         |
|[allow_all.rego](./allow_all.rego)|等价于关闭策略引擎。无条件释放资源|
|[deny_all.rego](./deny_all.rego)|拒绝释放任何资源|
|[affirming.rego](./affirming.rego)|仅当 AS 提供了肯定性（affirming）的 EAR 令牌时才释放资源|
