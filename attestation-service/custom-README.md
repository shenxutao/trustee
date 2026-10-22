# 证明服务（Attestation Service）

> 说明：本文件是 `attestation-service/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../custom-文档清单.md)。

证明服务（AS 或 CoCo-AS）用于校验硬件证据。
AS 最初是为与[机密容器的 KBS](../kbs) 配合使用而设计的，
但它也可以用于非常广泛的场景。
任何需要校验 TEE 证据的场合都可以使用 AS。

目前，AS 可以校验来自以下 TEE 的证据：
- Intel TDX
- Intel SGX
- AMD SEV-SNP
- ARM CCA
- Hygon CSV
- Azure 上的 Intel TDX + vTPM
- Azure 上的 AMD SEV-SNP + vTPM
- IBM Secure Execution（SE）：[配合 IBM SE 使用证明服务](../deps/verifier/src/se/README.md)
- NVIDIA

# 总览
```
                                      ┌───────────────────────┐
┌───────────────────────┐ 证据        │     证明服务（AS）     │
│                       ├────────────►│                       │
│  验证请求方           │             │ ┌────────┐ ┌──────────┴───────┐
│  （例如 KBS）         │             │ │ 策略   │ │ 参考值提供服务   │◄───参考值
│                       │◄────────────┤ │ 引擎   │ │ （RVPS）         │
└───────────────────────┘ 证明结果    │ └────────┘ └──────────┬───────┘
                        令牌         │                       │
                                      │ ┌───────────────────┐ │
                                      │ │  校验器驱动        │ │
                                      │ └───────────────────┘ │
                                      │                       │
                                      └───────────────────────┘
