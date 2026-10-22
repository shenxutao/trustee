# 证明令牌（Attestation Token）

> 说明：本文件是 `attestation-service/docs/attestation_token.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

证明服务会生成一个 EAR 证明令牌，其中包含许多声明项（claims）。

关于证明令牌的整体结构，请参考 [EAR 规范](https://datatracker.ietf.org/doc/draft-ietf-rats-ear/)。

一般而言，证明令牌会为构成 TCB 的每一个设备（包括 CPU）包含一个评估结果（Appraisal）。
这些评估结果可以通过 EAR 令牌中的 `submods` 字段访问。
每一个 submod 都会被赋予一个由 CPU 类别和设备序号构成的通用键。
例如，CPU 的 submod 名为 `cpu0`。
如果 guest 中证明了 GPU，则会出现一个 `gpu0` submod。
如果存在多个 GPU，则还会有额外的 `gpuN` submod。
Hygon DCU 的证明以同样方式使用 `dcu0` 以及额外的 `dcuN` submod。

`cpu0` 被视为主要的证明方（primary attester），并带有关联的特殊信息。

# ReportData 与 InitData

ReportData 和 InitData 是 Trustee 的两个关键概念。
ReportData 指的是 guest 在证明时提供的数据。
它有时也被称为用户数据（user data）。KBS 协议使用该字段来度量 nonce 与 TEE 公钥。
在证明过程中，Trustee 会确保硬件证据中的对应字段与给定连接所期望的 report data 取值相匹配。

InitData 是一个更强大也更微妙的概念。更多信息参见 [InitData 规范](../../kbs/docs/initdata.md)。
其基本思想是：InitData 是对启动期配置字段（例如 SNP 上的 HostData 或 TDX 上的 MRConfig）的泛化。
InitData 用于向 guest 供给动态的、被度量的、但非机密的配置数据。
InitData 明文是一个包含该配置的 TOML 或 JSON 文件，
而 InitData 哈希则是该文件的哈希值，它会被加入度量值中。
客户端可以选择性地把 InitData 明文提供给 Trustee。
如果提供了，Trustee 会依据硬件证据校验该明文，
并将 InitData 明文暴露给策略引擎，同时作为证明令牌的一部分。

InitData 与 ReportData 通常都会被包含在证明令牌中。
这些字段位于 `cpu0` 评估结果的 `AnnotatedEvidence` 扩展之下。

如果提供了 InitData 明文，会对 InitData 做一些转换，使其更易于消费。

因此，令牌中的 InitData 部分可能包含以下字段：
- `cdh.toml`：InitData 中 CDH 配置文件的 JSON 版本。
- `aa.toml`：InitData 中 AA 配置文件的 JSON 版本。
- `agent_policy_claims`：来自 Kata Agent policy 的 `policy_data` 声明项（JSON 形式）。

这些字段只有在 InitData 明文中包含相应数据时才会出现。

# 硬件声明项

带注解的证据扩展中还会包含由各校验器提取出的、与硬件相关的声明项。
这些内容列在 [TCB 声明项](./tcb_claims.md)文档中。
