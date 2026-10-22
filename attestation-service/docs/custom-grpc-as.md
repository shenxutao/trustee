# gRPC 证明服务

> 说明：本文件是 `attestation-service/docs/grpc-as.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

`grpc-as` 是一个基于 gRPC 协议的证明服务应用。

目前支持以下类型的证据：
- `az-snp-vtpm`：Azure SNP vTPM
- `sgx`：Intel SGX
- `snp`：AMD SNP
- `tdx`：Intel TDX
- `cca`：Arm CCA
- `sample`：一个虚拟平台。仅用于测试与示例
- `csv`：Hygon CSV
- `az-tdx-vtpm`：Azure TDX vTPM
- `se`：IBM Secure Execution

## 快速开始

用户可以使用[gRPC CoCoAS 的社区版镜像](https://github.com/confidential-containers/trustee/pkgs/container/staged-images%2Fcoco-as-grpc)来校验证明报告。

```shell
# 在本地运行 gRPC CoCoAS 服务
docker run -d \
  -p 50004:50004 \
  ghcr.io/confidential-containers/staged-images/coco-as-grpc:latest
```

随后可以用一个证明请求来请求该服务。我们提供了一个[校验 SGX quote 的示例请求](../tests/coco-as/request.json)。

你可以使用[该工具](https://github.com/confidential-containers/guest-components/tree/main/attestation-agent/attester#evidence-getter-tool)在任意受支持的平台上生成报告。

```shell
# 使用如下命令安装 grpcurl
# go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

cd <path-to-attestation-service>

REQ=$(cat tests/coco-as/request.json)
grpcurl \
  -plaintext \
  -import-path ../protos \
  -proto ../protos/attestation.proto \
  -d @ 127.0.0.1:50004 attestation.AttestationService/AttestationEvaluate <<EOF
