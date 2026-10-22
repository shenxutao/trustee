# RESTful 证明服务

> 说明：本文件是 `attestation-service/docs/restful-as.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

`restful-as` 是一个基于 RESTful 的证明服务应用。

目前支持以下类型的证据：
- `az-snp-vtpm`：Azure SNP vTPM
- `sev`：AMD SEV（未实现）
- `sgx`：Intel SGX
- `snp`：AMD SNP
- `tdx`：Intel TDX
- `cca`：Arm CCA
- `sample`：一个虚拟平台。仅用于测试与示例
- `csv`：Hygon CSV
- `az-tdx-vtpm`：Azure TDX vTPM
- `se`：IBM Secure Execution

## 快速开始

用户可以使用[restful CoCoAS 的社区版镜像](https://github.com/confidential-containers/trustee/pkgs/container/staged-images%2Fcoco-as-restful)来校验证明报告。

```shell
# 在本地运行 restful CoCoAS 服务
docker run -d \
  -p 8080:8080 \
  ghcr.io/confidential-containers/staged-images/coco-as-restful:latest
```

随后可以用一个证明请求来请求该服务。我们提供了一个[校验 SGX quote 的示例请求](../tests/coco-as/request.json)。

你可以使用[该工具](https://github.com/confidential-containers/guest-components/tree/main/attestation-agent/attester#evidence-getter-tool)在任意受支持的平台上生成报告。

```shell
cd <path-to-attestation-service>
curl -k -X POST http://127.0.0.1:8080/attestation \
     -i \
     -H 'Content-Type: application/json' \
     -d @tests/coco-as/request.json
