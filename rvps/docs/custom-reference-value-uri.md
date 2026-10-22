# 参考值 URI

> 说明：本文件是 `rvps/docs/reference-value-uri.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

## 背景

[证明服务](../../attestation-service/)的策略目前提供了 `query_reference_value` 扩展，
它允许策略在评估期间从 RVPS 检索参考值（RV）。
随着 RV 使用场景的增长（例如在构建流水线中为 CoCo/Kata 发布 RV），
我们需要一种被推荐且可互操作的方式来组织和索引 RV，
以便它们能够跨不同的项目/域以及版本易于管理。

与此同时，当今 RVPS 的存储接口本质上是一个键值模型，其中键可以是任意字符串。
这意味着即使没有标准化的键格式，系统也已经可以工作。
因此，本 RFC 定义了一份针对 RV 标识符的约定/规范**建议**，
主要是为了提升一致性与可运维性，同时保持对任意键的向后兼容。

> [!NOTE]  
> 在[讨论](https://github.com/confidential-containers/trustee/issues/1159)中确认了一个未来的方向：
> "拉取模型"（由 RVPS 从外部来源获取 RV），这正是需要能够定位外部资源的 URI 的主要驱动因素。
> 然而，由于"外部位置 URI 设计"与拉取模型设计紧密耦合，
> 我们明确把本 RFC 的范围先限定在一种最小格式上，
> 而把 authority/外部指针的部分留给后续的提案。

## 规范

### 范围与非目标

#### 范围内

1. 定义一种供 `query_reference_value` 消费的推荐 RV 标识符格式。
2. 提供命名空间/路径 + 可选 tag 的语义，适合用于组织 RV 与版本。
3. 定义围绕缺失 tag 以及类 latest 用法的预期行为（作为建议）。

#### 范围外（未来工作）
1. 把 RVPS 地址 / 远端位置 / 插件机制编码进 URI（拉取模型）。
2. 标准化的 push/update API

### 语义："实例局部"的标识符，"全局有意义"的约定

一个 RV URI 是在某个具体的 RVPS 实例内被解析的。具体而言：

`query_reference_value(rv_uri)` 是针对该 AS 部署所配置/使用的 RVPS 求值的。
同一个 `rvps:///...` 标识符在不同的 RVPS 实例上可能解析为不同的内容，
取决于该实例上设置了什么。

虽然目前解析是实例局部的，但建议参考值的发布者采用稳定且唯一的 URI 命名约定，以便：

1. 不同组件通过命名空间路径清晰分隔。
2. 不同版本通过 tag 清晰分隔。
3. 同一个逻辑上的 RV 可以在工具链、文档与策略之间被一致地引用 —— 即使多个 RVPS 实例镜像了同一套命名方案。

### RV 标识符格式（推荐）

#### 规范形式
```
rvps:/// <namespace-1> / <namespace-2> / ... / <namespace-n> [ : <tag> ]
```

其中，
1. 前缀必须（MUST）是 `rvps:///`（scheme + 空 authority）。
2. 至少**必须（MUST）**有一个命名空间段。
3. `:tag` 是**可选的（OPTIONAL）**。

示例：

- `rvps:///confidential-containers/kata/measurements:v0.18.0`
- `rvps:///org/project/component`
- `rvps:///coco/kata/tdx/rtmr0:latest`

#### 命名空间（`/<namespace-x>/...`）

1. 命名空间段是层次化的，且纯粹用于组织。
2. 本规范不强制规定各段的含义；发布者/运维方自行定义其约定。
3. 实现**应当（SHOULD）**把整个 `/<...>` 部分（若存在则加上 tag）作为查找键。

> [!NOTE]  
> **与 KBS 资源 URI 的区别：** KBS 资源 URI 使用固定的三段结构（例如 `kbs:///repo/type/tag`，分别对应 repository、type 和 tag）。RV URI 不同：它拥有可变数量的命名空间段（`/<namespace-1>/<namespace-2>/.../<namespace-n>`）以及一个可选的 `:tag`。对于 RV URI 没有固定的"三段"规则；路径段的数量由发布者选择。

#### 字符指引（推荐，不强制）：

1. 使用 URL 安全字符；避免空格。
2. 优先使用小写字母、`-`、`_`、`.`

#### Tag（`:tag`）

1. Tag 表示一个版本/变体选择器。
2. Tag 不保证不可变（例如 latest 可能会移动）。
3. 运维方**应当（SHOULD）**优先使用不可变的发布 tag（例如 `v0.18.0`）以得到稳定的策略。
4. **没有默认行为：** 对存储层而言，RV 标识符就是一个不透明字符串。省略 tag（例如 `rvps:///coco/kata/measurements`）与显式使用 `:latest` tag（例如 `rvps:///coco/kata/measurements:latest`）**并不**等价，除非某个实现显式定义了这样的行为。策略与工具链应当使用它们打算解析的完整字符串；不要假定"没有 tag"就意味着"latest"，反之亦然。

