# 证明代理事件日志规范

> 说明：本文件是 `kbs/docs/confidential-containers-eventlog.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

## 简介

[机密计算事件日志（Confidential Computing Eventlog，CCEL）](https://uefi.org/specs/UEFI/2.11/38_Confidential_Computing.html)框架在操作系统启动阶段建立信任的过程中扮演着关键角色，
确保基础系统组件经过完整性与安全性验证。它基于
[TCG EFI Protocol Specification（第 5.2 节）](https://trustedcomputinggroup.org/wp-content/uploads/EFI-Protocol-Specification-rev13-160330final.pdf)与
[TCG PC Client Platform Firmware Profile Specification（第 9.2.2 节）](https://trustedcomputinggroup.org/wp-content/uploads/TCG_PCClient_PFP_r1p05_05_3feb20.pdf)
所规定的 `EFI_TCG2_EVENT_LOG_FORMAT_TCG_2` 格式（又称 `Crypto Agile Log Entry Format`）。
它提供了覆盖 pre-OS 事件的详细事件类型。为了把这种信任进一步延伸到启动过程之后，
我们引入了证明代理事件日志（Attestation Agent Eventlog，AAEL）格式，
以便为 post-OS 阶段提供全面的度量能力。AAEL 框架在设计时就考虑了可扩展性，
能够广泛支持众多类型的度量事件，包括内存操作、网络事务、文件系统活动以及与容器相关的流程。

AAEL 可以无缝集成进 `EFI_TCG2_EVENT_LOG_FORMAT_TCG_2`，
从而丰富现有的事件日志框架，并为计算环境提供一种整体性的安全与完整性方案。
理想情况下，AAEL 应当被嵌入由内核维护的 Eventlog 中，形成统一的日志系统。
然而，内核社区尚未就未来版本是否支持 Eventlog、以及可能采用何种格式（无论是 TCG2 还是其他格式[1]）给出明确指引。
鉴于这种不确定性，我们目前独立地提出 AAEL 格式。
关于它将如何存储在底层 Eventlog 中的决定，将在实现阶段随着更多信息而明确。
由于这些原因，本规范的部分内容可能会变更或被移除。

## 架构

支撑 AAEL 的架构可以与现有的机密计算系统无缝集成，
在 CVM 中于内核层之上提供一个健壮的事件日志与校验框架。
[证明代理（Attestation Agent，AA）](https://github.com/confidential-containers/guest-components/tree/main/attestation-agent)是这一过程的核心，
它负责在 AAEL 框架内记录事件，并把日志安全地绑定到硬件运行时度量寄存器。

它接收来自系统组件的日志记录请求。

这些调用方按照规范所定义的 AAEL Event 记录事件，从而确保日志数据的一致性与可靠性。

Trustee 的 [CoCo 证明服务（CoCo AS）](https://github.com/confidential-containers/trustee/tree/main/attestation-service)
提供了一个灵活的平台，以一种通用化的方式处理 AAEL 日志，
确保与各种机密计算环境兼容，以支持多样化的事件校验。
此外，AS 可以利用 AAEL Event Spec 对 AAEL 事件进行详细分析并强制执行策略，以实现精确的校验与审查。
该能力支持高级安全策略的制定与执行，从而增强对虚拟机状态的评估。

```                                                                         
                                                                    
