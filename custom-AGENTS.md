# AGENTS.md（中文译文）

> 说明：本文件是 `AGENTS.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](custom-文档清单.md)。

## 项目概述（Project Overview）

本仓库是一个 Rust workspace，提供用于证明机密 Guest（可信执行环境，Trusted Execution Environment，TEE）并向其投递机密（secrets）的工具与组件。它是
[Confidential Containers](https://github.com/confidential-containers) 项目的一部分。

主要组件如下：
- **KBS**（Key Broker Service，密钥代理服务）——远程证明与机密投递服务器（依赖方，Relying Party）
- **AS**（Attestation Service，证明服务）——校验 TEE 证据（验证者，Verifier）
- **RVPS**（Reference Value Provider Service，参考值提供者服务）——管理用于证据校验的参考值（reference values）
- **KBS Client** / **Trustee CLI**——用于测试、配置与密钥获取的客户端工具

上述组件的集合称为 "Trustee"，它既可以部署为若干通过 RPC 耦合的独立服务，也可以部署为单一的一体化（all-in-one）二进制程序。

## 安全考量（Security Considerations）

远程证明与机密投递是对安全性敏感的操作。对代码库的修改应当充分考虑其安全影响，新增代码应当被全面的测试所覆盖，以验证其安全属性。我们鼓励使用能在类型系统中表达不变条件的编码模式，例如使用类型状态模式（type state pattern）或新类型（newtype）来防止 API 被误用。始终考虑新功能的攻击面，并尽力将其最小化。

## API 稳定性（API Stability）

本项目会被打包并向下游分发。Trustee 目前还没有为其所有载荷与协议提供机器可强制校验的 schema，但我们在迭代过程中会努力避免对使用者造成破坏。如果破坏不可避免，我们会在提交信息中明确指出，以便它能在发布说明（release notes）中被追踪与汇总。

## 与 Guest Components 的关系（Relationship to Guest Components）

confidential-containers/guest-components 是 Trustee 运行在 TEE 中的对应方。它提供证明证据，并消费由 Trustee 投递的机密。这两个仓库会同步开发与发布，但它们是解耦的，可以独立使用。在 `kbs-client` crate 中，我们会引用 guest-components 的某个修订版本，在 Trustee 的任一修订版本中，我们都会针对该版本进行测试并保证兼容性。

## 示例验证器（Sample Verifiers）

本地开发不太可能发生在能够产生真实证明证据的 TEE 中。因此这里提供了 "sample" 与 "sample-device" 两对 attester/verifier，它们可在本地开发与测试中作为端到端流程的哑桩（dummy stub）使用。作为发布内容一部分的任何默认策略，都必须确保拒绝这些虚假证据；同时证明服务的发布构建应当排除 sample 验证器。

## 约定（Conventions）

### 提交（Commits）

提交信息必须包含一个子系统标识，以前缀加冒号的形式给出，例如
"verifier: fix hex encoding"。这不必严格匹配，但它仍应指出被改动的代码库的大致范围，例如 "kbs:"、"as:"、"ci:" 等。

每个提交都应当可以单独编译，并且包含原子化的改动。

## PR（Pull Request）

PR 标题必须包含一个子系统标识，以前缀加冒号的形式给出，与提交信息类似。我们避免在 PR 描述中塞入过多信息，尤其不鼓励过度使用流行词与 emoji。我们希望知道改动是**为什么**做的、是**如何**做的，以及在适用情况下潜在的负面影响和已经考虑过的备选方案。

## 仓库结构（Repository Structure）

```
├── kbs/                        # Key Broker Service
│   ├── docker/                 # Dockerfiles
│   ├── docs/                   # KBS documentation
│   ├── config/                 # Kubernetes manifests
│   ├── test/                   # E2E tests, test data
│   └── Makefile
├── attestation-service/        # Attestation Service
│   ├── docker/                 # Dockerfiles
│   ├── tests/                  # integration tests, OPA policies, evidence templates
│   └── Makefile
├── rvps/                       # Reference Value Provider Service
│   ├── docker/                 # Dockerfile
│   └── Makefile
├── tools/
│   ├── kbs-client/             # KBS client library & CLI
│   └── trustee-cli/            # Unified Trustee CLI tool
│       └── Makefile
├── deps/                       # Shared library crates
│   ├── verifier/               # TEE verifier implementations
│   ├── policy-engine/          # OPA policy engine (regorus)
│   ├── key-value-storage/      # Storage abstraction (PostgreSQL, local)
│   └── eventlog/               # TCG event log parser
├── integration-tests/          # Workspace-level integration tests
├── protos/                     # Protocol Buffer definitions (attestation, reference)
├── hack/                       # Helper scripts (e2e.sh, release-helper.sh)
├── docker-compose.yml          # Full-stack local deployment
├── .github/workflows/          # CI/CD pipelines
└── .devcontainer/              # Dev container configurations
```

## 构建与开发（Build & Development）

### 前置条件（Prerequisites）

- **Rust 工具链**：版本锁定在 `rust-toolchain.toml`（当前为 `1.95.0`）
- **Protobuf 编译器**（`protoc`），用于 gRPC 代码生成
- **OpenSSL** 开发库
- **tpm2-tss**（用于 TPM verifier 特性）
- **libsgx-dcap-quote-verify**（用于 Intel SGX 与 TDX verifier 特性），从 https://download.01.org/intel-sgx/ 安装

### 构建（Building）

每个组件都有自己的 `Makefile`。常用目标：

```bash
# KBS
make -C kbs build                 # Build KBS (default: background-check mode)
make -C kbs AS_TYPE=coco-as       # Select attestation service type
make -C kbs ALIYUN=true           # Enable Aliyun KMS backend
make -C kbs VAULT=true            # Enable HashiCorp Vault backend

