# Sample 提取器

> 说明：本文件是 `rvps/src/extractors/sample/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../custom-文档清单.md)。

该提取器会直接从输入中提取参考值，而**不会**校验任何签名。

该格式仅用于测试与演示。它应当被一份包含软件供应链信任关系的、经过签名的来源所取代。

## 来源格式

`Message` 中 sample 来源的格式如下
```json
{
    "<name-1>": [
        "<reference-value-1>",
        "<reference-value-2>",
        ...
    ],
    "<name-2>": [
        "<reference-value-1>",
        "<reference-value-2>",
    ],
    ...
}
```

默认过期时间为 12 个月，哈希算法为 `sha384`。