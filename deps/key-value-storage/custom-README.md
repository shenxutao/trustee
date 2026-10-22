# 键值存储（Key-Value Storage）

> 说明：本文件是 `deps/key-value-storage/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

面向 Rust 应用的键值存储接口及多种后端实现。

## 概述

本 crate 提供统一的 `KeyValueStorage` trait 和多种后端实现，允许应用根据自身需求选择最合适的存储后端。它同时支持内存型与持久化存储，适用于多种部署场景。

## 特性

- **统一接口**：所有后端共同实现的 `KeyValueStorage` trait
- **多种后端**：支持 memory（内存）、local file system（本地文件系统）、local JSON file（本地 JSON 文件）、PostgreSQL 和 Redis 存储
- **线程安全**：所有实现均为线程安全，支持并发访问
- **异步支持**：基于 async/await 构建，实现高效的 I/O 操作
- **灵活配置**：通过配置结构体即可轻松完成配置

## 可用后端

### 内存（`Memory`）

基于 `HashMap` 的内存键值存储。数据不会被持久化，程序终止后即丢失。

- **使用场景**：测试、缓存、临时存储
- **特性**：访问速度快，无 I/O 开销

### 本地文件系统（`LocalFs`）

每个键值对以独立文件形式存储在本地文件系统中。

- **使用场景**：单机环境、需要文件级访问的场景
- **特性**：文件级粒度，自动创建目录

详见 [LocalFs README](src/local_fs/README.md)。

### 本地 JSON（`LocalJson`）

所有键值对存储在单个 JSON 文件中。

- **使用场景**：单机环境、简单的持久化存储
- **特性**：人类可读格式，便于查看

详见 [LocalJson README](src/local_json/README.md)。

### PostgreSQL（`Postgres`）

将键值对存储在 PostgreSQL 数据库表中。

- **使用场景**：生产环境、分布式系统、高可用需求
- **特性**：ACID 事务、并发访问、可扩展性

详见 [Postgres README](src/postgres/README.md)。

### Redis 协议存储（`Redis`）

将键值对存储在 Redis 协议服务器中，并通过命名空间前缀实现逻辑隔离。

- **使用场景**：分布式部署、低延迟共享存储
- **特性**：命名空间隔离、异步访问、运维模型简单

详见 [Redis README](src/redis/README.md)。

## 统一存储后端配置

`StorageBackendConfig` 提供了一种统一的方式来配置存储后端，使同一应用中的多个组件可以共享该配置。这对于 KBS（Key Broker Service，密钥代理服务）这类需要针对不同用途管理多个存储命名空间的应用尤为有用。

### 什么是命名空间？

**命名空间（namespace）** 是同一存储后端内部对数据进行逻辑隔离的机制。使用统一存储后端配置时，应用的各个组件可以共用同一种存储后端类型和配置，但各自的数据通过不同的命名空间分开存放。

例如：
- 使用 `LocalFs` 后端时：不同命名空间存放在不同子目录中
- 使用 `LocalJson` 后端时：不同命名空间使用不同的 JSON 文件名
- 使用 `Postgres` 后端时：不同命名空间使用不同的表名

这样你可以：
- 为所有组件使用同一份存储配置
- 按组件保持数据的逻辑隔离
- 简化部署与维护

### 配置结构

统一存储后端配置使用如下结构：

```toml
[storage_backend]
storage_type = "LocalFs"  # 或 "Memory"、"LocalJson"、"Postgres"、"Redis"

[storage_backend.backends.local_fs]
dir_path = "/opt/confidential-containers/storage/local_fs"

[storage_backend.backends.local_json]
file_dir_path = "/opt/confidential-containers/storage/local_json"

[storage_backend.backends.postgres]
host = "localhost"
port = 5432
db = "postgres"
username = "postgres"
password = "password"

