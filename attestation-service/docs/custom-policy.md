# 证明策略（开发中）

> 说明：本文件是 `attestation-service/docs/policy.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

CoCo AS 基于 Rego 提供灵活的策略支持，以方便实现自定义的校验规则。

该策略会被执行以生成证明令牌，即 EAR 令牌。

EAR 令牌支持表达能力丰富的策略结果。
策略应当暴露 `data.policy.trust_claims`，并至少设置一个 AR4SI 信任声明项（trust claim），其取值为 `-127` 到 `127` 之间的整数。

受支持的信任声明项键包括：
* `instance-identity`
* `configuration`
* `executables`
* `file-system`
* `hardware`
* `runtime-opaque`
* `storage-opaque`
* `sourced-data`

这些维度可用于返回一份详细但通用的、关于证明方 TCB 的描述。
关于这些信任声明项的更多信息（包括其数值所代表的含义），参见[此处](https://datatracker.ietf.org/doc/draft-ietf-rats-ar4si/)。

策略还可以选择性地定义一个名为 `data.policy.extensions` 的声明项。
通过该声明项指定的任何扩展都会被加入证明令牌。
扩展声明项应包含一个扩展对象数组。
一个扩展对象的形式如下：
```json
{
    "name": "<name-of-extension>",
    "key": "<index-of-extension>",
    "value": ...
}
```

扩展名称应当是抗冲突的公共声明项名称（Collision-Resistant Public Claim Names）。
扩展键应当是一个 i32。
如果该扩展已在 EAR 规范中注册，其键为正数。
否则，键应当为负数。
扩展值可以是任何合法的 JSON，包括嵌套对象。

默认策略包含一个名为 `ear.trustee.identifiers` 的扩展，用于存储标识工作负载的字段。
在机密容器中，这些标识符通常基于 Kata Agent Policy 或 InitData 中的其他配置字段。
其他运行时可能使用其他机制，例如事件日志（event log）。
`identifiers` 扩展包含两个子字段。

* `validated` 标识符：通过证明与硬件证据绑定，并且由 guest 运行时绑定到工作负载的标识符。
* `unvalidated` 标识符：其他任何描述该工作负载的标识符。

默认策略大致面向 CoCo 运行时，包含用于处理以下 `validated identifiers` 的逻辑。
当运行时提供/度量了所需信息时，它们会被包含在 `ear.trustee.identifiers` 扩展中。
* `pod_image`：该 Pod 使用的所有镜像的列表
* `pod_image_count`：一个 Pod 使用的镜像数量
* `container_image_uids`：Pod 中容器运行时所使用的 uid 列表

AS 策略会被提供 `input`。

`input` 是由校验器生成的 TCB 声明项。

示例参见[默认策略](../src/ear_token/ear_default_policy_cpu.rego)。

> [!NOTE]
> 我们现在在 AS 策略中提供了一个新的扩展。如果任何字段或取值应当从 RVPS 获取，
> 可以在 rego 中使用函数 `query_reference_value(key_name)`，
> 以键 id `key_name` 从 RVPS 取得参考值。如果在 RVPS 中找不到该键，该扩展将求值为 `null`。
>
> 一个片段示例如下
> ```
> executables := 3 if {
> 	input.tpm
> 	input.tpm.pcr11 == query_reference_value("pcr11")
> }
> ```

对于 EAR 令牌的签发，策略查找是通过在策略 id 后追加设备类别来完成的。
对于 `policy_ids: ["default"]` 的请求，AS 会评估：

- `default_cpu`：面向 CPU 类别的证据
- `default_gpu`：面向 GPU 类别的证据
- `default_switch`：面向 switch 类别的证据
- `default_ppcie`：面向 ppcie 类别的证据

AS 随发行版提供这些默认策略：`default_cpu`、`default_gpu`、`default_switch` 和 `default_ppcie`。

如果提供了多个 `policy_ids`，仅第一个会被用于生成 EAR 令牌。

## 如何使用策略

对于 [gRPC CoCo AS](../../protos/attestation.proto) 和 [Restful CoCo AS](./restful-as.md)，
我们都有一个名为 `policy_ids` 的参数，用于指定使用哪些策略来执行证据校验。

对于一个正在运行的 CoCoAS，我们可以设置任意的新策略。

下面给出一些快速指引。

### gRPC CoCo AS（gRPC 形态）

首先，启动一个 gRPC CoCo AS
```shell
cd attestation-service/tests/e2e
make grpc.pid
```

然后，向 CoCo AS 设置一条新策略。
我们准备了一个[示例 SetPolicy 请求](../tests/coco-as/policy/grpc-set-policy.json)，
带有[策略](../tests/coco-as/policy/new_policy.rego)和策略 id `example-policy`。
该策略应当使用 `Base64 URL SAFE NO PAD` 编码。

```shell
REQ=$(cat ../../tests/coco-as/policy/grpc-set-policy.json)
grpcurl \
  -plaintext \
  -import-path ../../protos \
  -proto ../../protos/attestation.proto \
  -d @ 127.0.0.1:50004 attestation.AttestationService/SetAttestationPolicy <<EOF
