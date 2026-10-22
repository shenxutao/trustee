# 键值存储的 PostgreSQL 后端

> 说明：本文件是 `deps/key-value-storage/src/postgres/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../custom-文档清单.md)。

本模块为键值存储系统提供基于 PostgreSQL 的 `KeyValueStorage` 实现。键值对存储在单张表中，值内容以二进制数据（BYTEA）形式保存，以确保任意二进制内容都能安全存储。

## 特性

- 将键值对存储在可配置的表中（`key` 为主键，`value` 为二进制类型）
- 通过 `sqlx` 实现安全的参数绑定

## 快速开始（使用 Docker）

使用提供的辅助脚本启动本地 PostgreSQL，便于开发调试：

```bash
bash set-up.sh
```

该脚本的作用：
- 启动一个名为 `postgres` 的容器，映射到宿主机端口 `6432`
- 为方便本地非生产环境使用，设置 `POSTGRES_HOST_AUTH_METHOD=trust`
- 在容器内执行 `set-up.sql`，创建键值表

清理环境：
```bash
docker stop policy-postgres && docker rm policy-postgres
```

## 配置

`PostgresClient::new` 接收一个 `Config`，其字段及默认值如下：

- `db`（默认值：`postgres`）
- `username`（默认值：`postgres`）
- `password`（可选）
- `port`（默认值：`5432`）
- `host`（默认值：`localhost`）
- `table`（默认值：`key_value`）

内部构造的连接串格式为：
`postgresql://username[:password]@host:port/db`

如果设置了 `POSTGRES_URL` 环境变量且其值为 postgres 连接 URI，则会优先使用该变量，而忽略上述配置项。

## 测试

代码中有一个被标记为忽略的异步测试，用于演示端到端用法。本地运行方式如下：

```bash
# 确保数据库已运行（例如通过 set-up.sh 启动）
cargo test --package key-value-storage --lib -- postgres::tests::test_postgres_client --exact --show-output --ignored
```

或者，在确保存在与你的 `Config` 相匹配且可访问的 PostgreSQL 命名空间后，运行常规测试套件。