┌─────────────────────────────────────┐                             
│                                     │                             
│          ┌────────────────┐         │                             
│          │    系统组件     │         │                             
│          └───────┬────────┘         │                             
│                  │                  │                             
│                  │                  │                             
│                  │                  │                             
│       ┌──────────▼──────────┐       │      ┌─────────────────────┐
│       │                     │       │      │                     │
│       │     证明代理        ├───────┼─────►│      证明服务       │
│       │                     │       │      │                     │
│       └───┬─────────┬───────┘       │      └─────────────────────┘
│           │         │               │                             
│           │         ▼               │                             
│    Extend │     ┌──────┐            │                             
│           │     │ AAEL │            │                             
│           │     └──────┘            │                             
│     ┌─────▼───────────────────┐     │                             
│     │   运行时度量 / PCR       │     │                             
│     └─────────────────────────┘     │                             
│                                     │                             
│            机密虚拟机               │                             
└─────────────────────────────────────┘                               
```

在本规范中，我们提供关于机密计算场景下证明代理事件日志（AAEL）格式的详细信息。

## 规范

AAEL 是一个面向 guest 内核层之上事件日志的规范。

### 证明代理事件日志（AAEL）

1. AAEL 与动态度量寄存器的绑定

由 AAEL 记录的条目会绑定到一个特定的寄存器值上，以确保所记录事件的完整性。
当 AAEL 被集成进一个既有的 Eventlog 框架（例如 `EFI_TCG2_EVENT_LOG_FORMAT_TCG_2`）时，
对底层寄存器的更新遵循该 Eventlog 规范的指引，通常使用 `EVENT` 字段来存储 AAEL 条目。

每加入一个新事件，都会在指定的寄存器上执行一次 extend 操作。
当平台支持 (v)TPM 接口时，该寄存器对应 (v)TPM 的 PCR 寄存器。
在仅使用 TEE 的平台上，PCR 会按照与平台相关的规则映射到特定的
[机密计算度量寄存器](https://uefi.org/specs/UEFI/2.11/38_Confidential_Computing.html#virtual-platform-cc-event-log)。
这种映射确保了 AAEL 在 TEE 与非 TEE 环境中都适用。

对于 AAEL 直接以 RAW 格式存储的场景，AAEL 条目的摘要会被直接用于扩展指定的寄存器。

2. AAEL 日志条目格式

AAEL 日志条目由两种类型组成：Event Entry（事件条目）与 INIT Entry（初始化条目）。

__INIT Entry__ 在 AAEL 开始时（即 AA 首次初始化时）只记录一次，
捕获某个特定 PCR 的当前值，格式为

```
INIT/<hash-algorithm> <hex-digest>
```
其中，
- `<hash-algorithm>`：可以是 `sha256`、`sha384` 或 `sha512`。
- `<hex-digest>` 是 base16 编码的 PCR 寄存器值。其长度**必须**与 `<hash-algorithm>` 对齐。必要时**必须**通过补零或截断来对齐摘要长度。

__Event Entry__ 按以下格式记录具体事件
```
<Domain> <Operation> <Content>
```

其中，
- `Domain`：**必需**。事件所属的域，**建议**使用 URI。
- `Operation`：**必需**。该域内的具体操作。
- `Content`：**必需**。该操作的详细上下文。

事件格式要求

- `Domain` 与 `Operation` 两个字段都**不得**包含任何[空白字符](https://www.ascii-code.com/characters/white-space-characters)
- 三个字段都**必须**仅由[可打印字符](https://www.ascii-code.com/characters/printable-characters)组成。
- `Domain` 与 `Operation` 字段由单个空格分隔（ASCII 码 `0x20`）。
- `Operation` 与 `Content` 字段由单个空格分隔（ASCII 码 `0x20`）。
- `Content` 字段**不得**包含任何换行符（LF，`\n`，ASCII 码 `0x0A`）。
- AAEL 中的每一行**必须**以换行符（LF，`\n`，ASCII 码 `0x0A`）结尾，并且**不得**包含任何额外的回车符（CR，`\r`，ASCII 码 `0x0D`）。

这三个字段的语义由调用 AA 记录事件的具体组件定义。

3. 度量 API

证明代理必须提供一个支持 AAEL 记录的 API，以便与现有的事件日志框架集成。

```proto
// Extend the dynamic/runtime measurement with given materials. This would change the state
// of current TEE's status, e.g. TDX's RTMR, (v)TPM's PCR, by adding a record in eventlog.
message ExtendRuntimeMeasurementRequest {
    // The domain to which this event entry belongs. This domain is used to distinguish the semantics of log entries in different contexts.
    string Domain = 1;

    // Concrete operation type that this event entry records.
    string Operation = 2;

    // Detailed content of the operation that this event entry records.
    string Content = 3;

    // Which PCR will be extended with the hash of this entry.
    optional uint64 RegisterIndex = 4;
}
```

默认情况下，`RegisterIndex` 被设为 `17`，因为它在支持动态度量信任根（dynamic root of trust for measurement）方面发挥作用。
值得注意的是，PCR `17` 并未在 [Linux TPM PCR Registry](https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/) 中注册。
此外，[TCG TRUSTED BOOT CHAIN IN EDK II](https://tianocore-docs.github.io/edk2-TrustedBootChain/release-1.00/3_TCG_Trusted_Boot_Chain_in_EDKII.html)
规定 __PCR [17-22] 代表平台的动态度量信任根（DRTM）。__

### 机密容器事件规范

机密容器事件规范（CoCo Event Spec）建立在 AAEL 框架之上，
规定了与机密容器这一独特场景相关的事件类型。

该规范涵盖与 guest 中机密容器生命周期紧密相关的事件。
通过聚焦这些生命周期事件，CoCo Event Spec 确保了对安全计算环境中关键容器操作的全面监控与校验。

```
                                                                    