$REQ
EOF
```

然后，我们可以用该策略来校验证据。我们使用[该请求](../tests/coco-as/policy/check.json)来完成这一步。

```shell
REQ=$(cat ../../tests/coco-as/policy/check.json)
grpcurl \
  -plaintext \
  -import-path ../../protos \
  -proto ../../protos/attestation.proto \
  -d @ 127.0.0.1:50004 attestation.AttestationService/AttestationEvaluate <<EOF
$REQ
EOF
```

### Restful CoCo AS（RESTful 形态）

首先，启动一个 Restful CoCo AS
```shell
cd attestation-service/tests/e2e
make restful.pid
```

然后，向 CoCo AS 设置一条新策略。
我们准备了一个[示例 SetPolicy 请求](../tests/coco-as/policy/restful-set-policy.json)，
带有[策略](../tests/coco-as/policy/new_policy.rego)和策略 id `example-policy`。
该策略应当使用 `Base64 URL SAFE NO PAD` 编码。

```shell
curl -k -X POST http://127.0.0.1:8080/policy \
     -i \
     -H 'Content-Type: application/json' \
     -d @../../tests/coco-as/policy/restful-set-policy.json
```

然后，我们可以用该策略来校验证据。我们使用[该请求](../tests/coco-as/policy/check.json)来完成这一步。

```shell
curl -k -X POST http://127.0.0.1:8080/attestation \
     -i \
     -H 'Content-Type: application/json' \
     -d @../../tests/coco-as/policy/check.json
```

## 如何编写策略（实验性）

:warning: **警告：** 策略特性仍处于开发之中，在得到稳定版本之前可能会被灵活变更。

我们将通过提供一些示例来介绍策略的格式，以展示其使用场景。

1. [默认策略](../src/ear_token/ear_default_policy_cpu.rego)。该策略基于参考值为多个信任声明项赋值。
2. 一个 [SGX 策略](../tests/coco-as/policy/example-1.rego)。客户端希望确保 `mr_signer` 与 `mrenclave` 都是预期值。
3. 一个 [TDX 策略](../tests/coco-as/policy/example-2.rego)。客户端希望确保 TDX 模块（由 `tdx.quote.body.mr_seam` 反映）、guest 固件（由 `tdx.quote.body.mr_td` 反映）、tcb_status（由 `input.tdx.tcb_status` 反映）、collateral 过期状态（由 `input.tdx.collateral_expiration_status` 反映）都符合预期。
4. 一个 [IBM SE 策略](../tests/coco-as/policy/example-3.rego)。客户端希望确保 `se.version`、`se.tag`、`se.user_data`、`se.image_phkh` 和 `se.attestation_phkh` 都是预期值。