# Attestation Service
make -C attestation-service build
make -C attestation-service grpc-as
make -C attestation-service restful-as

# RVPS
make -C rvps build

# Trustee CLI
make -C tools/trustee-cli build
```

或者构建整个 workspace：

```bash
cargo build --workspace
```

按 crate 带特性运行测试：

```bash
cargo test -p verifier \
    --no-default-features \
    --features snp-verifier
```

### 特性开关（Feature Flags）

特性开关被大量用于选择 TEE 验证器与后端：

- **验证器（Verifiers）**：`tdx-verifier`、`sgx-verifier`、`snp-verifier`、`cca-verifier`、
  `csv-verifier`、`tpm-verifier`、`all-verifier`、`az-snp-vtpm-verifier`、
  `az-tdx-vtpm-verifier` 等
- **KBS 模式（KBS modes）**：`coco-as-builtin`（进程内 AS）、`coco-as-grpc`（远端 gRPC AS）、
  `intel-trust-authority-as`
- **KMS 后端（KMS backends）**：`aliyun`、`pkcs11`
- **插件（Plugins）**：`nebula-ca-plugin`、`vault`
- **架构相关（Architecture-specific）**：某些验证器被限定在 x86_64、aarch64 或 s390x 上

### 测试（Testing）

```bash
# Unit tests per crate
cargo test -p kbs
cargo test -p attestation-service
cargo test -p rvps
cargo test -p verifier

# Linting and formatting (per-component Makefiles)
make -C kbs lint                  # cargo clippy
make -C kbs format                # cargo fmt

# OPA policy validation
make -C attestation-service opa-check

# E2E tests
make -C kbs/test
make -C attestation-service/tests/e2e
```

### Docker Compose（本地集群，Local Cluster）

```bash
docker-compose up -d
```

启动顺序：**setup**（证书生成）→ **rvps**（:50003）→ **as**（:50004）→ **kbs**（:8080）→ **keyprovider**（:50000）

## 架构与关键概念（Architecture & Key Concepts）

- **RATS 模型**：KBS 作为依赖方（Relying Party），AS 作为验证者（Verifier），RVPS 提供背书/参考值（endorsements/reference values）
- **gRPC**：AS 与 RVPS 暴露由 `protos/` 定义的 gRPC 服务（attestation.proto、reference.proto）
- **策略引擎（Policy engine）**：通过 `regorus` crate 使用 OPA（Rego）策略，用于证明决策
- **TEE 支持**：TDX、SGX、SNP、CCA、CSV、Hygon DCU、SE、NVIDIA GPU、TPM、Azure vTPM（SNP 与 TDX）
- **目标架构**：x86_64、aarch64、s390x

## 代码约定（Code Conventions）

- **Edition**：Rust 2021
- **错误处理（Error handling）**：应用使用 `anyhow`，库使用 `thiserror`
- **异步运行时（Async runtime）**：`tokio`（full features）
- **序列化（Serialization）**：全程使用 `serde` / `serde_json`
- **日志（Logging）**：`log` crate + `env_logger`，正在向 `tracing` / `tracing-subscriber` 迁移
- **CLI 解析（CLI parsing）**：使用 `clap` 的 derive 宏
- **测试（Testing）**：参数化测试使用 `rstest`，需要独占访问的测试使用 `serial_test`
- **格式化（Formatting）**：提交前始终运行 `cargo fmt --all`，以保证代码格式一致
- **静态检查（Linting）**：提交前始终运行 `cargo clippy --all-targets -- -D warnings`，并修复所有 warning 与 error

## CI/CD（持续集成与交付）

工作流位于 `.github/workflows/`：
- **kbs-rust.yml** / **as-rust.yml** / **trustee-cli-rust.yml** —— 在 ubuntu-24.04（以及 ARM、s390x）上构建与测试
- **kbs-e2e.yml** / **as-e2e.yml** —— 端到端测试
- **build-*-image.yml** / **push-*-image-to-ghcr.yml** —— 容器镜像构建与 GHCR 发布
- **actionlint.yml** —— 工作流校验
- **link.yml** —— 失效链接检查
- **scorecard.yaml** —— OpenSSF scorecard
