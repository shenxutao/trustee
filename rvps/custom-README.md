# 参考值提供服务

> 说明：本文件是 `rvps/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../custom-文档清单.md)。

参考值提供服务（Reference Value Provider Service，简称 RVPS）是一个组件，
用于接收软件供应链的来源（provenance）/元数据、对其进行校验，并提取出参考值。
所有参考值都会存储在 RVPS 内部。当证明服务查询特定软件的声明项时，
RVPS 会以相关的参考值作为响应。

## 架构

RVPS 包含以下组件：

- 提取器（Extractors）：提取器包含若干子模块，用于处理不同类型的来源。每个子模块会消费输入的 Message，然后生成输出的参考值。

- 存储（Storage）：RVPS 使用 `key-value-storage` crate 提供一个统一的键值存储接口。所有经过校验的参考值都会被存放在某个存储后端中。当证明服务请求时，会提供相关的参考值。存储后端可以配置为使用不同的实现，例如内存、本地文件系统、本地 JSON 文件或 PostgreSQL。

## 消息流

下图说明了 RVPS 的消息流：

![](./diagrams/rvps.svg)

### Message（消息）

有一种协议用于分发二进制产物的来源信息。它会被 RVPS 接收并处理，
如果工作正常，RVPS 随后会生成一份参考值。

```
{
    "version": <VERSION-NUMBER-STRING>,
    "type": <TYPE-OF-THE-PROVENANCE-STRING>,
    "provenance": #provenance,
}
```

`"version"` 字段是本消息的版本，使可扩展性成为可能。

`"type"` 字段指明该消息所携带来源的具体类型。

`"provenance"` 字段是传递给 RVPS 的主要内容。该字段包含需要由 RVPS 解析的载荷。
来源的含义取决于其类型以及处理它的具体提取器。

### Trust Digests（可信摘要）

它是证明服务真正请求并使用、用来与从 HW TEE 收集到的证据进行比对的值。它们通常是摘要。为避免歧义，它们被命名为 `trust digests` 而不是 `reference values`。

## 运行 RVPS

### 前置条件

安装 protocol buffer 编译器软件包 `protobuf-compiler`。

### 直接构建

通过这种方式，RVPS 可以作为一个独立服务运行。[gRPC protos](../protos/reference.proto) 已定义。

我们可以使用以下命令运行

```bash
git clone https://github.com/confidential-containers/trustee
cd trustee/rvps
make build && sudo make install
```

运行 RVPS
```shell
rvps
```

默认情况下，RVPS 监听 `localhost:50003` 等待请求。

### 容器镜像

我们可以构建一个 RVPS docker 镜像

```bash
cd .. && docker build -t rvps -f rvps/docker/Dockerfile .
```

运行该容器
```bash
docker run -d -p 50003:50003 rvps --address 0.0.0.0:50003
```

或者我们也可以把 RVPS 构建为 podman 镜像

```bash
cd .. && podman build -t rvps -f rvps/docker/Dockerfile .
```

运行
```bash
podman run -d -p 50003:50003 --net host rvps
```

### 配置文件

RVPS 可以通过 `-c` 参数以指定的配置文件启动。一份配置文件形如：

```json
{
    "storage": {
        "storage_type": "LocalFs",
        "backends": {
            "local_fs": {
                "dir_path": "/opt/confidential-containers/storage/local_fs"
            }
        }
    },
    "extractors": {
        "swid_extractor": {}
    }
}
```

#### 存储后端配置

<a id="storage-backend-configuration"></a>

`storage` 对象使用统一的存储后端配置格式。

关于统一存储后端配置格式的详细信息，包括**命名空间（namespace）**是什么以及它如何工作，
参见 [Key-Value Storage README](../deps/key-value-storage/README.md#unified-storage-backend-configuration)。

#### RVPS 中的存储命名空间

当 RVPS 作为独立服务运行时，它使用单个存储命名空间：

| 命名空间名称 | 组件 | 说明 |
|----------------|-----------|-------------|
| `reference_value` | RVPS | 存储用于软件供应链校验的参考值 |

详细的配置选项与示例参见 [Key-Value Storage README](../deps/key-value-storage/README.md#unified-storage-backend-configuration)。

#### 提取器配置

<a id="extractors-configuration"></a>

RVPS 使用提取器来处理不同类型的来源并提取参考值。`extractors` 配置是可选的，
允许你配置特定提取器的行为。

| 属性 | 类型 | 说明 | 是否必需 | 默认值 |
|----------|------|-------------|----------|---------|
| `swid_extractor` | Object | SWID/RIM 提取器的配置 | 否 | None（使用默认值） |

**可用提取器：**

RVPS 支持以下提取器（默认启用）：

1. **Sample 提取器**（`sample`）：一个用于测试与演示的简单提取器。它直接从输入中提取参考值，而不校验签名。详情参见 [Sample Extractor README](src/extractors/sample/README.md)。

2. **SWID/RIM 提取器**（`swid`）：从带有 RIM（Reference Integrity Manifest）绑定的 SWID tag 中提取参考值。目前已在 NVIDIA GPU vbios 的 RIM manifest 上完成测试。详情参见 [SWID RIM Extractor README](src/extractors/swid/README.md)。
   - 目前 `swid_extractor` 没有配置项（空对象 `{}`）
   - 该提取器不校验 manifest 签名；在提供给 RVPS 之前应当先验证其真实性
   - 默认过期时间：12 个月
   - 默认哈希算法：`sha384`

**配置示例：**

```json
{
    "storage": {
        "storage_type": "LocalFs",
        "backends": {
            "local_fs": {
                "dir_path": "/opt/confidential-containers/storage/local_fs"
            }
        }
    },
    "extractors": {
        "swid_extractor": {}
    }
}
```

**注意：** 如果没有指定 `extractors`，RVPS 会为所有提取器使用默认配置。
sample 提取器始终启用，而 SWID 提取器在未显式配置时将使用默认设置。

#### 配置示例

使用 LocalFs 存储：

```json
{
    "storage": {
        "storage_type": "LocalFs",
        "backends": {
            "local_fs": {
                "dir_path": "/opt/confidential-containers/storage/local_fs"
            }
        }
    }
}
```

使用 LocalJson 存储：

```json
{
    "storage": {
        "storage_type": "LocalJson",
        "backends": {
            "local_json": {
                "file_dir_path": "/opt/confidential-containers/storage/local_json"
            }
        }
    }
}
```

使用 PostgreSQL 存储：

```json
{
    "storage": {
        "storage_type": "Postgres",
        "backends": {
            "postgres": {
                "host": "localhost",
                "port": 5432,
                "db": "rvps",
                "username": "postgres",
                "password": "password"
            }
        }
    }
}
```

使用内存存储（默认）：

```json
{
    "storage": {
        "storage_type": "Memory"
    }
}
```

## 把 RVPS 集成进证明服务

### 原生模式（不推荐）

在这种模式下，RVPS 会作为证明服务二进制内部的一个 crate 工作。

![](./diagrams/rvps-native.svg)

> [!NOTE]
> **存储配置：** 默认情况下，原生模式下的 RVPS 使用统一的 `storage_backend` 配置（参见[存储后端配置](#storage-backend-configuration)），并使用 `reference_value` 命名空间。不过，你也可以选择性地在 `rvps_config` 中提供一个 `storage_type` 字段，仅覆盖 RVPS 所使用的存储类型。
> 这允许你为不同的组件使用不同的存储后端（例如 KBS 资源使用 LocalFs，而 RVPS 参考值使用 LocalJson）。

**共享存储的示例（默认行为）：**

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
    }
}
```

