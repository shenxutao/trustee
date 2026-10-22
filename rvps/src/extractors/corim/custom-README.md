# CoRIM 提取器

> 说明：本文件是 `rvps/src/extractors/corim/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../custom-文档清单.md)。

该提取器解析 CoRIM manifest 并提取参考值。

CoRIM 规范的范畴不止于简单规定参考值，
但我们的提取器与 RVPS 内部的 RV 表示紧密绑定。

因此，本提取器会把 CoRIM 以及其中的 CoMID 映射为由 URI 标识的 RV 值。

我们使用一种灵活的机制把 CoRIM 中的 RV 映射到 RV URI。
具体而言，在解析 manifest 时发现的任何类 id 字段都会被用斜杠拼接起来，
以生成参考值的 URI。

该提取器目前只支持 CoRIM 的一个子集。
只支持 reference triples，并且并非所有类型的标识符都会被解析。

以下标识符会在存在时被加入 RV URI。
该列表是有序的，因此第一个可用的标识符将成为 URI 的基。

* Environment map vendor 字段
* Environment map model 字段
* MKey String、integer 或 Uuid
* Mval name
* Mval serial number

如果 mval 是一个摘要列表，则会在 URI 末尾追加字符串 "digests"，
其取值将是一个摘要列表。

如果 mval 是原始字节，则会在 URI 末尾追加字符串 "raw_bytes"，
其取值将是转换为十六进制的字节。

例如，一个 RV URI 可能形如：

* `ACME/RoadRunner/31fb5abf-023e-4992-aa4e-95f9c1503bfa/digests`
* `ACME/RoadRunner/31fb5abf-023e-4992-aa4e-95f9c1503bfa/raw_bytes`
