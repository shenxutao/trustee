# TCB 声明项

> 说明：本文件是 `attestation-service/docs/tcb_claims.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

以下声明项将由相应的校验器从硬件证据中提取出来。

这些声明项会暴露给证明服务的策略，并被包含在证明令牌中。

## 声明项数据格式

策略的输入是一个由校验器输出组合而成的 JSON 对象。大多数情况下，它遵循如下形态：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `<tee-name>` | object | 与具体 TEE 相关的声明项（例如：`tdx`、`sgx`、`snp`、`se`、`tpm`、`nvidia`、`hygondcu`、`az-tdx-vtpm`、`az-snp-vtpm`、`sample`、`csv`、`cca`） |
| `report_data` | string | 从证据中提取的十六进制/base64 编码的 report data（格式取决于校验器） |
| `init_data` | string | 从证据中提取的十六进制/base64 编码的 init-data 哈希（在受支持时） |
| `init_data_claims` | object | 解析后的 init-data 声明项（在提供且校验通过 init-data 时存在） |
| `runtime_data_claims` | object | 解析后的 runtime-data 声明项（在提供且校验通过 report-data 时存在） |

`<tee-name>` 与序列化后的 [`Tee`](https://docs.rs/kbs-types/latest/kbs_types/enum.Tee.html) 变体字符串一致（例如 `az-tdx-vtpm`，而非 `az_tdx_vtpm`）。在 Rego 中，带连字符的键必须使用方括号语法，例如 `input["az-tdx-vtpm"]`。

最小化示例：

```json
{
  "tdx": {
    "quote": {
      "body": {
        "mr_td": "<hex>"
      }
    }
  },
  "report_data": "<hex>",
  "init_data": "<hex>",
  "init_data_claims": {},
  "runtime_data_claims": {}
}
```

## Sample（示例校验器）

**这只是一个测试用校验器**。
- `sample.svn`：字符串。sample 证据的版本信息。
- `sample.report_data`：字符串（Base64 标准编码）。生成证据时的 report data。
- `sample.init_data`：字符串（Base64 标准编码）。Init data 哈希。
- `sample.launch_digest`：字符串。用于策略测试的虚拟启动摘要。恒为 `abcde`。
- `sample.platform_version.major`：数值。Sample 平台主版本号。恒为 `1`。
- `sample.platform_version.minor`：数值。Sample 平台次版本号。恒为 `4`。
- `sample.debug`：布尔值。Sample 调试标志（在 sample 校验器中恒为 false）。恒为 `false`。

## Sample Device（示例设备校验器）

**这只是一个用于设备类别证明的测试用校验器**。
- `sampledevice.svn`：字符串。sample 设备证据的版本信息。
- `sampledevice.report_data`：字符串（Base64 标准编码）。生成证据时的 report data。

## Intel TDX（英特尔 TDX）

<a id="intel-tdx"></a>

- `tdx.uefi_event_logs`：**可选**。从 ccel 日志文件解析出的对象列表。是否出现取决于是否存在 CCEL。结构与定义参见 [UEFI Eventlog](#uefi-eventlog)。

以下字段始终存在。
- `tdx.quote.header`：对象。TDX Quote 头部。
  - `tdx.quote.header.version`：字符串（十六进制）。Quote 格式版本。目前支持 `0400`（V4 版本 quote 格式）与 `0500`（V5 版本 quote 格式）。
  - `tdx.quote.header.att_key_type`：字符串（2 字节十六进制）。签名所使用算法的枚举。
  - `tdx.quote.header.tee_type`：字符串（4 字节十六进制）。TDX 恒为 `81000000`。
  - `tdx.quote.header.reserved`：字符串（4 字节十六进制）。保留字段。
  - `tdx.quote.header.vendor_id`：字符串（16 字节十六进制）。QE 厂商的 UID。QE 是 TEE 内部一个经过签名的软件组件，用于协助生成 tdx quote。
  - `tdx.quote.header.user_data`：字符串（20 字节十六进制）。自定义的证明密钥所有者数据。
- `tdx.quote.body`：对象。TDX Quote 主体。
  - `tdx.quote.body.mr_config_id`：字符串（48 字节十六进制）。用于 guest TD 中非所有者定义的配置的软件自定义 ID，例如运行时或操作系统配置。
  - `tdx.quote.body.mr_owner`：字符串（48 字节十六进制）。guest TD 所有者的软件自定义 ID。
  - `tdx.quote.body.mr_owner_config`：字符串（48 字节十六进制）。用于 guest TD 中所有者定义的配置的软件自定义 ID，例如与工作负载相关而非与运行时或操作系统相关的配置。
  - `tdx.quote.body.mr_td`：字符串（48 字节十六进制）。TD 初始内容的度量值。
  - `tdx.quote.body.mrsigner_seam`：字符串（48 字节十六进制）。第三方 tdx-module 签名者的度量值（SHA384 哈希）。若为 0，则表示该 tdx-module 来自 Intel。
  - `tdx.quote.body.report_data`：字符串（64 字节十六进制）。用于 guest TD 上非所有者定义配置的软件自定义 ID。
  - `tdx.quote.body.seam_attributes`：字符串（8 字节十六进制）。对于 tdx 1.0，必须为 `000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000`。
  - `tdx.quote.body.td_attributes`：字符串（8 字节十六进制）。TD 的属性。另见 `tdx.td_attributes`。
  - `tdx.quote.body.mr_seam`：字符串（48 字节十六进制）。SEAM 模块的度量值。
  - `tdx.quote.body.tcb_svn`：字符串（16 字节十六进制）。TEE 硬件 tcb 版本，由 Intel 定义并具有含义。每次固件更新都会改变该字段。
  - `tdx.quote.body.xfam`：字符串（8 字节十六进制）。TD 的 XFAM。
  - `tdx.quote.body.rtmr_0`：字符串（48 字节十六进制）。运行时可扩展度量寄存器 0。
  - `tdx.quote.body.rtmr_1`：字符串（48 字节十六进制）。运行时可扩展度量寄存器 1。
  - `tdx.quote.body.rtmr_2`：字符串（48 字节十六进制）。运行时可扩展度量寄存器 2。
  - `tdx.quote.body.rtmr_3`：字符串（48 字节十六进制）。运行时可扩展度量寄存器 3。
  - `tdx.quote.body.tee_tcb_svn2`：字符串（16 字节十六进制）。TEE TCB SVN 数组（用于 TD Preserving）。
  - `tdx.quote.body.mr_servicetd`：字符串（48 字节十六进制）。如果存在一个或多个已绑定或预绑定的 service TD，该字段是这些已绑定 service TD 的 `TDINFO` 的 SHA384 哈希；否则该字段为 0。
- `tdx.quote.type`：**可选**。字符串（2 字节十六进制）。指示 quote v5 的类型。`0200` 表示 TDX 1.0 quote，`0300` 表示 TDX 1.5 quote。只有 V5 quote 格式包含该字段。
- `tdx.quote.size`：**可选**。字符串（4 字节十六进制）。Quote 主体长度。只有 V5 quote 格式包含该字段。
- `tdx.td_attributes`：对象。`tdx.quote.body.td_attributes` 翻译后的标志位。
  - `tdx.td_attributes.debug`：布尔值。指示 TD 是否运行在 TD 调试模式（置 1）或不是（置 0）。在 TD 调试模式下，CPU 状态与私有内存可被宿主 VMM 访问。
  - `tdx.td_attributes.key_locker`：布尔值。指示该 TD 是否被允许使用 Key Locker。
  - `tdx.td_attributes.perfmon`：布尔值。指示该 TD 是否被允许使用 Perfmon 与 PERF_METRICS 能力。
  - `tdx.td_attributes.protection_keys`：布尔值。指示该 TD 是否被允许使用 Supervisor Protection Keys。
  - `tdx.td_attributes.septve_disable`：布尔值。决定在 TD 访问 PENDING 页面时是否禁用把 EPT violation 转换为 #VE。
- `tdx.advisory_ids`：字符串列表。Intel® Product Security Center Advisories。
- `tdx.collateral_expiration_status`：字符串。如果与输入的 `expiration_check_date` 相比没有任何 collateral 过期，则应为 `"0"`。
- `tdx.earliest_expiration_date`：字符串。RFC3339 格式的日期时间值 —— 所有 collateral 中最早的 nextUpdate 值（即过期日期）。
- `tdx.earliest_issue_date`：字符串。RFC3339 格式的日期时间值 —— 所有 collateral 中最早的 issueDate。
- `tdx.is_cached_keys`：布尔值。指示平台根密钥是否被 SGX Registration Backend 缓存。_注意：该字段仅在 sgx_type 设为 "scalable" 或 "Scalable with Integrity" 时提供。_
- `tdx.is_dynamic_platform`：布尔值。指示平台是否可以通过额外软件包进行扩展。_注意：该字段仅在 sgx_type 设为 "scalable" 或 "Scalable with Integrity" 时提供。_
- `tdx.is_smt_enabled`：布尔值。指示平台是否启用了 SMT（同步多线程）。_注意：该字段仅在 sgx_type 设为 "scalable" 或 "Scalable with Integrity" 时提供。_
- `tdx.latest_issue_date`：字符串。RFC3339 格式的日期时间值 —— 所有 collateral 中最晚的 issueDate 值。
- `tdx.pck_crl_num`：数值。指示所用 PCK 证书的新鲜度。
- `tdx.platform_instance_id`：**可选**。字符串（16 字节十六进制）。仅在证书由 PCK Platform CA 签发时出现；当证书由 PCK Processor CA 签发时不存在。这些证书是把平台唯一硬件标识与其 TCB 关联起来的 PCK 证书。
- `tdx.platform_provider_id`：字符串（16 字节十六进制）。平台供应 ID（PPID）。
- `tdx.root_ca_crl_num`：数值。指示所用根 CA 证书的新鲜度。
- `tdx.root_key_id`：字符串（48 字节十六进制）。collateral 根签名者的 ID（根 CA 公钥的 SHA-384 哈希）。
- `tdx.sgx_type`：字符串。SGX 中所使用的内存类型。可以取（`Standard`、`Scalable`、`Scalable with Integrity`）之一。
- `tdx.tcb_date`：字符串。RFC3339 格式的日期时间值 —— tcbInfo 与 qeIdentity 中最早的日期。
- `tdx.tcb_eval_num`：数值。指示所用参考值的新鲜度。
- `tdx.tcb_status`：字符串。TCB 等级状态，可以取以下任一值：
  - `UpToDate` —— 被证明平台已打上最新的固件与软件补丁，且不存在适用的已知安全公告。
  - `SWHardeningNeeded` —— 平台固件与软件处于最新的安全补丁级别，但存在只能通过修改 enclave 或 TD 软件来缓解的漏洞。
  - `ConfigurationNeeded` —— 平台固件与软件处于最新的安全补丁级别，但需要修改平台硬件配置以缓解漏洞。
  - `ConfigurationAndSWHardeningNeeded` —— 该状态是 `SWHardeningNeeded` 与 `ConfigurationNeeded` 的组合。
  - `OutOfDate` —— 被证明平台的软件和/或固件未按照最新的 TCB Recovery（TCB-R）打补丁。
  - `OutOfDateConfigurationNeeded` —— 被证明平台未按照最新的 TCB-R 打补丁。需要进行硬件配置。
  - `TDRelaunchAdvised` —— 平台固件与软件处于最新的安全补丁级别，但该 TD 是在通过 TD Preserving 更新应用新的 TDX TCB 组件之前启动的。重新启动该 TD 会改变证明结果。
  - `TDRelaunchAdvisedConfigurationNeeded` —— 平台固件与软件处于最新的安全补丁级别，但存在可能使该 TD 暴露于漏洞的平台硬件配置。重新启动该 TD 会改变证明结果。

以下字段报告的是 TD 当前的 TCB 等级，而不是它最初启动时的等级（后者由上面的 `tdx.tcb_date`/`tdx.tcb_status`/`tdx.advisory_ids` 反映）。只有对于经历过 TD Preserving 更新的 TD，它们才与启动时的字段不同。_注意：这些字段仅在所安装的 DCAP quote 校验库报告 supplemental data 的 minor_version 为 5 或更高（DCAP 1.27+）时提供；较老的库会完全省略它们。_
- `tdx.tcb_date_current`：字符串。RFC3339 格式的日期时间值 —— 当前 TCB 等级的 TCB 日期。
- `tdx.tcb_status_current`：字符串。当前 TCB 等级的 TCB 等级状态。取值集合与 `tdx.tcb_status` 相同。
- `tdx.advisory_ids_current`：字符串列表。适用于当前 TCB 等级的 Intel® Product Security Center Advisories。

## Intel SGX（英特尔 SGX）

- `sgx.header`：对象。SGX quote 头部。
  - `sgx.header.version`：字符串（2 字节十六进制）。该 quote 结构的版本。
  - `sgx.header.att_key_type`：字符串（2 字节十六进制）。sgx_attestation_algorithm_id_t。描述签名类型。
  - `sgx.header.att_key_data_0`：字符串（4 字节十六进制）。可信执行环境的类型。对 SGX 恒为 `00000000`。
  - `sgx.header.qe_svn`：字符串（2 字节十六进制）。生成 quote 时 Quoting Enclave 的 ISV_SVN。
  - `sgx.header.pce_svn`：字符串（2 字节十六进制）。生成 quote 时 PCE 的 ISV_SVN。
  - `sgx.header.vendor_id`：字符串（16 字节十六进制）。QE 厂商的唯一标识。
  - `sgx.header.user_data`：字符串（20 字节十六进制）。自定义的证明密钥所有者数据。
- `sgx.body`：对象。SGX quote 主体。
  - `sgx.body.cpu_svn`：字符串（16 字节十六进制）。CPU 的安全版本。
  - `sgx.body.misc_select`：字符串（4 字节十六进制）。SSA.MISC 中定义的哪些字段。
  - `sgx.body.reserved1`：字符串（12 字节十六进制）。保留字段。
  - `sgx.body.isv_ext_prod_id`：字符串（16 字节十六进制）。ISV 分配的扩展产品 ID。
  - `sgx.body.attributes.flags`：字符串（8 字节十六进制）。Enclave 所拥有的特殊能力。
  - `sgx.body.attributes.xfrm`：字符串（8 字节十六进制）。Enclave 所拥有的 XFRM。
  - `sgx.body.mr_enclave`：字符串（32 字节十六进制）。Enclave 的 ENCLAVE 度量值。
  - `sgx.body.reserved2`：字符串（32 字节十六进制）。保留字段。
  - `sgx.body.mr_signer`：字符串（32 字节十六进制）。Enclave 的 SIGNER 度量值。
  - `sgx.body.reserved3`：字符串（32 字节十六进制）。保留字段。
  - `sgx.body.config_id`：字符串（64 字节十六进制）。Enclave 的 CONFIGID。
  - `sgx.body.isv_prod_id`：字符串（2 字节十六进制）。Enclave 的产品 ID。
  - `sgx.body.isv_svn`：字符串（2 字节十六进制）。Enclave 的安全版本。
  - `sgx.body.config_svn`：字符串（2 字节十六进制）。Enclave 的 CONFIGSVN。
  - `sgx.body.reserved4`：字符串（42 字节十六进制）。保留字段。
  - `sgx.body.isv_family_id`：字符串（16 字节十六进制）。ISV 分配的 Family ID。
  - `sgx.body.report_data`：字符串（64 字节十六进制）。由用户提供的数据。
- `sgx.platform_instance_id`：**可选**。字符串（16 字节十六进制）。仅在证书由 PCK Platform CA 签发时出现；当证书由 PCK Processor CA 签发时不存在。这些证书是把平台唯一硬件标识与其 TCB 关联起来的 PCK 证书。

## Azure TDX 机密虚拟机（az-tdx-vtpm）

<a id="az-tdx-vtpm"></a>

- `["az-tdx-vtpm"].*`：声明项继承自 [TDX 布局](#intel-tdx)。
- `["az-tdx-vtpm"].tpm`：对象。TPM PCR 值。
  - `["az-tdx-vtpm"].tpm.pcr{01,..,n}`：字符串（十六进制）。该 TEE 的 vTPM quote 的 SHA256 PCR 寄存器。
  - `["az-tdx-vtpm"].tpm.init_data`：**可选**。字符串（十六进制）。用作 initdata 摘要的寄存器（PCR 8）。

> [!NOTE]
> 在该 TEE 中，TD Report 与 TD Quote 是在早期启动阶段获取的。内核、Initrd 与 rootfs 都会被度量进 vTPM 的寄存器中。

## Azure SEV-SNP 机密虚拟机（az-snp-vtpm）

- `["az-snp-vtpm"].*`：声明项继承自 [SEV-SNP](#amd-sev-snp) 布局。
- `["az-snp-vtpm"].tpm`：对象。TPM PCR 值。
  - `["az-snp-vtpm"].tpm.pcr{01,..,n}`：字符串（十六进制）。该 TEE 的 vTPM quote 的 SHA256 PCR 寄存器。
  - `["az-snp-vtpm"].tpm.init_data`：**可选**。字符串（十六进制）。用作 initdata 摘要的寄存器（PCR 8）。
- `["az-snp-vtpm"].uefi_event_logs`：**可选**。逐条目解析后的 AAEL（Attestation Agent Eventlog）条目，当 guest 记录了运行时度量（例如容器镜像拉取）时存在。在对外暴露之前，会先重放并针对每条事件实际对应的 PCR 寄存器进行匹配。

注意：在该 TEE 中，TD Report 与 TD Quote 是在早期启动阶段获取的。内核、Initrd 与 rootfs 都会被度量进 vTPM 的寄存器中。

## IBM Secure Execution for Linux（SEL，IBM Z 安全执行）

- `se.version`：数值。该 quote 结构的版本。
- `se.cuid`：字符串（16 字节十六进制）。被证明 guest 的唯一 ID（配置唯一 ID）。
- `se.tag`：字符串（16 字节十六进制）。SE 头部 tag。
- `se.image_phkh`：字符串（十六进制）。SE 镜像公钥宿主密钥哈希。
- `se.attestation_phkh`：字符串（十六进制）。SE 证明公钥宿主密钥哈希。
- `se.report_data`：字符串（64 字节十六进制）。由用户提供的数据。

## Arm CCA（Arm 机密计算架构）

CCA 声明项被分组为 `cca.realm` 与 `cca.platform`：

- `cca.realm`：对象。CCA Realm 令牌声明项。
  - `cca.realm.cca_realm_personalization_value`：字符串（Base64 标准编码）。每个 Realm 自定义的个性化值。
  - `cca.realm.cca_realm_initial_measurement`：字符串（Base64 标准编码）。Realm 的初始度量值。
  - `cca.realm.cca_realm_extensible_measurements`：字符串列表。Realm 的可扩展度量值。
    - `cca.realm.cca_realm_extensible_measurements[i]`：字符串（Base64 标准编码）。Realm 的第 i 个可扩展度量值。
  - `cca.realm.cca_realm_hash_algo_id`：字符串（Base64 标准编码）。RMI 哈希算法。
  - `cca.realm.cca_realm_challenge`：字符串（Base64 标准编码）。用于执行证明的 challenge。
- `cca.platform`：对象。CCA Platform 令牌声明项。
  - `cca.platform.cca_platform_instance_id`：字符串（Base64 标准编码）。硬件平台实例 ID。
  - `cca.platform.cca_platform_implementation_id`：字符串（Base64 标准编码）。硬件实现 ID。
- `cca.report_data`：字符串（Base64 标准编码）。由 realm challenge 派生出的 report data。
- `cca.init_data`：字符串（Base64 标准编码）。init data 摘要字段，与 `cca.realm.cca_realm_personalization_value` 相同。

## NVIDIA（英伟达 GPU）

本地校验器仅支持 Hopper，并返回以下声明项。

- `arch`：设备架构。仅支持 `Hopper`
- `measurements`：度量值及其各自索引的列表
- `uuid`：设备 UUID
- `config.board_id`：Board ID
- `config.chip_sku`：Chip SKU（库存单位）
- `config.chip_sku_mod`：Chip SKU mod
- `config.cpr_info`：Compute Protected Region 信息
- `config.driver_version`：NVIDIA 驱动版本
- `config.fwid`：固件 ID。可在报告与签名证书中找到
- `config.gpu_info`：GPU 信息
- `config.measurement_count`：`measurements` 中每一项对应一个 measurement_count。每个 measurement_count 表示相应度量值被扩展到当前取值所经历的次数
- `config.nvdec0_status`：NVIDIA 解码器状态
- `config.project`：Project
- `config.project_sku`：Project SKU
- `config.project_sku_mod`：Project SKU mod
- `config.protected_pcie_status`：Protected PCIe 状态
- `config.vbios_version`：设备 VBIOS 版本

远端校验器导出来自 NRAS 的声明项，列在[此处](https://docs.nvidia.com/attestation/advanced-documentation/latest/claims-guide/gpu_claims.html)。
使用的是声明项版本 3。总体声明项中的 `x-nvidia-overall-att-result` 会与完整的 detached 声明项集合一并包含。

## NVIDIA DPU（DICE，英伟达 DPU）

NVIDIA DPU 校验器使用来自 BlueField DPU 的 TCG DICE
（Device Identifier Composition Engine）证书链来校验证明证据。

- `nvidia-dpu.device_serial`：来自 DeviceID 证书的设备序列号
- `nvidia-dpu.device_class`：设备类别标识（例如 "bluefield3"），来自 Subject CN
- `nvidia-dpu.firmware_layers_count`：TCG DICE TcbInfo 扩展中 FWID 条目的数量
- `nvidia-dpu.accumulated_fwid`：固件各层摘要拼接后的 SHA-384 哈希（十六进制编码）
- `nvidia-dpu.alias_timestamp`：Alias 证书签发时间戳（unix 秒）

注意：`FirmwareLayer.name` 与 `FirmwareLayer.version` 仅作信息用途 ——
它们不在 TBS 与 report_data_signature 的覆盖范围内，因此不受完整性保护。

## AMD SEV-SNP（AMD SEV 安全嵌套分页）

<a id="amd-sev-snp"></a>

- `snp.chip_id`：平台实例的唯一标识（若被屏蔽，该字段可能为 0）
- `snp.measurement`：覆盖 guest 初始内存的启动摘要
- `snp.platform_smt_enabled`：系统上是否启用了同步多线程
- `snp.platform_tsme_enabled`：系统上是否启用了 Transparent SME
- `snp.policy_abi_major`：guest 所允许的最低 ABI 主版本
- `snp.policy_abi_minor`：guest 所允许的最低 ABI 次版本
- `snp.policy_debug_allowed`：guest 是否允许使用 SNP 调试特性
- `snp.policy_migrate_ma`：是否可以向 guest 连接迁移代理
- `snp.policy_single_socket`：guest 是否只能在单个 socket 上被激活
- `snp.policy_smt_allowed`：guest 是否可以运行在启用了 SMT 的系统上
- `snp.reported_tcb_bootloader`：ASP bootloader 的上报 SVN
- `snp.reported_tcb_microcode`：上报的微码版本
- `snp.reported_tcb_snp`：SNP 固件的上报 SVN
- `snp.reported_tcb_tee`：ASP OS 的上报 SVN

声明项映射中只包含上报的 TCB 版本。
一份 SEV-SNP 证明报告包含四组 TCB 版本信息。
通常这四个值都相同，但有时上报的 TCB 可能落后于真实的固件版本。
这样做是为了在提供方升级到临时固件时，尽量减少策略与证书的变动。
实际固件必须始终新于或等于上报的 TCB。
一般而言，策略应当针对上报的 TCB 进行评估。

## TPM（可信平台模块）

- `tpm.init_data`：字符串（十六进制）。SHA256 PCR[08] 值（十六进制）
- `tpm.report_data`：字符串（十六进制）。来自 quote 的 nonce（十六进制）
- `tpm.pcr00` ... `tpm.pcr23`：字符串（十六进制）。SHA256 PCR 值（十六进制；索引数量取决于 quote）
- `tpm.ak_public`：字符串（Base64 标准编码）。PEM 格式的 AK（证明密钥）公钥（base64 编码的 DER）

## Hygon DCU（海光 DCU）

每份证明报告会在 `hygondcu` 之下产生一组声明项。多个 DCU 会被分别评估（EAR 令牌 submods 为证据列表顺序中的 `dcu0`、`dcu1`、……）：

- `hygondcu.body`：对象。Hygon DCU 的报告主体。
  - `hygondcu.body.version`：数值。固件版本。
  - `hygondcu.body.chip_id`：字符串（16 字节十六进制）。DCU 芯片 ID。
  - `hygondcu.body.user_data`：字符串（64 字节十六进制）。用于证明的 challenge 数据。
  - `hygondcu.body.measure`：字符串（32 字节十六进制）。固件的度量值。
  - `hygondcu.body.reserved`：字符串（128 字节十六进制）。保留字段。
  - `hygondcu.body.sig_usage`：字符串（十六进制）。签名的用途。
  - `hygondcu.body.sig_algo`：字符串（十六进制）。签名所使用的算法。
- `hygondcu.report_data`：字符串（64 字节十六进制）。与 `body.user_data` 取值相同。

## Hygon CSV（海光 CSV）

- `csv.version`：字符串。quote 的版本。目前只有 `1` 和 `2` 合法。
- `csv.policy`：对象。CSV Guest 策略。
  - `csv.policy.nodbg`：数值。禁止调试该 guest。
  - `csv.policy.noks`：数值。禁止与其他 guest 共享密钥。
  - `csv.policy.es`：数值。置位时要求使用 CSV2。
  - `csv.policy.nosend`：数值。禁止把该 guest 发送到另一个平台。
  - `csv.policy.domain`：数值。该 guest 不得被传输到不在该域内的另一个平台。
  - `csv.policy.csv`：数值。该 guest 不得被传输到不具备 CSV 能力的另一个平台。
  - `csv.policy.csv3`：数值。要求使用 CSV3。
  - `csv.policy.asid_reuse`：数值。允许与同一用户拥有的其他 guest 共享 asid。
  - `csv.policy.hsk_version`：数值。该 guest 不得被传输到 HSK 版本更低的另一个平台。
  - `csv.policy.cek_version`：数值。该 guest 不得被传输到 CEK 版本更低的另一个平台。
  - `csv.policy.api_major`：数值。该 guest 不得被传输到平台版本更低的另一个平台。
  - `csv.policy.api_minor`：数值。该 guest 不得被传输到平台版本更低的另一个平台。
- `csv.user_pubkey_digest`：字符串（32 字节十六进制）。用于保护用户/虚拟机监控器与 PSP 之间通信的会话的公钥摘要。
- `csv.vm_id`：字符串（16 字节十六进制）。由 guest 所有者自定义的 VM 标识。
- `csv.vm_version`：字符串（16 字节十六进制）。由 guest 所有者自定义的 VM 版本信息。
- `csv.report_data`：字符串（64 字节十六进制）。用于证明的 challenge 数据。
- `csv.mnonce`：字符串（16 字节十六进制）。由用户生成的随机 nonce，用于保护 struct TeeInfoSigner。
- `csv.measure`：字符串（32 字节十六进制）。VM 的启动摘要。
- `csv.anonce`：字符串（4 字节十六进制）。针对上述字段的签名。
- `csv.sig_usage`：字符串（4 字节十六进制）。签名的用途。
- `csv.sig_algo`：字符串（4 字节十六进制）。签名所使用的算法。
- `csv.serial_number`：字符串。CPU 序列号。

如果 quote 版本为 `2`，还会有以下额外字段。

- `csv.build`：数值。固件构建的版本。
- `csv.rtmr_version`：数值。VM 的 rtmr 版本。
- `csv.reserved0`：字符串（14 字节十六进制）。保留字段，供将来使用。
- `csv.rtmr0`：字符串（32 字节十六进制）。rtmr 寄存器 0，它始终等于 @measure 字段。
- `csv.rtmr1`：字符串（32 字节十六进制）。rtmr 寄存器 1。
- `csv.rtmr2`：字符串（32 字节十六进制）。rtmr 寄存器 2。
- `csv.rtmr3`：字符串（32 字节十六进制）。rtmr 寄存器 3。
- `csv.rtmr4`：字符串（32 字节十六进制）。rtmr 寄存器 4。
- `csv.reserved1`：字符串（656 字节十六进制）。保留字段，供将来使用。

## 附录

### UEFI Eventlog（UEFI 事件日志）

<a id="uefi-eventlog"></a>

#### UEFI Eventlog（UEFI 事件日志） 声明项

UEFI 事件日志是一个事件条目列表。不同的事件条目可以通过 `[]` 运算符索引。
每个条目包含以下字段：

- `uefi_event_logs[i]`：对象。第 i 个事件日志条目。
- `uefi_event_logs[i].index`：数值。度量寄存器索引。
- `uefi_event_logs[i].event_type`：字符串。参见 [Event Types](#event-types) 小节。
- `uefi_event_logs[i].digest_matches_event`：布尔值。摘要数组与事件数据的比较结果。可以在策略中针对以下事件列表检查它们是否受到保护（`EV_EFI_ACTION`、`EV_SEPARATOR`、`EV_EFI_VARIABLE_AUTHORITY`、`EV_EFI_GPT_EVENT`、`EV_EVENT_TAG`、`EV_EFI_VARIABLE_DRIVER_CONFIG`）。
- `uefi_event_logs[i].digests[j]`：对象。该事件的第 j 个摘要数组。
  - `uefi_event_logs[i].digests[j].alg`：哈希算法（`RSA`、`TDES`、`SHA-1`、`SHA-256`、`SHA-384`、`SHA-512`、`SM3`）。
  - `uefi_event_logs[i].digests[j].digest`：字符串（十六进制）。针对前一字段所定义哈希计算出的摘要值。
- `uefi_event_logs[i].event`：字符串（Base64 标准编码）。原始事件数据。
- `uefi_event_logs[i].details`：对象。从事件数据解析出的属性列表。
  - `uefi_event_logs[i].details.string`：**可选**。字符串。解析出的 UTF-8 取值。
  - `uefi_event_logs[i].details.unicode_name`：**可选**。字符串。解析出的度量事件的 Unicode 名称。
  - `uefi_event_logs[i].details.unicode_name_length`：**可选**。数值。度量事件的 Unicode 名称长度。
  - `uefi_event_logs[i].details.variable_data`：**可选**。字符串（Base64 标准编码）。事件变量数据。
  - `uefi_event_logs[i].details.variable_data_length`：**可选**。数值。变量数据的长度。
  - `uefi_event_logs[i].details.variable_name`：**可选**。字符串。变量名称。
  - `uefi_event_logs[i].details.device_paths`：**可选**。字符串列表。解析出的设备路径列表。
    - `uefi_event_logs[i].details.device_paths[j]`：字符串。第 j 个解析出的设备路径。
  - `uefi_event_logs[i].details.data`：**可选**。从事件处理出的附加信息。目前仅用于 [AAEL](../../kbs/docs/confidential-containers-eventlog.md) 条目的 `EV_EVENT_TAG` 事件类型。
    - `uefi_event_logs[i].details.data.domain`：字符串。AAEL 事件域名。
    - `uefi_event_logs[i].details.data.operation`：字符串。AAEL 事件操作名。
    - `uefi_event_logs[i].details.data.content`：字符串或对象。AAEL 事件内容。AS 会尝试把原始 content 字段按 JSON 格式解析。如果成功，此处的 `content` 字段将是一个 JSON 对象；如果失败，它将是一个 JSON 字符串。

#### Event Types（事件类型）

<a id="event-types"></a>

来自 [TCG PC Client Platform Firmware Profile Specification Section 10.4.1](https://trustedcomputinggroup.org/wp-content/uploads/TCG_PCClient_PFP_r1p05_v22_02dec2020.pdf) 的度量事件名称。目前我们支持以下事件类型
- `EV_PREBOOT_CERT`
- `EV_POST_CODE`
- `EV_UNUSED`
- `EV_NO_ACTION`
- `EV_SEPARATOR`
- `EV_ACTION`
- `EV_EVENT_TAG`
- `EV_S_CRTM_CONTENTS`
- `EV_S_CRTM_VERSION`
- `EV_CPU_MICROCODE`
- `EV_PLATFORM_CONFIG_FLAGS`
- `EV_TABLE_OF_DEVICES`
- `EV_COMPACT_HASH`
- `EV_IPL`
- `EV_IPL_PARTITION_DATA`
- `EV_NONHOST_CODE`
- `EV_NONHOST_CONFIG`
- `EV_NONHOST_INFO`
- `EV_OMIT_BOOT_DEVICE_EVENTS`
- `EV_EFI_EVENT_BASE`
- `EV_EFI_VARIABLE_DRIVER_CONFIG`
- `EV_EFI_VARIABLE_BOOT`
- `EV_EFI_BOOT_SERVICES_APPLICATION`
- `EV_EFI_BOOT_SERVICES_DRIVER`
- `EV_EFI_RUNTIME_SERVICES_DRIVER`
- `EV_EFI_GPT_EVENT`
- `EV_EFI_ACTION`
- `EV_EFI_PLATFORM_FIRMWARE_BLOB`
- `EV_EFI_HANDOFF_TABLES`
- `EV_EFI_PLATFORM_FIRMWARE_BLOB2`
- `EV_EFI_HANDOFF_TABLES2`
- `EV_EFI_VARIABLE_BOOT2`
- `EV_EFI_HCRTM_EVENT`
- `EV_EFI_VARIABLE_AUTHORITY`
- `EV_EFI_SPDM_FIRMWARE_BLOB`
- `EV_EFI_SPDM_FIRMWARE_CONFIG`