```

AS 提供一套简单的 API。它接收证明证据，并返回一个包含两步校验结果的证明令牌。
AS 既可以作为 Rust crate（库）直接被使用，也可以构建为独立服务，对外暴露 REST 或 gRPC API。
在机密容器中，AS 的客户端是 KBS，而证据则来源于 guest 内部的证明代理（Attestation Agent）。

AS 的校验过程分为两步。

1. 校验证据本身的格式与来源（即校验证据的签名）。这一步由[校验器驱动](#校验器驱动)完成。
2. 评估证据所声明的声明项（即检查度量值是否符合客户端预期）。这一步由[策略引擎](#策略引擎)和 [RVPS](#参考值提供服务)完成。

第一步由某个平台特定的[校验器驱动](#校验器驱动)完成。
第二步由[策略引擎](#策略引擎)在 [RVPS](#参考值提供服务) 的协助下完成。

# 快速开始

请参考：
- [RESTful CoCo AS](docs/restful-as.md#quick-start)
- [gRPC CoCo AS](docs/grpc-as.md#quick-start)

# 进阶主题

面向开发者的进阶主题。

## 库

AS 可以被构建为 Rust crate，并导入到任何需要提供证明服务的项目中。

由于 AS 的 API 尚未完全稳定，AS crate 需要直接从 GitHub 导入：

```toml
attestation-service = { git = "https://github.com/confidential-containers/trustee" }
```

## 证据格式：

证明请求中必须包含证明证据。
证明证据的格式取决于平台以及校验器的具体实现。

具体证据格式请参考各个校验器：
- Intel TDX：[TdxEvidence](../deps/verifier/src/tdx/mod.rs)
- Intel SGX：[SgxEvidence](../deps/verifier/src/sgx/mod.rs)
- AMD SNP：[SnpEvidence](../deps/verifier/src/snp/mod.rs)
- Azure SNP vTPM：[Evidence](../deps/verifier/src/az_snp_vtpm/mod.rs)
- Azure TDX vTPM：[Evidence](../deps/verifier/src/az_tdx_vtpm/mod.rs)
- Arm CCA：[CcaEvidence](../deps/verifier/src/cca/mod.rs)
- Hygon CSV：[CsvEvidence](../deps/verifier/src/csv/mod.rs)
- IBM Secure Execution（SE）：[SeEvidence](../deps/verifier/src/se/mod.rs)
- NVIDIA：[NvDeviceEvidence](../deps/verifier/src/nvidia/mod.rs)
- (v)TPM：[TpmEvidence](../deps/verifier/src/tpm/mod.rs)

## 输出

如果 TEE 证据校验未失败，AS 将返回一个证明结果令牌（Attestation Results Token）。

证明结果令牌是一个 [EAT Attestation Results（EAR）](https://www.ietf.org/archive/id/draft-fv-rats-ear-05.html)，
采用 [JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519) 格式，
其中包含解析出的证据声明项，例如 TCB 状态。

关于证明令牌的更多信息参见[此处](./docs/attestation_token.md)。

典型 EAR 令牌 JWT 的声明项部分参见[此处](./docs/example.token.json)。

## 架构

### 校验器驱动

校验器驱动负责解析与具体硬件 TEE（HW-TEE）相关的证明证据，并执行以下任务：

1. 校验证据中 TEE quote/report 的 HW-TEE 硬件签名。

2. 解析证据，并将 TCB 状态整理为 JSON 声明项返回。

受支持的校验器驱动：

- `sample`：一个虚拟的 TEE 校验器驱动，用于测试/演示 AS 的功能。
- `tdx`：面向 Intel Trust Domain Extensions（Intel TDX）的校验器驱动。
- `snp`：面向 AMD Secure Encrypted Virtualization-Secure Nested Paging（AMD SNP）的校验器驱动。
- `sgx`：面向 Intel Software Guard Extensions（Intel SGX）的校验器驱动。
- `az-snp-vtpm`：面向基于 SNP 的 Azure vTPM（Azure SNP vTPM）的校验器驱动。
- `cca`：面向 Confidential Compute Architecture（Arm CCA）的校验器驱动。
- `csv`：面向 China Security Virtualization（Hygon CSV）的校验器驱动。
- `hygon-dcu`：面向 Hygon Deep Computing Unit（DCU）的校验器驱动。
- `se`：面向 IBM Secure Execution（SE）的校验器驱动。
- `nvidia`：面向 NVIDIA 设备的校验器驱动。
- `tpm`：面向可信平台模块（TPM）的校验器驱动。
> [!WARNING]  
> **TPM 设备说明**：TPM 设备（Azure vTPM 系列除外）并未与 TEE 背书绑定。
> 当把 TPM 作为独立的证明设备使用（未与 Azure vTPM 集成）时，你必须确保自己信任该 TPM 本身，
> 因为 TPM 与底层 TEE 硬件之间不存在密码学绑定关系。
> 该特性主要用于开发目的。

#### 与架构相关的校验器配置

AS 会根据目标架构支持不同的校验器驱动集合：

- **x86_64**：通过 `all-verifier` 包含全部校验器驱动
- **aarch64**：包含 `cca`
- **s390x**：包含 `snp`、`nvidia` 和 `se`

### 策略引擎

[OPA](https://www.openpolicyagent.org/docs/latest/) 是一个灵活的策略引擎。
AS 允许用户在执行证据校验时定义并上传自己的 OPA 策略。
证明请求中的 `policy_ids` 字段决定评估哪些策略。
策略评估的结果会反映在令牌中。

**注意**：更多关于 Rego 的信息，请参考 [Policy Language](https://www.openpolicyagent.org/docs/latest/policy-language/) 文档。

如果策略未更新，AS 将使用[默认策略](src/ear_token/ear_default_policy_cpu.rego)。

> [!NOTE]
> 仅支持 EAR 令牌，Simple 令牌已被移除。
> 如果你此前使用的是 Simple 令牌，需要修改[配置文件](./docs/config.md#attestationtokenbroker)中的
> `attestation_token_broker` 部分。此外，
> 原有的 Simple 令牌策略可能无法工作，你可以参考[该目录](./src/ear_token/)下新的 `.rego` 策略。

策略的具体用法请参考[该指南](docs/policy.md)。

### 参考值提供服务

[参考值提供服务](../rvps/README.md)（RVPS）是集成进 AS 的一个模块，用于校验、存储和提供参考值。
RVPS 接收并校验来自软件供应链的来源输入，存储度量值，
并在 AS 校验证据时依据证据内容为 AS 生成参考值声明项。

参考值提供服务支持不同的部署模式，
详情请参考[该文档](../rvps/README.md#run-mode)。