$REQ
EOF
```

然后会返回一个响应
```json
{
  "attestationToken": "eyJhbGciOiJSUzM4NCIsInR5cCI6IkpXVCJ9.eyJjdXN0b21pemVkX2NsYWltcyI6eyJ0ZXN0X2tleSI6InRlc3RfdmFsdWUifSwiZXZhbHVhdGlvbi1yZXBvcnRzIjpbeyJldmFsdWF0aW9uLXJlc3VsdCI6IntcImFsbG93XCI6dHJ1ZX0iLCJwb2xpY3ktaGFzaCI6ImMwZTc5Mjk2NzFmYjY3ODAzODdmNTQ3NjBkODRkNjVkMmNlOTYwOTNkZmIzM2VmZGEyMWY1ZWIwNWFmY2RhNzdiYmE0NDRjMDJjZDE3N2IyM2E1ZDM1MDcxNjcyNjE1NyIsInBvbGljeS1pZCI6ImRlZmF1bHQifV0sImV4cCI6MTcwMTY3Mjk2NSwiaXNzIjoiQ29Dby1BdHRlc3RhdGlvbi1TZXJ2aWNlIiwiandrIjp7ImFsZyI6IlJTMzg0IiwiZSI6IkFRQUIiLCJrdHkiOiJSU0EiLCJuIjoiMGhGUHdHNmdTSmJKV1NaTFR6SzRfVThHSWo5WVdYc3FZbDFjbUhveHlhQ05oQ3JCYVVUcW5KdHlPRVpMLVBDcWJBS2VXNGFCWnY1M3Zycm13OU41S2lHMHNOTVJOMUc1V2V2RTFNSEEzQU1qNlNlSWtwT1hzT01DNzJBNUZrZFIzRG1hM3dMaW5tZUVHYk9xZE5rN2IzMHdtWkRhVG13QTJJSjdnNVhPZk8zNTl6YWFLaDFRZDdPUXRkT2RfaV8tQlEzQlpEYnZ4R1ctWmRsdHVwWXBjRVQwWUZLSlE1NTdPSGtsOGMxT3BVdFc5ODlEQjM3d1BGTlRxM25oU3ZveDBwYWdDd3FwZ3JCYXVVUDBlOGlkX1VhSGFVZWlPd2tXc2UxdkdYQW55cFZqUlhhdERhS2dzbzZ5QjdGQ3pMUmRwM3JzWW1kd1lMaTdtMms3TkNPaE9RIn0sIm5iZiI6MTcwMTY3MjY2NSwidGNiLXN0YXR1cyI6eyJzZ3guYm9keS5hdHRyaWJ1dGVzLmZsYWdzIjoiMDcwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LmF0dHJpYnV0ZXMueGZybSI6ImU3MDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5jb25maWdfaWQiOiIwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LmNvbmZpZ19zdm4iOiIwMDAwIiwic2d4LmJvZHkuY3B1X3N2biI6IjA2MDYwYzBjZmZmZjAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkuaXN2X2V4dF9wcm9kX2lkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5pc3ZfZmFtaWx5X2lkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5pc3ZfcHJvZF9pZCI6IjAwMDAiLCJzZ3guYm9keS5pc3Zfc3ZuIjoiMDAwMCIsInNneC5ib2R5Lm1pc2Nfc2VsZWN0IjoiMDEwMDAwMDAiLCJzZ3guYm9keS5tcl9lbmNsYXZlIjoiOGYxNzNlNDYxM2ZmMDVjNTJhYWYwNDE2MmQyMzRlZGFlOGM5OTc3ZWFlNDdlYjIyOTlhZTE2YTU1MzAxMWM2OCIsInNneC5ib2R5Lm1yX3NpZ25lciI6IjgzZDcxOWU3N2RlYWNhMTQ3MGY2YmFmNjJhNGQ3NzQzMDNjODk5ZGI2OTAyMGY5YzcwZWUxZGZjMDhjN2NlOWUiLCJzZ3guYm9keS5yZXBvcnRfZGF0YSI6Ijc0NjU3Mzc0MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkucmVzZXJ2ZWQxIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwIiwic2d4LmJvZHkucmVzZXJ2ZWQyIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMCIsInNneC5ib2R5LnJlc2VydmVkMyI6IjAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guYm9keS5yZXNlcnZlZDQiOiIwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJzZ3guaGVhZGVyLmF0dF9rZXlfZGF0YV8wIjoiMDAwMDAwMDAiLCJzZ3guaGVhZGVyLmF0dF9rZXlfdHlwZSI6IjAyMDAiLCJzZ3guaGVhZGVyLnBjZV9zdm4iOiIwZDAwIiwic2d4LmhlYWRlci5xZV9zdm4iOiIwODAwIiwic2d4LmhlYWRlci51c2VyX2RhdGEiOiJkY2NkZTliMzFjZTg4NjA1NDgxNzNiYjRhMmE1N2ExNjAwMDAwMDAwIiwic2d4LmhlYWRlci52ZW5kb3JfaWQiOiI5MzlhNzIzM2Y3OWM0Y2E5OTQwYTBkYjM5NTdmMDYwNyIsInNneC5oZWFkZXIudmVyc2lvbiI6IjAzMDAifX0.SG7TUxm0E3yZs7rozijScMJIZTY8WVPZN3Yxu2CsW8HFE6lDLymdTzc1XTVrYb97PpGc6oCLwuLax786XHLN250SY_IW5GmR5WKRcYSGSQtOnYfsY7AMX3hvpV3rHGjP0QWZo_ezUp9yIbnJNwSprmFTzcNZkr2YNr1KmwWU-LhGSVCyviQwgtnqnhmQGwH-nHCcmgk0F3su_hdoFXImCggSHStXECAJ0cNjpAuTCsSQvrB4g3lM-dMii-D6a58uB_TGuOVf8Yqj9Gi6PrdxvIJZc1LSDDgo9uYuavNzunU3S3TkA2ZLDK4HIB0zDWfOnS2ZTrjyRdu5ZkzoGVK_YQ"
}
```

该值是一个 base64 编码的 JWT。该 JWT 的载荷展示在 [example.token.json](./example.token.json) 中。

更多配置项请参考[该文档](./config.md)。

## 进阶主题

### 从源码构建

对于进阶开发者，可以从源码构建一个 gRPC CoCoAS。步骤如下。

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
grpc-as --help
```

- 查看版本信息，运行：
```shell
grpc-as --version
```

启动证明服务并指定其 gRPC 服务的监听端口：
```shell
grpc-as --socket 127.0.0.1:50004
```

如果你希望查看运行时日志，运行：
```shell
RUST_LOG=debug grpc-as --socket 127.0.0.1:50004
```

或者你也可以在 podman 容器中运行该二进制：
```shell
# 构建 grpc-as 容器镜像
podman build \
    -t grpc-as \
    -f attestation-service/docker/as-grpc/Dockerfile \
    .

# 运行 grpc-as 容器
podman run \
    -d \
    -p 50004:50004 \
    --net host \
    grpc-as
```

#### 镜像构建

构建并运行容器镜像
```shell
git clone https://github.com/confidential-containers/trustee
cd trustee
docker build \
  -t coco-as:grpc \
  -f attestation-service/docker/as-grpc/Dockerfile \
  --build-arg VERIFIER=all-verifier \
  . 
```

### API（接口）

gRPC CoCo-AS 的 API 定义在 [proto](../../protos/attestation.proto) 中。

为了分发令牌校验密钥，`grpc-as` 还会在与 gRPC 服务相同的 socket 上提供一个标准的 HTTP JWKS 端点：

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

从与 gRPC 服务相同的监听端口获取密钥：

```shell
curl http://127.0.0.1:50004/.well-known/jwks.json
```
