# SWID RIM 提取器

> 说明：本文件是 `rvps/src/extractors/swid/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../custom-文档清单.md)。

该提取器接收一份带有 RIM 绑定与 SWID tag 的 manifest，并从中提取参考值。
该提取器并不处理此类 manifest 中所有可能的声明项，
并且仅在一块 H100 GPU 的 vbios 的 nvidia RIM manifest 上做过测试。
未来可以扩展为更通用的 RIM/SWID 支持。

目前，该提取器不校验 manifest 的签名。
在把 manifest 提供给 RVPS 之前，应当先验证其真实性。

默认过期时间为 12 个月，哈希算法为 `sha384`。