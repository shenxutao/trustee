# CoCo AS 配置文件

> 说明：本文件是 `attestation-service/docs/config.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

机密容器 KBS 的各项属性可以通过一份 JSON 格式的配置文件进行配置。

## 可配置属性

以下各节列出了可以通过配置文件设置的 CoCo AS 属性。

### 全局属性

以下属性可以全局设置，即不属于任何配置小节：

<a id="global-properties"></a>

| 属性                       | 类型                        | 说明                                                | 是否必需 | 默认值 |
|----------------------------|-----------------------------|-----------------------------------------------------|----------|---------|
| `rvps_config`              | [RVPSConfiguration][3]      | RVPS 配置                                           | 否       | `BuiltIn` |
| `attestation_token_broker` | [AttestationTokenBroker][1] | 证明结果令牌配置。                                   | 否       | 见下文 |
| `verifier_config`          | Object                      | 可选的、与具体校验器相关的配置（例如 TPM）           | 否 | 无 |
| `storage_backend`          | [StorageBackendConfig][4]   | 面向所有存储需求的统一存储后端配置                   | 否 | 见下文 |

[1]: #attestationtokenbroker
[2]: #tokensignerconfig
[3]: #rvps-configuration
[4]: #unified-storage-backend-configuration

#### AttestationTokenBroker（证明令牌代理）

<a id="attestationtokenbroker"></a>

| 属性       | 类型                    | 说明                                          | 是否必需 | 默认值 |
|----------------|-------------------------|------------------------------------------------------|----------|---------|
| `duration_min` | Integer                 | 证明结果令牌的有效时长（单位：分钟）。 | 否       | `5`     |
| `issuer_name`  | String                  | 证明结果令牌的签发者名称。         | 否       |`CoCo-Attestation-Service`|
| `developer_name`  | String               | 用作 EAR 中校验者 ID 组成部分的开发者名称 | 否       |`https://confidentialcontainers.org`|
| `build_name`  | String                  | 用作 EAR 中校验者 ID 组成部分的构建名称         | 否       | 由 Cargo 包与 AS 版本自动生成|
| `profile_name`  | String                  | 描述该 EAR 令牌的 Profile         | 否       |tag:github.com,2024:confidential-containers/Trustee`|
| `signer`       | [TokenSignerConfig][2]  | 证明结果令牌的签名材料。    | 否       | 无       |
| `verbose_token` | Bool                   | 在证明令牌中包含详细信息    | 否       | True       |

#### TokenSignerConfig（令牌签名配置）

<a id="tokensignerconfig"></a>

该小节是**可选的**。省略时会生成并使用一个新的 EC 密钥对。

证书链（通过 `cert_path` 或 `cert_url` 配置）会以 `jwk.x5c` 的形式嵌入签发的 JWT 头部。
KBS 通过依据 `[attestation_token].trusted_certs_paths` 校验证书链来校验这些令牌。参见
[KBS 证明令牌校验](../../kbs/docs/attestation_token_verification.md)。

| 属性       | 类型    | 说明                                             | 是否必需 | 默认值 |
|----------------|---------|---------------------------------------------------------|----------|---------|
| `key_path`     | String  | EC 密钥对文件（PEM 格式）路径。                     | 是      | -       |
| `cert_url`     | String  | EC 公钥证书链（PEM 格式）URL。       | 否       | -       |
| `cert_path`    | String  | EC 公钥证书链（PEM 格式）文件路径。 | 否       | -       |

#### RVPS 配置

<a id="rvps-configuration"></a>

| 属性       | 类型                    | 说明                                          | 是否必需 | 默认值 |
|----------------|-------------------------|------------------------------------------------------|----------|---------|
| `type`         | String                  | 可以取 `BuiltIn`（内置 RVPS）或 `GrpcRemote`（连接到远端 gRPC RVPS） | 否       | `BuiltIn` |

##### 内置 RVPS

如果 `type` 被设为 `BuiltIn`，可以设置以下额外属性：

| 属性 | 类型 | 说明 | 是否必需 | 默认值 |
|----------|------|-------------|----------|---------|
| `extractors` | Object | 来源提取器（provenance extractors）的可选配置 | 否 | 无 |
| `storage_type` | String | 可选的 RVPS 专用存储类型覆盖。若提供，仅对 RVPS 覆盖 `storage_backend.storage_type`。与具体后端相关的参数仍复用 `storage_backend.backends` 中的配置。 | 否 | 无 |

> [!NOTE]
> **存储配置：** 默认情况下，内置 RVPS 使用统一的 `storage_backend` 配置（参见[存储后端配置](#unified-storage-backend-configuration)），并使用 `reference_value` 命名空间。不过，你也可以选择性地提供 `storage_type` 字段，仅覆盖 RVPS 所使用的存储类型。
> 这允许你为不同的组件使用不同的存储后端（例如 KBS 与 AS 策略使用 LocalFs，而 RVPS 参考值使用 LocalJson）。

关于提取器配置的详细信息，包括可用的提取器及其选项，参见 [RVPS README](../../rvps/README.md#extractors-configuration)。

**示例：**
```json
{
    "rvps_config": {
        "type": "BuiltIn",
        "extractors": {
            "swid_extractor": {}
        }
    }
}
```

##### 远端 RVPS

如果 `type` 被设为 `GrpcRemote`，可以设置以下额外属性

| 属性       | 类型                    | 说明                             | 是否必需 | 默认值          |
|----------------|-------------------------|-----------------------------------------|----------|------------------|
| `address`      | String                  | RVPS 服务的远端地址       | 否       | `127.0.0.1:50003`|

#### 校验器配置

<a id="verifier-configuration"></a>

`verifier_config` 小节允许进行与具体 TEE 相关的校验器配置。每个校验器都可以根据已启用的特性拥有自己的配置。

##### NVIDIA GPU 校验器

当启用 `nvidia-verifier` 特性时可用。详情参见 [`nvidia-verifier` 文档](../../deps/verifier/src/nvidia/README.md)。

| 属性          | 类型                             | 说明                                     | 是否必需 | 默认值 |
|-------------------|----------------------------------|-------------------------------------------------|----------|---------|
| `nvidia_verifier` | [NvidiaVerifierConfig][nvidia-1] | NVIDIA GPU 校验器配置               | 否       | -       |

[nvidia-1]: #nvidiaverifierconfig

###### NvidiaVerifierConfig（NVIDIA 校验器配置）

<a id="nvidiaverifierconfig"></a>

| 属性 | 类型   | 说明                                                     | 是否必需 | 默认值 |
|----------|--------|-----------------------------------------------------------------|----------|---------|
| `type`   | String | 校验类型：`"Local"` 或 `"Remote"`                      | 否       | `"Local"` |
| `verifier_url` | String | 远端校验器 URL（仅用于 `"Remote"` 类型）            | 否       | `https://nras.attestation.nvidia.com/v4/attest` |