[storage_backend.backends.redis]
url = "redis://127.0.0.1:6379"
```

或者使用 JSON 格式：

```json
{
    "storage_backend": {
        "storage_type": "LocalFs",
        "backends": {
            "local_fs": {
                "dir_path": "/opt/confidential-containers/storage/local_fs"
            }
        }
    }
}
```

### 配置属性

| 属性 | 类型 | 说明 | 是否必填 | 默认值 |
|----------|------|-------------|----------|---------|
| `storage_type` | String | 存储后端类型。可选值：`Memory`/`memory`、`LocalFs`/`local_fs`、`LocalJson`/`local_json`、`Postgres`/`postgres`、`Redis`/`redis` | 否 | `Memory` |
| `backends` | Object | 后端专属配置对象 | 否 | - |

`backends` 对象可包含以下子段：

| 属性 | 类型 | 说明 | 是否必填 | 默认值 |
|----------|------|-------------|----------|---------|
| `local_fs` | Object | LocalFs 后端配置（当 `storage_type = "LocalFs"` 时必填） | 视条件而定 | - |
| `local_json` | Object | LocalJson 后端配置（当 `storage_type = "LocalJson"` 时必填） | 视条件而定 | - |
| `postgres` | Object | PostgreSQL 后端配置（当 `storage_type = "Postgres"` 时必填） | 视条件而定 | - |
| `redis` | Object | Redis 后端配置（当 `storage_type = "Redis"` 时必填） | 视条件而定 | - |

### LocalFs 后端配置

当 `storage_type = "LocalFs"` 时，可在 `backends.local_fs` 下设置以下属性：

| 属性 | 类型 | 说明 | 是否必填 | 默认值 |
|----------|------|-------------|----------|---------|
| `dir_path` | String | 所有存储命名空间的基础目录路径。不同命名空间将存放在该路径下的子目录中。 | 否 | `/opt/confidential-containers/storage/local_fs` |

**示例：**
- 命名空间 `"kbs"` → 存放在 `<dir_path>/kbs/`
- 命名空间 `"resource"` → 存放在 `<dir_path>/resource/`

### LocalJson 后端配置

当 `storage_type = "LocalJson"` 时，可在 `backends.local_json` 下设置以下属性：

| 属性 | 类型 | 说明 | 是否必填 | 默认值 |
|----------|------|-------------|----------|---------|
| `file_dir_path` | String | 所有 JSON 文件的基础目录路径。不同命名空间将使用该目录下的不同文件名。 | 否 | `/opt/confidential-containers/storage/local_json` |

**示例：**
- 命名空间 `"kbs"` → 存放在 `<file_dir_path>/kbs.json`
- 命名空间 `"resource"` → 存放在 `<file_dir_path>/resource.json`

### PostgreSQL 后端配置

当 `storage_type = "Postgres"` 时，可在 `backends.postgres` 下设置以下属性：

| 属性 | 类型 | 说明 | 是否必填 | 默认值 |
|----------|------|-------------|----------|---------|
| `db` | String | PostgreSQL 数据库名 | 否 | `postgres` |
| `username` | String | PostgreSQL 数据库用户名 | 否 | `postgres` |
| `password` | String | PostgreSQL 数据库密码 | 否 | 无 |
| `port` | Integer | PostgreSQL 数据库端口 | 否 | `5432` |
| `host` | String | PostgreSQL 数据库主机 | 否 | `localhost` |

**示例：**
- 命名空间 `"kbs"` → 存放在表 `kbs` 中
- 命名空间 `"resource"` → 存放在表 `resource` 中

> **注意：** 如果设置了 `POSTGRES_URL` 环境变量且其值为 PostgreSQL 连接 URI，则会优先使用该变量，而忽略上述配置参数。

### 内存后端

当 `storage_type = "Memory"` 时，无需额外配置。每个命名空间各自拥有独立的内存存储，数据不会被持久化。

### Redis 后端配置

当 `storage_type = "Redis"` 时，可在 `backends.redis` 下设置以下属性：

| 属性 | 类型 | 说明 | 是否必填 | 默认值 |
|----------|------|-------------|----------|---------|
| `url` | String | Redis 连接 URL | 否 | `redis://127.0.0.1:6379` |

**示例：**
- 命名空间 `"kbs"` 下的键 `"policy/default"` → 存储为 Redis 键 `kbs:policy/default`

## 键格式

支持的键字符集包括：
- 小写字母：`a-z`
- 大写字母：`A-Z`
- 数字：`0-9`
- 特殊字符：`-`、`_`、`.`、`/`

**键的限制：**
- 键不能以 `.`（英文句点）开头
- 键不能以 `/`（斜杠）开头