```

然后，HTTP 响应体中会返回一个令牌，形如
```plaintext
eyJhbGciOiJSUzM4NCIsInR5cCI6IkpXVCJ9.eyJjdXN0b21pemVkX2NsYWltcyI6eyJ0ZXN0X2tleSI6InRlc3RfdmFsdWUifSwiZXZhbHVhdGlvbi1yZXBvcnRzIjpbeyJldmFsdWF0aW9uLXJlc3VsdCI6IntcImFsbG93XCI6dHJ1ZX0iLCJwb2xpY3ktaGFzaCI6ImMwZTc5Mjk2NzFmYjY3ODAzODdmNTQ3NjBkODRkNjVkMmNlOTYwOTNkZmIzM2VmZGEyMWY1ZWIwNWFmY2RhNzdiYmE0NDRjMDJjZDE3N2IyM2E1ZDM1MDcxNjcyNjE1NyIsInBvbGljeS1pZCI6ImRlZmF1bHQifV0sImV4cCI6MTcwMTY3Mjk2NSwiaXNzIjoiQ29Dby1BdHRlc3RhdGlvbi1TZXJ2aWNlIiwiandrIjp7ImFsZyI6IlJTMzg0IiwiZSI6IkFRQUIiLCJrdHkiOiJSU0EiLCJuIjoiMGhGUHdHNmdTSmJKV1NaTFR6SzRfVThHSWo5WVdYc3FZbDFjbUhveHlhQ05oQ3JCYVVUcW5KdHlPRVpMLVBDcWJBS2VXNGFCWnY1M3Zycm13OU41S2lHMHNOTVJOMUc1V2V2RTFNSEEzQU1qNlNlSWtwT1hzT01DNzJBNUZrZFIzRG1hM3dMaW5tZUVHYk9xZE5rN2IzMHdtWkRhVG13QTJJSjdnNVhPZk8zNTl6YWFLaDFRZDdPUXRkT2RfaV8tQlEzQlpEYnZ4R1ctWmRsdHVwWXBjRVQwWUZLSlE1NTdPSGtsOGMxT3BVdFc5ODlEQjM3d1BGTlRxM25oU3ZveDBwYWdDd3FwZ3JCYXVVUDBlOGlkX1VhSGFVZWlPd2tXc2UxdkdYQW55cFZqUlhhdERhS2dzbzZ5QjdGQ3pMUmRwM3JzWW1kd1lMaTdtMms3TkNPaE9RIn0sIm5iZiI6MTcwMTY3MjY2NSwidGNiLXN0YXR1cyI6eyJzZ3guYm9keS5hdHRyaWJ1dGVzLmZsYWdzIjoiMDcwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LmF0dHJpYnV0ZXMueGZybSI6ImU3MDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5jb25maWdfaWQiOiIwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LmNvbmZpZ19zdm4iOiIwMDAwIiwic2d4LmJvZHkuY3B1X3N2biI6IjA2MDYwYzBjZmZmZjAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkuaXN2X2V4dF9wcm9kX2lkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5pc3ZfZmFtaWx5X2lkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5pc3ZfcHJvZF9pZCI6IjAwMDAiLCJzZ3guYm9keS5pc3Zfc3ZuIjoiMDAwMCIsInNneC5ib2R5Lm1pc2Nfc2VsZWN0IjoiMDEwMDAwMDAiLCJzZ3guYm9keS5tcl9lbmNsYXZlIjoiOGYxNzNlNDYxM2ZmMDVjNTJhYWYwNDE2MmQyMzRlZGFlOGM5OTc3ZWFlNDdlYjIyOTlhZTE2YTU1MzAxMWM2OCIsInNneC5ib2R5Lm1yX3NpZ25lciI6IjgzZDcxOWU3N2RlYWNhMTQ3MGY2YmFmNjJhNGQ3NzQzMDNjODk5ZGI2OTAyMGY5YzcwZWUxZGZjMDhjN2NlOWUiLCJzZ3guYm9keS5yZXBvcnRfZGF0YSI6Ijc0NjU3Mzc0MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkucmVzZXJ2ZWQxIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkucmVzZXJ2ZWQyIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LnJlc2VydmVkMyI6IjAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5yZXNlcnZlZDQiOiIwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guaGVhZGVyLmF0dF9rZXlfZGF0YV8wIjoiMDAwMDAwMDAiLCJzZ3guaGVhZGVyLmF0dF9rZXlfdHlwZSI6IjAyMDAiLCJzZ3guaGVhZGVyLnBjZV9zdm4iOiIwZDAwIiwic2d4LmhlYWRlci5xZV9zdm4iOiIwODAwIiwic2d4LmhlYWRlci51c2VyX2RhdGEiOiJkY2NkZTliMzFjZTg4NjA1NDgxNzNiYjRhMmE1N2ExNjAwMDAwMDAwIiwic2d4LmhlYWRlci52ZW5kb3JfaWQiOiI5MzlhNzIzM2Y3OWM0Y2E5OTQwYTBkYjM5NTdmMDYwNyIsInNneC5oZWFkZXIudmVyc2lvbiI6IjAzMDAifX0.SG7TUxm0E3yZs7rozijScMJIZTY8WVPZN3Yxu2CsW8HFE6lDLymdTzc1XTVrYb97PpGc6oCLwuLax786XHLN250SY_IW5GmR5WKRcYSGSQtOnYfsY7AMX3hvpV3rHGjP0QWZo_ezUp9yIbnJNwSprmFTzcNZkr2YNr1KmwWU-LhGSVCyviQwgtnqnhmQGwH-nHCcmgk0F3su_hdoFXImCggSHStXECAJ0cNjpAuTCsSQvrB4g3lM-dMii-D6a58uB_TGuOVf8Yqj9Gi6PrdxvIJZc1LSDDgo9uYuavNzunU3S3TkA2ZLDK4HIB0zDWfOnS2ZTrjyRdu5ZkzoGVK_YQ
```

该值是一个 base64 编码的 JWT。该 JWT 的载荷展示在 [example.token.json](./example.token.json) 中。


更多配置项请参考[该文档](./config.md)。

## 进阶主题

### 从源码构建

对于进阶开发者，可以从源码构建一个 restful CoCoAS。步骤如下。

#### 二进制构建与运行

构建并安装二进制
```shell
git clone https://github.com/confidential-containers/trustee
cd trustee/attestation-service
WORKDIR=$(pwd)
make VERIFIER=all-verifier && make install

# 你可以通过修改 VERIFIER 的取值来使用不同的校验器
```

- 查看帮助信息，运行：
```shell
restful-as --help
```

- 查看版本信息，运行：
```shell
restful-as --version
```

启动证明服务并指定其 Web 服务的监听端口：
```shell
restful-as --socket 127.0.0.1:8080
```

如果你希望查看运行时日志，运行：
```shell
RUST_LOG=debug restful-as --socket 127.0.0.1:8080 -c config.json
```

#### 镜像构建

构建并运行容器镜像
```shell
git clone https://github.com/confidential-containers/trustee
cd trustee
docker build \
  -t coco-as:restful \
  -f attestation-service/docker/as-restful/Dockerfile \
  --build-arg VERIFIER=all-verifier \
  .
```

或者你也可以在 podman 容器中运行该二进制：
```shell
# 构建 restful-as 容器镜像
podman build \
    -t restful-as \
    -f attestation-service/docker/as-restful/Dockerfile \
    .

