# 个人文档清单（custom- 索引）

> 目的：作为本仓库所有 `custom-` 前缀文件的**总索引**，方便总览、并跟踪「译文相对 upstream 原文的过期情况」。
> 命名约定：每份译文与原文**同目录**、文件名加 `custom-` 前缀（个人新增文档直接放根目录，亦加 `custom-`）。

## 用法：如何发现「上游已更新、译文该同步」

「原文基准」列是**本地仓库中该原文最后一次被修改的提交与日期**（即翻译时所依据的版本）。当后续执行 `git fetch upstream` 后，若某原文在基准提交之后又有新提交，说明该译文可能已过期，需要重译对应段落。

```bash
git fetch upstream
# 检查某个原文是否在基准之后被改动（以 KBS 证明协议为例）
git log --oneline <基准commit>..upstream/main -- kbs/docs/kbs_attestation_protocol.md
```

> 说明：本清单的「原文基准」取自**当前本地仓库历史**（fork 时拉取的快照）。若本地尚未 fetch 过 upstream，请先 `git fetch upstream` 再比对。

## 索引表

| 序号 | 译文文件 | 对应原文 | 原文基准 (commit@日期) | 类型 | 文档介绍 |
|---|---|---|---|---|---|
| 1 | [.devcontainer/image-builder/custom-README.md](.devcontainer/image-builder/custom-README.md) | [.devcontainer/image-builder/README.md](.devcontainer/image-builder/README.md) | 7b3b02b@2025-10-24 | 翻译 | image-builder 开发容器说明 |
| 2 | [attestation-service/custom-README.md](attestation-service/custom-README.md) | [attestation-service/README.md](attestation-service/README.md) | 49758e9@2025-11-20 | 翻译 | AS 组件总说明：支持的 TEE、两步校验、校验器驱动、策略引擎与 RVPS |
| 3 | [attestation-service/docs/custom-amd-offline-certificate-cache.md](attestation-service/docs/custom-amd-offline-certificate-cache.md) | [attestation-service/docs/amd-offline-certificate-cache.md](attestation-service/docs/amd-offline-certificate-cache.md) | 1c82ce8@2026-09-04 | 翻译 | AMD VCEK 离线证书存储指南（隔离网络场景，Helm/Compose 两种部署） |
| 4 | [attestation-service/docs/custom-attestation_token.md](attestation-service/docs/custom-attestation_token.md) | [attestation-service/docs/attestation_token.md](attestation-service/docs/attestation_token.md) | 44b8bca@2026-05-26 | 翻译 | 证明令牌（EAR）结构、ReportData/InitData 与硬件声明项说明 |
| 5 | [attestation-service/docs/custom-config.md](attestation-service/docs/custom-config.md) | [attestation-service/docs/config.md](attestation-service/docs/config.md) | e26c575@2026-06-24 | 翻译 | CoCo AS 配置文件参考：全局属性、令牌代理、RVPS、各校验器与存储后端配置 |
| 6 | [attestation-service/docs/custom-grpc-as.md](attestation-service/docs/custom-grpc-as.md) | [attestation-service/docs/grpc-as.md](attestation-service/docs/grpc-as.md) | 5f5e784@2026-07-08 | 翻译 | gRPC 证明服务：支持证据类型、快速开始、源码/镜像构建与 JWKS API |
| 7 | [attestation-service/docs/custom-policy.md](attestation-service/docs/custom-policy.md) | [attestation-service/docs/policy.md](attestation-service/docs/policy.md) | 3ed9b84@2026-10-16 | 翻译 | 证明策略（Rego）规范：信任声明项、扩展、策略查找规则与示例 |
| 8 | [attestation-service/docs/custom-restful-as.md](attestation-service/docs/custom-restful-as.md) | [attestation-service/docs/restful-as.md](attestation-service/docs/restful-as.md) | ef211d7@2026-07-08 | 翻译 | RESTful 证明服务：快速开始、构建运行、HTTPS 与各端点载荷格式 |
| 9 | [attestation-service/docs/custom-tcb_claims.md](attestation-service/docs/custom-tcb_claims.md) | [attestation-service/docs/tcb_claims.md](attestation-service/docs/tcb_claims.md) | 3873be2@2026-08-25 | 翻译 | TCB 声明项参考：各 TEE（TDX/SGX/SNP/CCA/CSV/SE/TPM/NVIDIA 等）字段与 UEFI 事件日志 |
| 10 | [custom-AGENTS.md](custom-AGENTS.md) | [AGENTS.md](AGENTS.md) | 48b5d46@2026-08-11 | 翻译 | 贡献者指南：架构约定、构建/测试/lint 流程与提交、PR 规范 |
| 11 | [custom-DEVELOPMENT.md](custom-DEVELOPMENT.md) | [DEVELOPMENT.md](DEVELOPMENT.md) | 7b3b02b@2025-10-24 | 翻译 | 开发容器（devcontainer）使用与维护说明：镜像、features、dependabot |
| 12 | [custom-README.md](custom-README.md) | [README.md](README.md) | e865b87@2026-08-27 | 翻译 | 仓库总览：Trustee 组成、架构图、部署方式与许可证 |
| 13 | [custom-release-guide.md](custom-release-guide.md) | [release-guide.md](release-guide.md) | f107bc0@2025-02-05 | 翻译 | KBS 发布流程：候选确认、ghcr 镜像拷贝、kustomization 提 PR、打 tag |
| 14 | [deployment/helm-chart/custom-README.md](deployment/helm-chart/custom-README.md) | [deployment/helm-chart/README.md](deployment/helm-chart/README.md) | 7001db2@2026-10-20 | 翻译 | Helm Chart 参数与部署说明（生成文件的人工可读快照） |
| 15 | [deployment/helm-chart/custom-README.md.gotmpl](deployment/helm-chart/custom-README.md.gotmpl) | [deployment/helm-chart/README.md.gotmpl](deployment/helm-chart/README.md.gotmpl) | 7001db2@2026-10-20 | 翻译 | Helm Chart README 生成模板译文（{{...}} 原样保留） |
| 16 | [deps/key-value-storage/custom-README.md](deps/key-value-storage/custom-README.md) | [deps/key-value-storage/README.md](deps/key-value-storage/README.md) | cda6545@2026-05-26 | 翻译 | 键值存储抽象总说明与统一存储后端配置格式 |
| 17 | [deps/key-value-storage/src/local_fs/custom-README.md](deps/key-value-storage/src/local_fs/custom-README.md) | [deps/key-value-storage/src/local_fs/README.md](deps/key-value-storage/src/local_fs/README.md) | 9146cf8@2025-12-18 | 翻译 | LocalFs 存储后端配置说明 |
| 18 | [deps/key-value-storage/src/local_json/custom-README.md](deps/key-value-storage/src/local_json/custom-README.md) | [deps/key-value-storage/src/local_json/README.md](deps/key-value-storage/src/local_json/README.md) | ccb12ec@2025-12-18 | 翻译 | LocalJson 存储后端配置说明 |
| 19 | [deps/key-value-storage/src/postgres/custom-README.md](deps/key-value-storage/src/postgres/custom-README.md) | [deps/key-value-storage/src/postgres/README.md](deps/key-value-storage/src/postgres/README.md) | 044691d@2026-02-25 | 翻译 | PostgreSQL 存储后端配置说明 |
| 20 | [deps/key-value-storage/src/redis/custom-README.md](deps/key-value-storage/src/redis/custom-README.md) | [deps/key-value-storage/src/redis/README.md](deps/key-value-storage/src/redis/README.md) | c9d9f28@2026-08-07 | 翻译 | Redis 存储后端配置说明 |
| 21 | [deps/verifier/src/cca/custom-README.md](deps/verifier/src/cca/custom-README.md) | [deps/verifier/src/cca/README.md](deps/verifier/src/cca/README.md) | f3243bb@2025-06-17 | 翻译 | Arm CCA 校验器说明 |
| 22 | [deps/verifier/src/nvidia/custom-README.md](deps/verifier/src/nvidia/custom-README.md) | [deps/verifier/src/nvidia/README.md](deps/verifier/src/nvidia/README.md) | 248684d@2026-07-07 | 翻译 | NVIDIA GPU 校验器说明 |
| 23 | [deps/verifier/src/se/custom-README.md](deps/verifier/src/se/custom-README.md) | [deps/verifier/src/se/README.md](deps/verifier/src/se/README.md) | 5564a4f@2026-05-27 | 翻译 | IBM Secure Execution 校验器说明 |
| 24 | [deps/verifier/src/tpm/custom-README.md](deps/verifier/src/tpm/custom-README.md) | [deps/verifier/src/tpm/README.md](deps/verifier/src/tpm/README.md) | 2f87a43@2026-05-13 | 翻译 | TPM 校验器说明 |
| 25 | [deps/verifier/test_data/cca/conf/custom-README.md](deps/verifier/test_data/cca/conf/custom-README.md) | [deps/verifier/test_data/cca/conf/README.md](deps/verifier/test_data/cca/conf/README.md) | bcd7a79@2025-04-09 | 翻译 | CCA 校验器测试数据配置：local/remote 两种校验器配置项 |
| 26 | [kbs/config/kubernetes/custom-README.md](kbs/config/kubernetes/custom-README.md) | [kbs/config/kubernetes/README.md](kbs/config/kubernetes/README.md) | 6864c15@2026-05-06 | 翻译 | KBS Kubernetes 部署清单说明 |
| 27 | [kbs/custom-README.md](kbs/custom-README.md) | [kbs/README.md](kbs/README.md) | 7001db2@2026-10-20 | 翻译 | KBS 组件说明：后台检查/护照两种模式、构建、外部插件、存储与参考链接 |
| 28 | [kbs/custom-quickstart.md](kbs/custom-quickstart.md) | [kbs/quickstart.md](kbs/quickstart.md) | 03d215a@2026-06-17 | 翻译 | KBS 快速开始：本地部署、配置与端到端验证 |
| 29 | [kbs/docs/custom-admin.md](kbs/docs/custom-admin.md) | [kbs/docs/admin.md](kbs/docs/admin.md) | 74c7d62@2026-07-08 | 翻译 | KBS 管理员 API 与认证方式说明 |
| 30 | [kbs/docs/custom-attestation_token_verification.md](kbs/docs/custom-attestation_token_verification.md) | [kbs/docs/attestation_token_verification.md](kbs/docs/attestation_token_verification.md) | 03d215a@2026-06-17 | 翻译 | 证明令牌校验：信任锚配置与各证明服务对接指引 |
| 31 | [kbs/docs/custom-cluster.md](kbs/docs/custom-cluster.md) | [kbs/docs/cluster.md](kbs/docs/cluster.md) | b62c94e@2026-08-12 | 翻译 | docker compose 集群搭建与使用说明 |
| 32 | [kbs/docs/custom-confidential-containers-eventlog.md](kbs/docs/custom-confidential-containers-eventlog.md) | [kbs/docs/confidential-containers-eventlog.md](kbs/docs/confidential-containers-eventlog.md) | 673be30@2026-10-08 | 翻译 | 证明代理事件日志（AAEL）与 CoCo 事件规范 |
| 33 | [kbs/docs/custom-config.md](kbs/docs/custom-config.md) | [kbs/docs/config.md](kbs/docs/config.md) | 99d35c2@2026-10-16 | 翻译 | KBS 配置文件完整参考 |
| 34 | [kbs/docs/custom-ext_plugin.md](kbs/docs/custom-ext_plugin.md) | [kbs/docs/ext_plugin.md](kbs/docs/ext_plugin.md) | 3e16000@2026-10-16 | 翻译 | 外部 gRPC 插件：架构、认证模型、协议、部署、监控与排障 |
| 35 | [kbs/docs/custom-initdata.md](kbs/docs/custom-initdata.md) | [kbs/docs/initdata.md](kbs/docs/initdata.md) | 79722e7@2024-08-09 | 翻译 | Initdata 规范：数据结构、各 TEE 完整性绑定与摘要计算 |
| 36 | [kbs/docs/custom-kbs_attestation_protocol.md](kbs/docs/custom-kbs_attestation_protocol.md) | [kbs/docs/kbs_attestation_protocol.md](kbs/docs/kbs_attestation_protocol.md) | 3e16000@2026-10-16 | 翻译 | KBS 证明协议（RCAR）：载荷格式、HTTP 集成、错误类型与 OpenAPI |
| 37 | [kbs/docs/custom-metrics.md](kbs/docs/custom-metrics.md) | [kbs/docs/metrics.md](kbs/docs/metrics.md) | b13894b@2026-01-17 | 翻译 | KBS Prometheus 指标清单与抓取配置示例 |
| 38 | [kbs/docs/custom-rate-limiting.md](kbs/docs/custom-rate-limiting.md) | [kbs/docs/rate-limiting.md](kbs/docs/rate-limiting.md) | 7001db2@2026-10-20 | 翻译 | 速率限制为何应放在 KBS 前置代理上 |
| 39 | [kbs/docs/custom-resource_storage_backend.md](kbs/docs/custom-resource_storage_backend.md) | [kbs/docs/resource_storage_backend.md](kbs/docs/resource_storage_backend.md) | 338610f@2026-08-27 | 翻译 | 资源存储后端：本地文件系统、阿里云 KMS、AWS/GCP Secret Manager、Vault |
| 40 | [kbs/docs/custom-self-signed-https.md](kbs/docs/custom-self-signed-https.md) | [kbs/docs/self-signed-https.md](kbs/docs/self-signed-https.md) | 5564a4f@2026-05-27 | 翻译 | 使用自签名证书启用 KBS HTTPS 的实操步骤 |
| 41 | [kbs/docs/custom-vault_kv.md](kbs/docs/custom-vault_kv.md) | [kbs/docs/vault_kv.md](kbs/docs/vault_kv.md) | 71d4c48@2026-05-11 | 翻译 | Vault / OpenBao KV v1 资源后端：配置、路径映射、令牌策略与测试 |
| 42 | [kbs/docs/plugins/custom-nebula_ca.md](kbs/docs/plugins/custom-nebula_ca.md) | [kbs/docs/plugins/nebula_ca.md](kbs/docs/plugins/nebula_ca.md) | 7787541@2025-02-18 | 翻译 | Nebula CA 插件配置与使用 |
| 43 | [kbs/docs/plugins/custom-pkcs11.md](kbs/docs/plugins/custom-pkcs11.md) | [kbs/docs/plugins/pkcs11.md](kbs/docs/plugins/pkcs11.md) | f7f88c5@2025-03-19 | 翻译 | PKCS#11 插件配置与使用 |
| 44 | [kbs/docs/plugins/custom-spire.md](kbs/docs/plugins/custom-spire.md) | [kbs/docs/plugins/spire.md](kbs/docs/plugins/spire.md) | 99d35c2@2026-10-16 | 翻译 | SPIRE 插件配置与使用 |
| 45 | [kbs/sample_policies/custom-README.md](kbs/sample_policies/custom-README.md) | [kbs/sample_policies/README.md](kbs/sample_policies/README.md) | 338610f@2026-08-27 | 翻译 | 示例策略文件清单（allow_all/deny_all/affirming） |
| 46 | [kbs/src/plugins/implementations/provisioner/custom-README.md](kbs/src/plugins/implementations/provisioner/custom-README.md) | [kbs/src/plugins/implementations/provisioner/README.md](kbs/src/plugins/implementations/provisioner/README.md) | 28dc9be@2026-08-14 | 翻译 | Provisioner 插件：LUKS 密钥供给流程、响应格式与认证路径 |
| 47 | [rvps/custom-README.md](rvps/custom-README.md) | [rvps/README.md](rvps/README.md) | dfaab42@2026-05-27 | 翻译 | RVPS 组件说明：架构、消息流、运行方式、配置与客户端工具 |
| 48 | [rvps/docs/custom-reference-value-uri.md](rvps/docs/custom-reference-value-uri.md) | [rvps/docs/reference-value-uri.md](rvps/docs/reference-value-uri.md) | bb6567b@2026-03-10 | 翻译 | 参考值 URI 规范（rvps:///namespace[:tag]）与 CoCo 命名约定 |
| 49 | [rvps/src/extractors/corim/custom-README.md](rvps/src/extractors/corim/custom-README.md) | [rvps/src/extractors/corim/README.md](rvps/src/extractors/corim/README.md) | b47b9b9@2026-04-30 | 翻译 | CoRIM 提取器：CoMID 到参考值 URI 的映射规则 |
| 50 | [rvps/src/extractors/sample/custom-README.md](rvps/src/extractors/sample/custom-README.md) | [rvps/src/extractors/sample/README.md](rvps/src/extractors/sample/README.md) | f95bb04@2025-09-02 | 翻译 | Sample 提取器（仅测试/演示）及其来源格式 |
| 51 | [rvps/src/extractors/swid/custom-README.md](rvps/src/extractors/swid/custom-README.md) | [rvps/src/extractors/swid/README.md](rvps/src/extractors/swid/README.md) | f95bb04@2025-09-02 | 翻译 | SWID RIM 提取器说明与限制 |
| 52 | [tools/kbs-client/custom-README.md](tools/kbs-client/custom-README.md) | [tools/kbs-client/README.md](tools/kbs-client/README.md) | e908c2d@2026-05-22 | 翻译 | KBS 客户端工具（kbs-client）说明 |

## 统计

- 翻译类：`52` 份（覆盖仓库内全部原始 `.md` 文档与 Helm README 模板）
- 个人新增：`0` 份
- 合计：`52` 份 `custom-` 文件

## 维护约定

- 新增译文：同目录建 `custom-<原名>`，并在本表追加一行（填原文路径 + 原文基准 commit/日期）。
- 上游原文更新后：更新对应译文，并把「原文基准」刷新为该原文最新 commit。
- 删除某原文时：同步删除其 `custom-` 译文与本表行。
- 翻译边界：代码块、命令、URL、相对链接目标、配置键、JSON/YAML/TOML、API 路径与模板变量一律保持原样；
  为保持跨文档锚点可用，必要时在译文标题前保留 `<a id="原英文锚点"></a>`。
