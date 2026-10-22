# Initdata 规范

> 说明：本文件是 `kbs/docs/initdata.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

Initdata 规范定义了关键的数据结构与算法，
用于把任意定义良好的数据从不可信宿主注入 TEE。
为了保证这些数据的完整性，将利用 TEE 证据的 hostdata 能力或 (v)TPM 的动态度量能力。

## 简介

TEE 为用户提供一个隔离的执行环境，
以防止不可信宿主与外部软件栈窃听和篡改 TEE 内正在使用的用户数据。

远程证明技术用于校验运行在 TEE 中的软件足迹是否符合预期。
被度量的软件通常由硬件厂商（例如 TEE 硬件的固件与 tcb 安全版本）
或软件厂商（例如 VM 的 guest 内核）提供。这些组件相对静态，
这意味着它们多次部署之间可能都相同。

在某些场景中，用户会在启动时注入一些其他信息，例如
[kata 的策略文件](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-the-kata-agent-policy.md)、
[guest 中运行组件](https://github.com/confidential-containers/guest-components/tree/main/confidential-data-hub#configuration-file)的配置文件、
[用于指定 TEE 身份的身份文件](https://github.com/keylime/rust-keylime/blob/master/keylime-agent.conf)。

与 TEE 中运行的静态软件（例如 TDX VM 的 guest 固件、SGX enclave 的 libos）相比，
这些信息在不同部署之间会动态变化，并且通常是配置。
我们把这些信息或配置称为 _Initdata_。Initdata 机制将提供一种通过远程证明来保护其完整性的方式。
需要注意的是，机密性不由 initdata 机制保护，因为不可信宿主可以看到这些数据的明文。

为了实现这一目标，我们定义了以下内容
- 一个名为 **Initdata** 的数据结构。该结构由用户提供，
以键值格式承载任意数据，交给不可信宿主机在启动时注入 TEE。
我们并不限制该数据结构的编码方式，这意味着 JSON、TOML 和 YAML 都可以选择。
这将在 [Initdata](#initdata) 中介绍
- 一种数据完整性绑定机制。它将指导不可信宿主把 initdata 的摘要
绑定到证据中与硬件 TEE 相关的字段上。该字段将在远程证明期间由校验器检查。
这将在 [Integrity Binding for Different TEEs](#integrity-binding-for-different-tees) 中介绍
- 一种 initdata 摘要计算方法。该方法用于计算给定 initdata 的密码学哈希，
描述于 [Initdata Digest Calculating](#initdata-digest-calculating)。

本规范不定义 initdata 将如何被投递进 TEE。
不同的项目会有各自的实现方式。对于机密容器，
我们将使用 kata-runtime 与 kata-agent 协作来实现该功能。

## 术语

本节介绍本规范中使用的术语，以避免歧义。

- `Initdata`：一个数据结构，包含 initdata 数据以及其他有助于计算 initdata 摘要的信息。整个数据结构都会被投递进 guest。
- `Initdata Metadata`：initdata 的元数据字段，例如 `algorithm`、`version` 等。它们用于计算 initdata 摘要。
- `Initdata data`：在 TEE 启动时需要注入的数据。这些数据需要完整性保护，但不需要机密性保护。在 initdata 中，它们包含在 `data` 小节内。
- `Initdata digest`：按本规范计算出的 initdata 数据的摘要。它将用作 TEE hostdata/initdata 字段的取值。
- `TEE initdata/hostdata`：可以绑定到特定 TEE 实例的字段。该字段信息会被包含在 TEE 签名的远程证明报告中。典型的有 Intel TDX 的 `mr_config_id`、AMD SNP 的 `hostdata` 以及 Arm CCA 的 `CCA_REALM_PERSONALIZATION_VALUE`。为避免与 AMD SNP 的 hostdata 字段混淆，当我们不强调某个具体平台（例如 SNP）时，指的是各个 TEE 平台上对应的字段。

## 规范

### Initdata（初始化数据）

<a id="initdata"></a>

Initdata 定义了一种标准化的结构格式。请注意，
它并不指明具体的编码格式，但要求该编码格式必须支持表达键值对数据。
满足该要求的典型编码包括 JSON、TOML 和 YAML 等。

一个 initdata 应当（SHOULD）包含以下字段
- `version`：initdata 元数据的格式版本。版本号将提供可扩展性。本规范中的定义均为 `0.1.0`。
- `algorithm`：用于计算要写入 TEE initdata 的取值的哈希算法。典型的算法有 `sha-256`、`sha-384`、`sha-512`。名称遵循 [IANA Hash Function Textual Names](https://www.iana.org/assignments/hash-function-text-names/hash-function-text-names.xhtml)
- `data`：从字符串到字符串的键值映射。包含 initdata 的具体内容。

#### 不同编码的示例

假设存在一个具备如下取值的 initdata
- `version`：`0.1.0`
- `algorithm`：`sha384`
- `data`：包含两个对象。第一个的键名是 `attestation-agent.json`，取值是一个 JSON 字符串。第二个的键名是 `policy.rego`，取值是一个 rego 文件的字符串。

##### JSON 版本

JSON 版本的 initdata 如下所示
```json
{
  "algorithm": "sha384",
  "version": "0.1.0",
  "data": {
    "attestation-agent.json": "{\"aa_kbc_params\": \"cc_kbc::http://127.0.0.1:8080\"}",
    "policy.rego": "package agent_policy\nimport future.keywords.in\nimport future.keywords.every\nimport input\n\n# Default values, returned by OPA when rules cannot be evaluated to true.\ndefault CopyFileRequest := false\ndefault CreateContainerRequest := false\ndefault CreateSandboxRequest := true\ndefault DestroySandboxRequest := true\ndefault ExecProcessRequest := false\ndefault GetOOMEventRequest := true\ndefault GuestDetailsRequest := true\ndefault OnlineCPUMemRequest := true\ndefault PullImageRequest := true\ndefault ReadStreamRequest := false\ndefault RemoveContainerRequest := true\ndefault RemoveStaleVirtiofsShareMountsRequest := true\ndefault SignalProcessRequest := true\ndefault StartContainerRequest := true\ndefault StatsContainerRequest := true\ndefault TtyWinResizeRequest := true\ndefault UpdateEphemeralMountsRequest := true\ndefault UpdateInterfaceRequest := true\ndefault UpdateRoutesRequest := true\ndefault WaitProcessRequest := true\ndefault WriteStreamRequest := false"
  }
}
```

它会涉及大量转义字符。JSON 更适合设置简单的键值，如下所示
```json
{
  "algorithm": "sha384",
  "version": "0.1.0",
  "data": {
    "key1": "value1",
    "key2": "value2"
  }
}
```

##### TOML 版本

如果你希望避免转义字符，并直接使用 ascii 文件内容的明文作为 initdata，
TOML 格式会更合适。TOML 版本的 initdata 如下所示
```toml
algorithm = "sha384"
version = "0.1.0"