**使用 RVPS 专用存储类型覆盖的示例：**

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
    }
}
```

### gRPC 模式

在这种模式下，证明服务会连接到一个远端 RVPS。
这要求证明服务在构建时启用了 `rvps-grpc` 特性。

```bash
cd ../attestation-service && cargo run --bin as-grpc -- --config-file config.json
```

![](./diagrams/rvps-grpc.svg)

## 客户端工具

`rvps-tool` 是一个用于与 RVPS 交互的命令行客户端。它可以：
- 把参考值注册进 RVPS
- 从 RVPS 查询参考值

### 与 RVPS 交互的快速指引

在 docker 中运行 RVPS，或者通过以下命令运行
```bash
RVPS_ADDR=127.0.0.1:50003
rvps --address $RVPS_ADDR
```

以 [sample 格式](./src/extractors/sample/README.md)创建一条测试消息
```bash
cat << EOF > sample
{
    "test-binary-1": [
        "reference-value-1",
        "reference-value-2"
    ],
    "test-binary-2": [
        "reference-value-3",
        "reference-value-4"
    ]
}
EOF
provenance=$(cat sample | base64 --wrap=0)
cat << EOF > message
{
    "version" : "0.1.0",
    "type": "sample",
    "payload": "$provenance"
}
EOF
```

把该来源注册进 RVPS
```bash
rvps-tool register --path ./message --addr http://$RVPS_ADDR
```

它会显示类似下面的内容
```
[2023-03-09T04:44:11Z INFO  rvps_client] Register provenance succeeded.
```

随后我们来查询参考值
```bash
rvps-tool query --addr http://$RVPS_ADDR
```

输出应当类似如下：
```
[2025-01-24T06:04:41Z INFO  rvps_tool] Get reference values succeeded:
     {"test-binary-1":["reference-value-1","reference-value-2"],
      "test-binary-2":["reference-value-3","reference-value-4"]}
```