# 运行 restful-as 容器
podman run \
    -d \
    -p 50004:50004 \
    --net host \
    restful-as
```

### HTTPS 支持

生成自签名证书（可选）

```shell
# 生成一个 RSA 私钥
openssl genrsa -out private_key.key 2048

# 生成公钥证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout private_key.key -out certificate.crt
```

启动 HTTPS CoCo-AS 服务
```shell
RUST_LOG=debug restful-as \
    --socket 127.0.0.1:8080 \
    -c config.json \
    --https-pubkey-cert certificate.crt \
    --https-prikey private_key.key
```

### API（接口）

RESTful CoCo-AS 的端点如下：
- `/attestation`：接收证据校验请求。请求 POST 载荷形如
```json
{
    "verification_requests": [ // 其中每个 JSON 对象都是一个单独的证明请求。所有请求会以给定策略
                               // 一并处理。
        {
            "tee": "sgx", // TEE 类型。
            "evidence": "YWFhCg==...", // base64 编码的证据，使用 URL SAFE NO PAD，
            "runtime_data": {           // `runtime_data` 是可选的。如果提供，将校验运行时数据绑定。
                                        // `raw` 与 `structured` 两个字段互斥。
                "raw": "YWFhCg==...",   // Base64 编码的运行时数据字节串。整个字符串会被 base64
                                        // 解码。解码结果随后会被累积为一个摘要，用作期望的运行时数据，
                                        // 以与证据内部的对应值进行比对。
                                        //
                                        // 字母表为 URL_SAFE_NO_PAD。
                                        // 定义见 https://datatracker.ietf.org/doc/html/rfc4648#section-5
                                        
                "structured": {}        // JSON map 形式的运行时数据。CoCoAS 会按字段名的字典序
                                        // 重新排列该数据 JSON 对象的每一层，然后序列化输出为紧凑字符串，
                                        // 并对整体做哈希计算，以与证据内部的对应值进行比对。哈希算法由
                                        // `runtime_data_hash_algorithm` 定义。
                                        //
                                        // 校验完成后，结构化运行时数据字段会被包含进令牌声明项中。
            }, 
            "init_data": {              // `init_data` 是可选的。如果提供，将校验 init data 绑定。
                                        // `raw` 与 `structured` 两个字段互斥。
                "init_data_digest": "YWFhCg==...",  // Base64 编码的 init data 摘要字节串。整个字符串会被 base64
                                                    // 解码。解码结果随后会被累积为一个摘要，用作期望的 init data，
                                                    // 以与证据内部的对应值进行比对。
                                                    //
                                                    // 字母表为 URL_SAFE_NO_PAD。
                                                    // 定义见 https://datatracker.ietf.org/doc/html/rfc4648#section-5
                                        
                "init_data_toml": "algorithm = \"sha384\"..."       // Init data TOML。CoCoAS 会对整体做哈希计算，
                                                                    // 以与证据内部的对应值进行比对。所使用的哈希算法
                                                                    // 位于 Initdata Toml 的元数据中。
                                                                    // 
                                                                    // 参见 https://github.com/confidential-containers/trustee/blob/main/kbs/docs/initdata.md#toml-version
                                                                    // 
                                                                    // 校验完成后，init data 字段的 `.data` 部分
                                                                    // 会被包含进令牌声明项中。
            }, 
            "runtime_data_hash_algorithm": "sha384",// 用于计算运行时数据的哈希算法。目前可以为 
                                                    // "sha256"、"sha384" 或 "sha512"。若未指定，将选择 "sha384"。
            "init_data_hash_algorithm": "sha384",   // 用于计算 init data 的哈希算法。目前可以为 
                                                    // "sha256"、"sha384" 或 "sha512"。若未指定，将选择 "sha384"。
        }
    ],
    "policy_ids": ["default", "policy-1"]           // 用于校验证据的策略 ID 列表。如果
                                                    // 未提供，将使用 "default"。
}
```
- `/policy`：接收策略设置请求。请求 POST 载荷形如
```json
{
    "type": "rego",         // 策略类型
    "policy_id": "yyyyy",   // 策略 id 的原始字符串
    "policy": "xxxxx"       // base64 编码的策略内容
}
```
- `/.well-known/jwks.json`：返回一个 JWK Set 文档，用于校验由该 AS 实例签名的证明令牌。响应示例：
```json
{
    "keys": [
        {
            "alg": "ES256",
            "kty": "EC",
            "crv": "P-256",
            "x": "<base64url>",
            "y": "<base64url>"
        }
    ]
}
```
