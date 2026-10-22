# 使用 Trustee 认证服务验证 NVIDIA 设备

> 说明：本文件是 `deps/verifier/src/nvidia/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../custom-文档清单.md)。

本验证器提供两种方式来验证 NVIDIA 设备。

## 本地验证器
`local` 验证器会解析硬件证据（SPDM 消息）并提取度量值。
随后策略可以将这些度量值与参考值进行比对。

## 远程验证器
`remote` 验证器使用 NVIDIA NRAS 服务来校验证据。

要使用该方式，用户需先与 NVIDIA 签订许可协议。
协议说明见[此处](https://docs.nvidia.com/attestation/cloud-services/latest/license.html)，
其中包含面向研究与开发的条款。

启用 `remote` 验证器时，由 NRAS 负责根据参考值对证据进行判定。

`remote` 验证器不会将原始硬件度量值作为 TCB Claims 提供，而是导出与验证过程中每个步骤相关的 claims。

策略通过检查这些 claims 来确认认证（attestation）已成功完成。

可通过在认证服务配置文件中添加以下条目来启用远程验证器：

- JSON 格式（`as-config.json`）：

    ```json
    {
        "verifier_config" : {
            "nvidia_verifier": {
                "type": "Remote"
            }
        }
    }
    ```
  
- TOML 格式：

    ```toml
    [verifier_config.nvidia_verifier.verifier]
        type = "Remote"
    ```

此外，验证器配置也可以写在 KBS 配置文件 `kbs-config.toml` 中：

```toml
[attestation_service.verifier_config.nvidia_verifier]
    type = "Remote"
```

更多细节请参见 [NVIDIA GPU Verifier 配置小节](../../../../attestation-service/docs/config.md#nvidia-gpu-verifier)。