##### TPM 校验器

当启用 `tpm-verifier` 特性时可用。

| 属性      | 类型                           | 说明                      | 是否必需 | 默认值 |
|---------------|--------------------------------|----------------------------------|----------|---------|
| `tpm_verifier` | [TpmVerifierConfig][tpm-1]     | TPM 校验器配置       | 否       | -       |

[tpm-1]: #tpmverifierconfig

###### TpmVerifierConfig（TPM 校验器配置）

<a id="tpmverifierconfig"></a>

| 属性              | 类型   | 说明                                             | 是否必需 | 默认值                          |
|-----------------------|--------|---------------------------------------------------------|----------|----------------------------------|
| `trusted_ak_keys_dir` | String | 存放受信任的证明密钥（AK）文件的目录 | 否       | `/etc/tpm/trusted_ak_keys`       |
| `max_trusted_ak_keys` | Integer | 要加载的受信任 AK 密钥的最大数量              | 否       | `100`                            |

##### SNP 校验器

当启用 `snp-verifier` 特性时可用。详情参见 [AMD SNP 证书缓存指南](./amd-offline-certificate-cache.md)。

| 属性      | 类型                           | 说明                      | 是否必需 | 默认值 |
|---------------|--------------------------------|----------------------------------|----------|---------|
| `snp_verifier` | [SnpVerifierConfig][snp-1]     | AMD SEV-SNP 校验器配置 | 否     | -       |

[snp-1]: #snpverifierconfig

###### SnpVerifierConfig（SNP 校验器配置）

<a id="snpverifierconfig"></a>

