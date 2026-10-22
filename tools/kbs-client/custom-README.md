# KBS 客户端工具（KBS Client Tool）

> 说明：本文件是 `tools/kbs-client/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

这是一个简单的 KBS 客户端，用于便利地测试 KBS
以及其他基本的证明流程。

你可以在 TEE 内部运行该工具，以使用真实的证明证据发起请求。
你也可以提供已有的证据，或者使用 sample attester 作为回退方案。

该客户端工具还可用于向 KBS/AS 预置资源（resources）与策略（policies）。

如需更复杂的证明客户端，请参阅 [guest components](https://github.com/confidential-containers/guest-components)。

获取帮助：

```shell
kbs-client -h
```

我们在 [Github ORAS](https://github.com/confidential-containers/trustee/pkgs/container/staged-images%2Fkbs-client) 上提供了一个社区版 kbs-client。

## 构建与安装客户端

以支持默认特性的方式构建客户端二进制：

```shell
make -C ../../kbs cli
```

默认情况下，客户端构建时支持所有 attester。如果你只想以支持 sample attester 的方式构建（这需要
更少的依赖，因此通常在 CI 中很方便），可以传入
`sample_only` 特性：

```shell
make -C ../../kbs cli CLI_FEATURES=sample_only
```

构建产物位于 `../../target/release/kbs-client`。你可以按如下方式将其
安装到系统中：
```shell
sudo make -C ../../kbs install-cli
```

## 示例（Examples）

从 KBS 获取一个资源（在完成证明之后）：

```shell
./kbs-client --url http://127.0.0.1:8080 get-resource --path my_repo/resource_type/123abc
```

向 KBS 添加一个资源：

```shell
./kbs-client --url http://127.0.0.1:8080 config --admin-token-file ../../kbs/config/docker-compose/admin-token set-resource --path my_repo/resource_type/123abc --resource-file test_resource
```

设置一个资源策略：
```shell
./kbs-client --url http://127.0.0.1:8080 config --admin-token-file ../../kbs/config/docker-compose/admin-token set-resource-policy --policy-file allow_all.rego
```

