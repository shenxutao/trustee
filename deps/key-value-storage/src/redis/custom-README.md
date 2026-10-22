# Redis 协议存储

> 说明：本文件是 `deps/key-value-storage/src/redis/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../custom-文档清单.md)。

一种使用 **Redis 协议**（RESP）的键值存储实现。它 **并不专门绑定 Redis** —— 任何实现了 Redis 线协议（wire protocol）的后端都可以使用，例如：

- [Redis](https://redis.io/)
- [Valkey](https://valkey.io/)
- [KeyDB](https://docs.keydb.dev/)
- 其他兼容 Redis 协议的服务

除非另有说明，本文档中提到的“Redis 协议服务器”泛指上述任意兼容后端。

## 概述

`RedisClient` 是一种通过 Redis 协议连接的键值存储后端，实现了 `KeyValueStorage` trait。
它将每个键值对以字符串键的形式存储在远端服务器上，值以原始字节形式存储。

## 特性

- **持久化存储**：服务器可依据自身配置（AOF/RDB 或等效机制）对数据进行持久化
- **命名空间隔离**：通过键前缀 `{namespace}:{key}` 隔离不同命名空间
- **并发访问**：通过 Redis 协议服务器支持并发读写

## 配置

### 默认配置

- **默认 URL**：`redis://127.0.0.1:6379`

若未指定 `url`，则使用上述默认 URL。URL 方案遵循惯用的 Redis 客户端约定（`redis://`，启用 TLS 时为 `rediss://`）。

### 环境变量

如果设置了 `REDIS_URL` 环境变量，它将优先于配置文件中的 `url`。这样可以由密钥存储（例如 Kubernetes Secret）注入内嵌凭据的连接 URL，而不必写入配置文件。

## 实现细节

### 存储格式

每个键值对的存储形式为：

- **键**：`{namespace}:{key}`
- **值**：原始字节（`Vec<u8>`）

例如，命名空间为 `kbs` 时：

- `("policy/default", b"allow")` 存储为键 `kbs:policy/default`

### 列举操作

`list()` 使用模式 `{namespace}:*` 查询键，然后在返回前剥离命名空间前缀。

## 测试

端到端覆盖由一个被标记为忽略的异步测试提供（`redis::tests::test_redis_client`）。请先在本地启动一个 Redis 协议服务器；默认配置 URL `redis://127.0.0.1:6379` 对应发布在宿主机 `6379` 端口上的容器。

### 使用 Docker 启动服务器（以 Redis 为例）

```bash
docker run -d --name redis-kvs-test -p 6379:6379 redis:8-alpine
```

位于同一 URL 上的其他 Redis 协议镜像或部署，其使用方式完全相同。

使用完毕后停止并删除容器：

```bash
docker stop redis-kvs-test && docker rm redis-kvs-test
```

### 运行被忽略的测试

需要启用 `redis` feature（该 feature 为可选 feature，不属于 crate 的默认 features）：

```bash
cargo test -p key-value-storage --no-default-features --features redis --lib -- redis::tests::test_redis_client --exact --ignored --show-output
```

## 注意事项

1. **键的限制**：键只能包含 `a-z`、`A-Z`、`0-9`、`-`、`_`、`.`、`/`
2. **运维依赖**：需要存在可访问的、兼容 Redis 协议的服务器
3. **大键空间**：`list()` 使用基于模式的键查找，在键空间非常大时开销可能较高

## 适用场景

- 多个服务实例之间共享存储
- 低延迟的分布式键值访问
- 已在运行 Redis 协议后端（Redis、Valkey、云厂商托管服务等）的生产环境