### RVPS 中的解析规则（推荐行为）

由于底层存储是使用任意字符串键的键值存储，RVPS **可以（MAY）**
简单地把完整的 RV 标识符字符串作为键来实现以下内容。
不过，如果 RVPS 选择去解析该规范形式，则推荐以下行为以保证互操作性：

#### 存在 tag 时

按 (ns-path, tag) 精确匹配解析。

#### latest tag（latest 标签）

如果支持 latest：

RVPS **可以（MAY）**把它解析为一个可移动的别名。
RVPS **应当（SHOULD）**记录/审计所解析到的具体 RV 实例。

### 向后兼容 / 非强制

1. 本 RFC 并不要求所有参考值键都遵循该格式。
2. `query_reference_value` 与 RVPS 必须（MUST）继续支持任意字符串键（既有行为）。
3. `rvps:///...` 格式是一份推荐约定，用于提升一致性并支持未来的扩展。

### CoCo 命名约定

对于机密容器（CoCo）生态，我们定义一个推荐的公共前缀，
以标准化跨组件与跨版本的 RV 命名：

1. 公共前缀：`rvps:///github.com/confidential-containers/`

2. 组件与产物命名：
发布者**应当（SHOULD）**把与具体组件相关的路径置于该前缀之下，例如（仅为示意，非强制）：

- `rvps:///github.com/confidential-containers/tdx/mr_td:<release-tag>`
- `rvps:///github.com/confidential-containers/tdx/eventlog/kernel:<release-tag>`
- `rvps:///github.com/confidential-containers/snp/measurement:<release-tag>`

3. 版本管理：CoCo 发布者**应当（SHOULD）**使用 `:tag` 来区分发布版本（例如 `:v0.18.0`）。
要求可复现的策略**应当（SHOULD）**避免使用 `:latest`。

该约定旨在确保不同组件与不同发布版本自然地由路径和 tag 分隔，
并使得围绕一致的标识符方案构建共享工具链（发布者、CI 流水线、镜像 RVPS 实例）更加容易。

### 策略示例

证明策略使用 `query_reference_value(rv_uri)` 扩展从 RVPS 获取参考值。
其参数是 RV 标识符字符串（例如推荐的 `rvps:///...` 形式）。
以下是一个面向 TDX 的最小化 Rego 示例，说明如何按 URI 查询 RV；
在实践中，你应当使用发布者实际存入的那些 URI（例如带 CoCo 命名与版本 tag 的 URI）。

```rego
package policy

import rego.v1

# ...

# Example: TDX executables check using RV URIs (recommended format with tag)
executables := 3 if {
    input.tdx

    # RTMR measurements from RVPS using versioned URIs
    input.tdx.quote.body.rtmr_1 == query_reference_value("rvps:///github.com/confidential-containers/tdx/rtmr_1:v0.18.0")
    input.tdx.quote.body.rtmr_2 == query_reference_value("rvps:///github.com/confidential-containers/tdx/rtmr_2:v0.18.0")
}

# Example: TDX hardware check (mr_seam, tcb_svn, mr_td)
hardware := 2 if {
    input.tdx

    input.tdx.quote.header.tee_type == "81000000"
    input.tdx.quote.body.mr_seam == query_reference_value("rvps:///github.com/confidential-containers/tdx/mr_seam:v0.18.0")
    input.tdx.quote.body.tcb_svn == query_reference_value("rvps:///github.com/confidential-containers/tdx/tcb_svn:v0.18.0")
    input.tdx.quote.body.mr_td == query_reference_value("rvps:///github.com/confidential-containers/tdx/mr_td:v0.18.0")
}
```

> [!WARN]
> 该策略仅用作示例。其中提到的参考值 URI 尚不存在。
> 策略应当使用与发布到 RVPS 的内容相对应的精确 RV URI 字符串（包括命名空间路径与 tag）。
> 在策略中使用不可变 tag（例如 `v0.18.0`）可以确保证明决策的可复现性。

### 未来扩展点（拉取模型）

`://` 与第一个 `/` 之间的部分（即 URI 的 authority / 位置 / 方式）在本 RFC 中被有意留空。
后续的提案**可以（MAY）**定义如何引用外部 RV 来源，作为拉取模型的一部分，可能包括：
1. 基于 authority 的 RVPS 选择，和/或
2. 来源定位符编码，和/或
3. 从 RV ID 到拉取来源的结构化元数据映射。
任何未来的设计都**必须（MUST）**保持与形如 `rvps:///path[:tag]` 的标识符兼容。
