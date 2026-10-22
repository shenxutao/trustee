# 快速开始

> 说明：本文件是 `kbs/quickstart.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../custom-文档清单.md)。

本指南介绍如何在 Ubuntu 22.04 上本地构建并部署 KBS 各组件（不使用容器）。
指南同时涵盖配置与测试。
如果你想要更快捷的方式，可以考虑使用
[docker compose](./docs/cluster.md) 或 [KBS operator](https://github.com/confidential-containers/kbs-operator) 来部署 KBS 组件。

## 前置条件

安装 Rust 工具链：
```shell
curl https://sh.rustup.rs -sSf | sh
source "$HOME/.cargo/env"
```

安装依赖：
```shell
curl -L "https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key" | sudo gpg --dearmor --output /usr/share/keyrings/intel-sgx.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-sgx.gpg] https://download.01.org/intel-sgx/sgx_repo/ubuntu jammy main" \
	| sudo tee /etc/apt/sources.list.d/intel-sgx.list
sudo apt-get update
sudo apt-get install -y \
	build-essential \
	clang \
	libsgx-dcap-quote-verify-dev \
	libsgx-dcap-quote-verify \
	libtss2-dev \
	openssl \
	pkg-config \
	protobuf-compiler
```

生成用户认证密钥对和管理员令牌：
```shell
openssl genpkey -algorithm ed25519 > config/private.key
openssl pkey -in config/private.key -pubout -out config/public.pub
b64url() { openssl base64 -A | tr '+/' '-_' | tr -d '='; }
header='{"alg":"EdDSA","typ":"JWT"}'
iat=$(date +%s); exp=$((iat + 315360000))
payload="{\"issuer\":\"admin\",\"subject\":\"admin\",\"audiences\":[],\"iat\":${iat},\"exp\":${exp}}"
h64=$(printf '%s' "$header" | b64url)
p64=$(printf '%s' "$payload" | b64url)
sig=$(printf '%s' "${h64}.${p64}" | openssl pkeyutl -sign -inkey config/private.key -rawin | b64url)
printf '%s.%s.%s\n' "$h64" "$p64" "$sig" > config/admin-token
```

如果你希望以 HTTPS 方式连接 KBS，
应当使用受信任的 CA 为 KBS 生成服务端证书，
并通过启动参数将其传给 KBS 服务器。

## 后台检查模式

### 构建与部署

以后台检查模式构建 KBS：
```shell
make background-check-kbs
sudo make install-kbs
```

构建并安装 kbs-client：
```shell
make cli
sudo make install-cli
```

以 HTTP 服务器方式启动 KBS：
```shell
sudo kbs --config-file config/kbs-config.toml &
```

如果你希望以 HTTPS 服务器方式启动 KBS，可使用 `--private-key` 和 `--certificate`
指定服务端 HTTPS 私钥与证书的文件路径（两者均为 PEM 格式）。

### 上传资源数据

生成一个用于测试的资源数据文件：
```shell
cat > test/dummy_data << EOF
1234567890abcde
EOF
```

使用 `kbs-client` 将资源数据上传到 KBS 存储：

```shell
kbs-client --url http://127.0.0.1:8080 config --admin-token-file config/admin-token set-resource --resource-file test/dummy_data --path default/test/dummy
```

这里我们为该资源指定了一个自定义路径：`default/test/dummy`。

### 获取资源数据

执行以下命令从 KBS 获取资源数据：
```shell
kbs-client --url http://127.0.0.1:8080 get-resource --path default/test/dummy
```

如果你在 TEE 之外运行客户端，将会使用 sample attester。
默认情况下 KBS 会拒绝所有 sample 证据。
若要用 sample 证据测试 KBS，需要将资源策略
改为更宽松的策略。
可以通过如下命令完成：
```shell
kbs-client --url http://127.0.0.1:8080 config --admin-token-file config/admin-token set-resource-policy --policy-file sample_policies/allow_all.rego
```

## 护照模式

### 构建与部署

构建并启动用于令牌分发的 KBS：
```shell
make passport-issuer-kbs
make install-issuer-kbs
issuer-kbs --socket 127.0.0.1:50001 --insecure-http --auth-public-key config/public.pub
```

构建并启动用于资源分发的 KBS：
```shell
make passport-resource-kbs
make install-resource-kbs
resource-kbs --socket 127.0.0.1:50002 --insecure-http --auth-public-key config/public.pub
```

如果你希望以 HTTPS 服务器方式启动 KBS，可使用 `--private-key` 和 `--certificate` 指定服务端 HTTPS 私钥与证书。

### 上传资源数据

生成一个用于测试的资源数据文件：
```shell
cat > test/dummy_data << EOF
1234567890abcde
EOF
```

使用 `kbs-client` 将资源数据上传到 KBS 存储：

```shell
kbs-client --url http://127.0.0.1:50002 config --admin-token-file config/admin-token set-resource --resource-file test/dummy_data --path default/test/dummy
```

这里我们为该资源指定了一个自定义路径：`default/test/dummy`。

### 获取资源数据

生成 TEE 密钥对（RSA）：

```shell
openssl genrsa -traditional -out test/tee_key.pem 2048
openssl rsa -in test/tee_key.pem  -pubout -out test/tee_pubkey.pem
```

首先，使用 `kbs-client` 从负责签发令牌的 KBS 获取证明令牌（Attestation Token）：
```shell
kbs-client --url http://127.0.0.1:50001 attest --tee-key-file test/tee_key.pem > test/attestation_token
```

TEE 密钥的公钥部分会被包含在证明令牌的 claim 中，用于标识该 TEE。

然后使用 attestation_token 从 KBS 获取资源：
```shell
kbs-client --url http://127.0.0.1:50002 get-resource --attestation-token test/attestation_token --tee-key-file test/tee_key.pem --path default/test/dummy
```

## 自定义策略

### 证明策略
使用 `kbs-client` 为后台检查模式下的 KBS 设置自定义证明策略：
```shell
kbs-client --url http://127.0.0.1:50001 config --admin-token-file config/admin-token set-attestation-policy --policy-file /path/to/policy
```

其中 `/path/to/policy` 应替换为你策略文件的真实路径。

证明策略需要使用 [Open Policy Agent](https://www.openpolicyagent.org/) 所定义的 `rego` 语法来编写。
例如，你可以使用如下策略来校验 TDX 的 TCB SVN 与固件度量值：

```
package my_policy