[data]
"attestation-agent.json" = '''
{
"kbs_addr": "http://172.18.0.1:8080"
}
'''

"policy.rego" = '''
package agent_policy

import future.keywords.in
import future.keywords.every

import input

# Default values, returned by OPA when rules cannot be evaluated to true.
default CopyFileRequest := false
default CreateContainerRequest := false
default CreateSandboxRequest := true
default DestroySandboxRequest := true
default ExecProcessRequest := false
default GetOOMEventRequest := true
default GuestDetailsRequest := true
default OnlineCPUMemRequest := true
default PullImageRequest := true
default ReadStreamRequest := false
default RemoveContainerRequest := true
default RemoveStaleVirtiofsShareMountsRequest := true
default SignalProcessRequest := true
default StartContainerRequest := true
default StatsContainerRequest := true
default TtyWinResizeRequest := true
default UpdateEphemeralMountsRequest := true
default UpdateInterfaceRequest := true
default UpdateRoutesRequest := true
default WaitProcessRequest := true
default WriteStreamRequest := false'''
```

### Integrity Binding for Different TEEs（面向不同 TEE 的完整性绑定）

<a id="integrity-binding-for-different-tees"></a>

有多种方式可以把 initdata 的完整性绑定到 TEE 证据上。
许多 TEE 平台都支持 TEE initdata 字段。该字段可以在启动 TEE 时由不可信宿主设置，
并且该字段会被包含在 TEE 证据中以用于远程证明。

各平台及其证据中对应的字段
- Intel TDX：`mr_config_id`，48 字节。实际上 `mr_owner` 与 `mr_owner_config` 具有类似属性，但我们只选择 `mr_config_id` 用于该用途。
- AMD SNP：`hostdata`，32 字节。
- Arm CCA：`CCA_REALM_PERSONALIZATION_VALUE`，64 字节。
- Intel SGX：`CONFIGID`，64 字节。
- IBM SE：`user_data`，256 字节。

当用户希望部署一个 TEE 时，需要准备一份 initdata。
宿主（可能是不可信的）应当（SHOULD）以 initdata 摘要作为 TEE initdata 来启动 TEE 实例。

TEE 外部的软件以何种方式把 initdata 投递进 TEE，不在本文档的规定范围内。
TEE 内部的软件栈**必须（MUST）**确立 initdata 数据字段的完整性。

其他平台，例如基于 (v)TPM 的平台，可以在使用之前通过扩展 PCR 来记录 initdata 摘要。
这种方式同样可以完成与 TEE 证据的完整性绑定。

如果计算出的 initdata 摘要长度长于或短于 TEE initdata 字段的字节长度，
可以应用截断与填充规则，使 initdata 摘要与 TEE initdata 字段长度一致。
- 如果 `len(calculated initdata digest) > len(TEE initdata)`，则在 initdata 摘要末尾截断
`len(calculated initdata digest) - len(TEE initdata)` 个字节。
- 如果 `len(calculated initdata digest) < len(TEE initdata)`，则在 initdata 摘要末尾填充
`len(calculated initdata digest) - len(TEE initdata)` 个字节的 `\0`。

### Initdata Digest Calculating（Initdata 摘要计算）

<a id="initdata-digest-calculating"></a>

当我们获得一份 initdata 时，可以直接使用字段 `algorithm` 所指定的哈希算法
对整个 initdata 进行计算，从而得到 initdata 摘要。

这意味着同一份 initdata 的不同编码会得到不同的 initdata 摘要。
例如
```json
{
  "algorithm": "sha384",
  "version": "0.1.0",
  "data": {
    "key1": "value1",
    "key2": "value2"
  }
}
```
与
```toml
algorithm = "sha384"
version = "0.1.0"

[data]
"key1" = '''value1'''
"key2" = '''value2'''
```
显然会得到不同的摘要。因此具体的使用场景必须确保生产方与消费方使用相同的编码。

# 使用场景

## 机密容器

机密容器（CoCo）利用 Initdata 来注入诸如
[kata-agent 的策略](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-the-kata-agent-policy.md)、
[guest 组件](https://github.com/confidential-containers/guest-components)的配置等。

[Confidential Containers PeerPod](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/docs/initdata.md) 中的做法

initdata 的编码为 TOML。

为了确立 initdata 数据的完整性，TEE 内部的 CoCo 软件会：

按规范计算 data 字段的摘要，校验计算出的取值与用于完整性绑定的 TEE 字段取值一致。
执行远程证明，以校验用于完整性绑定的 TEE 字段的期望取值。