| 属性       | 类型                 | 说明                                   | 是否必需 | 默认值                         |
|----------------|----------------------|-----------------------------------------------|----------|---------------------------------|
| `vcek_sources` | Array of [VCEKSource][snp-2] | 用于获取 VCEK 证书的来源 | 否       | `[{"type": "KDS"}]`             |

[snp-2]: #vceksource

###### VCEKSource（VCEK 来源）

<a id="vceksource"></a>

一个 VCEK 来源可以是 `KDS`（AMD 密钥分发服务）或 `OfflineStore`（本地文件系统）。

对于 `KDS` 类型：

| 属性   | 类型   | 说明                    | 是否必需 | 默认值                               |
|------------|--------|--------------------------------|----------|---------------------------------------|
| `type`     | String | 必须为 `"KDS"`                | 是      | -                                     |
| `base_url` | String | KDS 服务的基地址       | 否       | `https://kdsintf.amd.com`             |

对于 `OfflineStore` 类型：

| 属性 | 类型   | 说明                                | 是否必需 | 默认值                                                              |
|----------|--------|--------------------------------------------|----------|----------------------------------------------------------------------|
| `type`   | String | 必须为 `"OfflineStore"`                   | 是      | -                                                                    |
| `path`   | String | 离线证书存储的路径          | 否       | `/opt/confidential-containers/attestation-service/kds-store`         |

##### Intel DCAP 校验器

当启用 `tdx-verifier`、`sgx-verifier` 或 `az-tdx-vtpm-verifier` 特性时可用。

| 属性        | 类型                      | 说明                                  | 是否必需 | 默认值 |
|-----------------|---------------------------|----------------------------------------------|----------|---------|
| `dcap_verifier` | [QcnlConfig][dcap-1]      | Intel DCAP 配置                     | 否       | -       |

[dcap-1]: #qcnlconfig

###### QcnlConfig（QCNL 配置）

<a id="qcnlconfig"></a>

| 属性             | 类型    | 说明                                          | 是否必需 | 默认值                                                      |
|----------------------|---------|------------------------------------------------------|----------|--------------------------------------------------------------|
| `collateral_service` | String  | PCS/PCCS collateral 服务的 URL，或用于离线校验的 `file://` 路径（见下方说明） | 是      | -                                                            |
| `tcb_update_type`    | String  | TCB 更新类型：`"early"` 或 `"standard"`           | 否       | `"early"`                                                    |

