# TPM 验证器

> 说明：本文件是 `deps/verifier/src/tpm/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../custom-文档清单.md)。

TPM 验证器用于校验来自基于 TPM（可信平台模块）系统的认证证据。
它验证由 AK（Attestation Key，认证密钥）签名的 TPM 引用（quote），并校验 PCR（平台配置寄存器）的值。

TPM 验证器要求预先配置受信任的 AK 公钥。


## 环境准备

### 1. 策略配置

为了校验收到的 AK 是可信的，你需要在策略中配置如下条件：

```rego
input.tpm.ak_public in query_reference_value(<reference_name_for_trusted_ak>)
```
在默认策略中，该参考值名称为 `trusted_ak`，但只要与策略中使用的名称一致，你可以任选名称。

### 2. 添加受信任的 AK 公钥

从你的 TPM 设备中导出 AK 公钥：

公钥文件应为 PEM 格式（base64 编码的 DER）：
```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtmS6twmGpeO2i8JpQ9ZV
...
-----END PUBLIC KEY-----
```
复制该密钥，并按如下方式设置参考值：

```bash
kbs-client config --auth-private-key <private_key_file> set-sample-reference-value <reference_name_for_trusted_ak> <trusted_ak>
```

## 证据格式

TPM 验证器期望的证据为如下 JSON 格式：

```json
{
  "ak_public": "base64-encoded DER public key",
  "tpm_quote": {
    "message": "base64-encoded attestation message",
    "pcrs": [
      "hex-encoded PCR0",
      "hex-encoded PCR1",
      ...
      "hex-encoded PCR23"
    ],
    "signature": "base64-encoded TPM signature"
  }
}
```