┌─────────────────────────────────────┐                             
│                                     │                             
│       ┌────────────────────┐        │                             
│       │ Kata-Agent/ASR/CDH │        │                             
│       └──────────┬─────────┘        │                             
│                  │                  │                             
│                  │                  │                             
│                  │                  │                             
│       ┌──────────▼──────────┐       │      ┌─────────────────────┐
│       │                     │       │      │                     │
│       │      证明代理       ├───────┼─────►│      证明服务       │
│       │                     │       │      │                     │
│       └───┬─────────┬───────┘       │      └─────────────────────┘
│           │         │               │                             
│           │         ▼               │                             
│    Extend │     ┌──────┐            │                             
│           │     │ AAEL │            │                             
│           │     └──────┘            │                             
│     ┌─────▼───────────────────┐     │                             
│     │   运行时度量 / PCR       │     │                             
│     └─────────────────────────┘     │                             
│                                     │                             
│            机密虚拟机               │                             
└─────────────────────────────────────┘                             
```

它被设计为灵活且可扩展的格式，可用于表示机密容器环境中的各种事件。
CoCo Event Spec 是基于 AAEL 规范对 `Domain`、`Operation`、`Content` 的具体定义。

CoCo 事件的 `domain` **必须**设为 `github.com/confidential-containers`。

`Content` 字段**必须**是 JSON。
`Content` JSON **必须**被规范化（依据 [RFC 8785](https://datatracker.ietf.org/doc/html/rfc8785) 方案）

具体受支持的 `Operation` 与 `Content` 定义如下表：
| Operation | Content | 说明 | Content 示例 |
| --- | --- | --- | --- |
| `PullImage` | `{"image":"<image-reference>","digest":"<digest>:<hex>"}` | 一次镜像拉取事件，包含镜像引用与 manifest 摘要 | `{"image":"alpine","digest":"sha256:0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0"}` |
| `InitData` | `{"digest":"<algorithm>:<hex>"}` | initdata 摘要，在平台无法以其他方式绑定它时记录（例如 MRCONFIGID 未设置的 TDX）。每次启动最多记录一次 | `{"digest":"sha384:0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c0c"}` |

欢迎通过提交 PR 来增加更多事件。

## 参考资料

[1] https://lore.kernel.org/linux-coco/42c5eba9-381b-4639-9131-f645b375d235@linux.intel.com/T/#m086550ee8ca4d0127657ca8a467bf7cf170bfb74