> **离线校验：** `file://` URL 必须指向由
> [Intel PCS Client Tool](https://github.com/intel/confidential-computing.tee.dcap/tree/main/tools/PcsClientTool)
> 生成的 JSON 文件。


## 配置示例

以内置 RVPS 运行（使用统一存储后端）：

```json
{
    "storage_backend": {
        "storage_type": "LocalFs",
        "backends": {
            "local_fs": {
                "dir_path": "/var/lib/attestation-service/storage"
            }
        }
    },
    "rvps_config": {
        "type": "BuiltIn"
    },
    "attestation_token_broker": {
        "duration_min": 5
    }
}
```

以内置 RVPS 运行，并配置提取器：

```json
{
    "storage_backend": {
        "storage_type": "LocalFs",
        "backends": {
            "local_fs": {
                "dir_path": "/var/lib/attestation-service/storage"
            }
        }
    },
    "rvps_config": {
        "type": "BuiltIn",
        "extractors": {
            "swid_extractor": {}
        }
    },
    "attestation_token_broker": {
        "duration_min": 5
    }
}
```

以内置 RVPS 运行，并使用 RVPS 专用的存储类型覆盖：

```json
{
    "storage_backend": {
        "storage_type": "LocalFs",
        "backends": {
            "local_fs": {
                "dir_path": "/var/lib/attestation-service/storage"
            },
            "local_json": {
                "file_dir_path": "/var/lib/rvps/references"
            }
        }
    },
    "rvps_config": {
        "type": "BuiltIn",
        "storage_type": "LocalJson"
    },
    "attestation_token_broker": {
        "duration_min": 5
    }
}
```

以远端 RVPS 运行：

```json
{
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
        "duration_min": 5
    }
}
```

令牌签名相关配置

```json
{
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
        "duration_min": 5,
        "issuer_name": "some-body",
        "signer": {
            "key_path": "/etc/coco-as/signer.key",
            "cert_url": "https://example.io/coco-as-certchain",
            "cert_path": "/etc/coco-as/signer.pub"
        }
    }
}
```

带 TPM 校验器的配置：

```json
{
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
        "duration_min": 5
    },
    "verifier_config": {
        "tpm_verifier": {
            "trusted_ak_keys_dir": "/etc/tpm/trusted_ak_keys",
            "max_trusted_ak_keys": 50
        }
    }
}
```

使用 KDS 的 AMD SEV-SNP 校验器配置：

```json
{
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
        "duration_min": 5
    },
    "verifier_config": {
        "snp_verifier": {
            "vcek_sources": [
                {
                    "type": "KDS",
                    "base_url": "https://kdsintf.amd.com"
                }
            ]
        }
    }
}
```

以统一存储后端运行：

```json
{
    "storage_backend": {
        "storage_type": "LocalFs",
        "backends": {
            "local_fs": {
                "dir_path": "/var/lib/attestation-service/storage"
            }
        }
    },
    "rvps_config": {
        "type": "BuiltIn"
    },
    "attestation_token_broker": {
        "duration_min": 5
    }
}
```


以统一存储后端使用 PostgreSQL 存储运行：

```json
{
    "storage_backend": {
        "storage_type": "Postgres",
        "backends": {
            "postgres": {
                "host": "localhost",
                "port": 5432,
                "db": "coco_as",
                "username": "postgres",
                "password": "password"
            }
        }
    },
    "rvps_config": {
        "type": "BuiltIn"
    },
    "attestation_token_broker": {
        "duration_min": 5
    }
}
```

使用离线存储的 AMD SEV-SNP 校验器配置：

```json
{
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
        "duration_min": 5
    },
    "verifier_config": {
        "snp_verifier": {
            "vcek_sources": [
                {
                    "type": "OfflineStore",
                    "path": "/var/lib/attestation-service/kds-store"
                }
            ]
        }
    }
}
```

带 NVIDIA GPU 校验器的配置（本地校验）：

```json
{
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
        "duration_min": 5
    },
    "verifier_config": {
        "nvidia_verifier": {
            "type": "Local"
        }
    }
}
```

带 NVIDIA GPU 校验器的配置（通过 NRAS 远端校验）：

```json
{
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
        "duration_min": 5
    },
    "verifier_config": {
        "nvidia_verifier": {
            "type": "Remote",
            "verifier_url": "https://nras.attestation.nvidia.com/v4/attest"
        }
    }
}
```

带 Intel DCAP 校验器的配置（TDX/SGX）：

```json
{
    "rvps_config": {
        "type": "GrpcRemote",
        "address": "127.0.0.1:50003"
    },
    "attestation_token_broker": {
        "duration_min": 5
    },
    "verifier_config": {
        "dcap_verifier": {
            "collateral_service": "https://api.trustedservices.intel.com/sgx/certification/v4/",
            "tcb_update_type": "early"
        }
    }
}
```
### 存储后端配置

<a id="unified-storage-backend-configuration"></a>

CoCo AS 支持统一存储后端配置，允许你声明一份单独的存储配置，
供证明服务中的所有存储需求使用。

这样无需为每个组件分别配置存储，从而简化了部署。

关于统一存储后端配置格式的详细信息，包括**命名空间（namespace）**是什么以及它如何工作，
参见 [Key-Value Storage README](../../deps/key-value-storage/README.md#unified-storage-backend-configuration)。

#### CoCo AS 中的存储命名空间

当使用统一存储后端配置时，CoCo AS 会创建以下存储命名空间：

| 命名空间名称 | 组件 | 说明 |
|----------------|-----------|-------------|
| `attestation_service_policy` | 证明令牌代理（Attestation Token Broker） | 存储 EAR（Entity Attestation Report）策略 |
| `reference_value` | 内置 RVPS | 存储用于软件供应链校验的参考值 |

统一存储后端配置是可选的。如果未提供，CoCo AS 会使用默认存储后端（内存）。
在生产环境中，请在配置文件中配置持久化后端（例如 LocalFs 或 Postgres）。

详细的配置选项与示例参见 [Key-Value Storage README](../../deps/key-value-storage/README.md#unified-storage-backend-configuration)。