import future.keywords.if

default allow = false

# The allowed reference values for the specific field in the quote
reference_tdx_tcb_svn = [ "03000500000000000000000000000000" ]
reference_tdx_mr_td = [ "abcd1234", "1234abcd", "a1b2c3d4" ]

allow if {
    input["tdx.quote.body.tcb_svn"] == reference_tdx_tcb_svn[_]
    input["tdx.quote.body.mr_td"] == reference_tdx_mrtd[_]
}
```

字段名请参考 [Attestation-Service](https://github.com/confidential-containers/attestation-service)。

### 资源策略
使用 `kbs-client` 为 KBS 设置自定义资源策略：
```shell
kbs-client --url http://127.0.0.1:50002 config --admin-token-file config/admin-token set-attestation-policy --policy-file /path/to/policy
```

其中 `/path/to/policy` 应替换为你策略文件的真实路径。

资源策略同样需要使用 [Open Policy Agent](https://www.openpolicyagent.org/) 所定义的 `rego` 语法来编写。

你可以阅读[默认资源策略文件](sample_policies/deny_all.rego)中的说明，了解资源策略的更多细节。

## 证明令牌证书

你可以通过配置文件来配置证明令牌（JWT）签名密钥的证书。

### 配置 AS 令牌的签名密钥与证书链

在 Issuer KBS 的配置文件中添加以下内容，以指定令牌签名密钥及其证书链，
两者都应为 PEM 格式。

#### Builtin AS 模式

在 KBS 自身的 TOML 配置文件中添加以下内容：
```toml
[as_config.attestation_token_broker.signer]
key_path = "/path/to/token-key.pem"
cert_path = "/path/to/token-cert-chain.pem"
```

关于 AS 签名材料如何映射到 KBS 信任锚，请参见[证明令牌校验](./docs/attestation_token_verification.md)。
配置参考请见 [config.md](./docs/config.md)。

#### gRPC AS 模式

在 gRPC AS 的 JSON 配置文件中添加以下内容：
```json
{
    ...

    "attestation_token_broker": {
        "duration_min": 5,
		"signer": {
			"key_path": "/path/to/token-key.pem",
			"cert_path": "/path/to/token-cert-chain.pem"
		}
    }
}
```

### 配置 KBS 的受信任根证书

证明令牌现在全部采用 JWT 格式。

在 Resource KBS 的配置文件中添加以下内容，以指定用于验证证明令牌可信性的受信任根证书（PEM 格式）
或 JWK 集合：

```toml
[attestation_token]
# Trust anchors (PEM) used to validate the `x5c` chain on a header-embedded `jwk`
# when `insecure_header_jwk` is false.
trusted_certs_paths = ["/path/to/trusted_cacert.pem"]

# Trusted JWK set (`file://` or `https://`) used when the JWT header carries `kid`
# instead of an embedded `jwk`.
trusted_jwk_sets = ["/url/to/trusted_jwk_set"]

# When false, header `jwk` keys must be endorsed via `x5c` and `trusted_certs_paths`.
# When true, the header `jwk` is trusted without endorsement checks (testing only).
insecure_header_jwk = false
```

当 `insecure_header_jwk` 为 `false` 且令牌头部包含 `jwk` 时，必须配置
`trusted_certs_paths`，否则校验会失败。使用 `kid` 的令牌则依赖
`trusted_jwk_sets`，不受 `insecure_header_jwk` 影响。

信任链搭建请参考[证明令牌校验](./docs/attestation_token_verification.md)，
配置参考请见 [config.md](./docs/config.md)。
