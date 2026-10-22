
# 搭配 IBM SE 验证器的 KBS

> 说明：本文件是 `deps/verifier/src/se/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../custom-文档清单.md)。

本文档用于指导开发者在本地运行搭配 IBM SE 验证器的 KBS，仅供开发目的使用。

## 目录

- [部署搭配 IBM SE 验证器的 KBS](#deployment-of-kbs-with-ibm-se-verifier)
- [管理员认证](#admin-authentication)
- [为 IBM SE 验证器设置认证策略](#set-attestation-policy)



# 部署搭配 IBM SE 验证器的 KBS

本节介绍不启用 rvps 校验情况下的 KBS 部署方式。

## 生成 RSA 密钥
按如下命令生成 RSA 4096 密钥对：
```bash
openssl genrsa -aes256 -passout pass:test1234 -out encrypt_key-psw.pem 4096
openssl rsa -in encrypt_key-psw.pem -passin pass:test1234 -pubout -out encrypt_key.pub
openssl rsa -in encrypt_key-psw.pem -out encrypt_key.pem
```


## 下载证书与 CRL
从以下地址下载这些材料：https://www.ibm.com/support/resourcelink/api/content/public/secure-execution-gen2.html
其中包括：

### 证书（Certs）
ibm-z-host-key-signing-gen2.crt
DigiCertCA.crt

### CRL
ibm-z-host-key-gen2.crl
DigiCertTrustedRootG4.crl
DigiCertTrustedG4CodeSigningRSA4096SHA3842021CA1.crl

注意：`DigiCertTrustedRootG4.crl` 和 `DigiCertTrustedG4CodeSigningRSA4096SHA3842021CA1.crl` 来自如下命令的输出：
```bash
# openssl x509 -in DigiCertCA.crt --text --noout |grep crl
                  URI:http://crl3.digicert.com/DigiCertTrustedRootG4.crl
# openssl x509 -in ibm-z-host-key-signing-gen2.crt --text --noout |grep crl
                  URI:http://crl3.digicert.com/DigiCertTrustedG4CodeSigningRSA4096SHA3842021CA1.crl
                  URI:http://crl4.digicert.com/DigiCertTrustedG4CodeSigningRSA4096SHA3842021CA1.crl
```

## 下载 HKD
按照 https://www.ibm.com/docs/en/linux-on-systems?topic=execution-obtain-host-key-document 下载 IBM Secure Execution 主机密钥文档（Host Key Document）。

## 获取 SE 头部（Header）
按照[生成 IBM Secure Execution 镜像](https://www.ibm.com/docs/en/cic/1.2.5?topic=kvm-generate-secure-execution-image)构建 `se.img`，并通过如下命令提取 hdr.bin。
```bash
./pvextract-hdr -o hdr.bin se.img
```

获取 `pvextract-hdr` 请参考 [ibm-s390-linux](https://github.com/ibm-s390-linux/s390-tools/blob/v2.33.1/rust/pvattest/tools/pvextract-hdr)。

## 生成管理员 JWT 密钥

生成一个 Ed25519 密钥对，用于签名和校验**管理 API** 的 bearer JWT（不涉及 HTTPS 证书或认证令牌）。

```bash
openssl genpkey -algorithm ed25519 > kbs.key
openssl pkey -in kbs.key -pubout -out kbs.pem
```

与之配套的 `kbs-config.toml` 配置以及如何创建 `admin-token` 文件，请参见[管理员认证](#admin-authentication)。

## （方案 1）以程序方式启动 KBS

- 构建 KBS
```bash
cargo install --locked --debug --path kbs --no-default-features --features coco-as-builtin,resource,opa
```

- 准备上述获取到的材料，结构类似如下：
```
/run/confidential-containers/ibmse#
.
├── certs
│   ├── ibm-z-host-key-signing-gen2.crt
|   └── DigiCertCA.crt
├── crls
│   └── ibm-z-host-key-gen2.crl
│   └── DigiCertTrustedRootG4.crl
│   └── DigiCertTrustedG4CodeSigningRSA4096SHA3842021CA1.crl
├── hdr
│   └── hdr.bin
├── hkds
│   └── HKD-3931-0275D38.crt
└── rsa
    ├── encrypt_key.pem
    └── encrypt_key.pub
```

> 注意：另一种做法是改用 [ibmse.rs](./ibmse.rs) 中列出的系统环境变量来覆盖上述文件。

- 准备 `kbs-config.toml`，内容类似如下：
```toml
[http_server]
sockets = ["0.0.0.0:8080"]
# Ideally we should use some solution like cert-manager to issue let's encrypt based certificate:
# https://cert-manager.io/docs/configuration/acme/
insecure_http = true

[attestation_token]
insecure_header_jwk = true

[attestation_service]
type = "coco_as_builtin"

[attestation_service.attestation_token_broker]
duration_min = 5

[attestation_service.rvps_config]
type = "BuiltIn"

[admin]
authorization_mode = "AuthenticatedAuthorization"

[admin.authentication.bearer_jwt]
identity_providers = [
  { issuer = "ibmse-dev", public_key_uri = "./kbs.pem" },
]

[admin.authorization.regex_acl]
acls = [{ role = "admin", allowed_endpoints = "^/kbs/.+$" }]

[storage_backend]
storage_type = "LocalFs"

[storage_backend.backends.local_fs]
dir_path = "/opt/confidential-containers/kbs"

[[plugins]]
name = "resource"
storage_backend_type = "kvstorage"
```

> 仅用于本地调试时，可以在 `[admin]` 下设置 `authorization_mode = "InsecureAllowAll"` 并跳过管理员令牌生成。下面的示例使用的是 `AuthenticatedAuthorization`。

- 启动 KBS 程序
```bash
export RUST_LOG=debug
export SE_SKIP_CERTS_VERIFICATION=true
./kbs --config-file ./kbs-config.toml
```

> 注意：`export SE_SKIP_CERTS_VERIFICATION=true` 仅在开发机上需要。如需离线校验证书，请使用 `export CERTS_OFFLINE_VERIFICATION=true`。

## （方案 2）通过 docker-compose 启动 KBS
- 构建 docker 镜像
```
DOCKER_BUILDKIT=1 docker build --build-arg --build-arg ARCH="s390x" -t ghcr.io/confidential-containers/staged-images/kbs:latest . -f kbs/docker/Dockerfile
```

- 准备 docker compose 文件，内容类似如下：
```
services:
  kbs:
    image: ghcr.io/confidential-containers/staged-images/kbs:latest
    command: [
        "/usr/local/bin/kbs",
        "--config-file",
        "/etc/kbs-config.toml",
      ]
    restart: always # keep the server running
    environment:
      - RUST_LOG=debug
      - SE_SKIP_CERTS_VERIFICATION=true
    ports:
      - "8080:8080"
    volumes:
      - ./data/kbs-storage:/opt/confidential-containers/kbs/repository:rw
      - ./data/attestation-service:/opt/confidential-containers/attestation-service:rw
      - ./kbs.pem:/kbs/kbs.pem
      - ./kbs-config.toml:/etc/kbs-config.toml
      - ./data/hkds:/run/confidential-containers/ibmse/hkds
      - ./data/certs:/run/confidential-containers/ibmse/certs
      - ./data/crls:/run/confidential-containers/ibmse/crls
      - ./data/hdr.bin:/run/confidential-containers/ibmse/hdr/hdr.bin
      - ./data/rsa/encrypt_key.pem:/run/confidential-containers/ibmse/rsa/encrypt_key.pem
      - ./data/rsa/encrypt_key.pub:/run/confidential-containers/ibmse/rsa/encrypt_key.pub
```

- 为容器准备 `kbs-config.toml`（其中 `public_key_uri` 必须与 `./kbs.pem:/kbs/kbs.pem` 这个挂载项相对应）：

```toml
[http_server]
sockets = ["0.0.0.0:8080"]
insecure_http = true

[attestation_token]
insecure_header_jwk = true

[attestation_service]
type = "coco_as_builtin"

[attestation_service.attestation_token_broker]
duration_min = 5

[attestation_service.rvps_config]
type = "BuiltIn"

[admin]
authorization_mode = "AuthenticatedAuthorization"

[admin.authentication.bearer_jwt]
identity_providers = [
  { issuer = "ibmse-dev", public_key_uri = "/kbs/kbs.pem" },
]

[admin.authorization.regex_acl]
acls = [{ role = "admin", allowed_endpoints = "^/kbs/.+$" }]

[storage_backend]
storage_type = "LocalFs"

[storage_backend.backends.local_fs]
dir_path = "/opt/confidential-containers/kbs"

[[plugins]]
name = "resource"
storage_backend_type = "kvstorage"
```

> 注意：compose 文件中的 `SE_SKIP_CERTS_VERIFICATION=true` 仅在开发机上需要。如需离校验证书，请使用 `CERTS_OFFLINE_VERIFICATION=true`。

- 准备相关材料，结构类似如下：
```
.
├── data
│   ├── attestation-service
│   │   ├── opa
│   │   │   └── default.rego
│   ├── certs
│   │   ├── ibm-z-host-key-signing-gen2.crt
│   │   └── DigiCertCA.crt
│   ├── crls
│   │   └── ibm-z-host-key-gen2.crl
│   │   └── DigiCertTrustedRootG4.crl
│   │   └── DigiCertTrustedG4CodeSigningRSA4096SHA3842021CA1.crl
│   ├── hdr.bin
│   ├── hkds
│   │   └── HKD-3931-0275D38.crt
│   ├── kbs-storage
│   │   ├── default
│   │   └── one
│   │       └── two
│   │           └── key
│   └── rsa
│       ├── encrypt_key.pem
│       └── encrypt_key.pub
├── admin-token
├── docker-compose.yaml
├── kbs-config.toml
├── kbs.key
└── kbs.pem
```

- 以 docker compose 应用方式启动 KBS
```bash
docker-compose up -d
docker-compose logs kbs
docker-compose down
```


# 管理员认证

KBS 管理 API（例如 `set-attestation-policy`）受 `[admin]` 配置保护。详见 [KBS Admin 模块](../../../../kbs/docs/admin.md) 与 [Admin API 配置](../../../../kbs/docs/config.md#admin-api-configuration)。

本指南使用 `AuthenticatedAuthorization`：KBS 会校验 bearer JWT，并依据正则 ACL 规则检查其中的 `role` claim。仅设置 `authorization_mode = "AuthenticatedAuthorization"` 是不够的 —— 还必须像上面 `kbs-config.toml` 示例那样同时配置 `[admin.authentication.bearer_jwt]` 和 `[admin.authorization.regex_acl]`。

## 生成管理员令牌

创建 `kbs.key` / `kbs.pem` 之后，签名一个长期有效的管理员 JWT。其中 `iss` 和 `aud` 的取值必须与 `kbs-config.toml` 中的 `identity_providers` 相匹配；该 JWT **必须**包含一个能与某条 ACL 条目（此处为 `admin`）相匹配的 `role` claim。

```bash
b64url() { openssl base64 -A | tr '+/' '-_' | tr -d '='; }

header='{"alg":"EdDSA","typ":"JWT"}'
iat=$(date +%s)
exp=$((iat + 315360000)) # 10 years
payload="{\"iss\":\"ibmse-dev\",\"role\":\"admin\",\"aud\":[\"kbs\"],\"iat\":${iat},\"exp\":${exp}}"

h64=$(printf '%s' "$header" | b64url)
p64=$(printf '%s' "$payload" | b64url)
signing_input=$(mktemp)
printf '%s' "${h64}.${p64}" > "${signing_input}"
sig=$(openssl pkeyutl -sign -inkey kbs.key -rawin -in "${signing_input}" | b64url)
rm -f "${signing_input}"
printf '%s.%s.%s\n' "$h64" "$p64" "$sig" > admin-token
```

通过 `--admin-token-file ./admin-token`（或使用绝对路径）将该文件传给 `kbs-client`。此独立 SE 部署不使用 trustee `docker compose` 的 `setup` 服务，因此除非你自行创建该路径，否则 `kbs-client` 中自动查找 `kbs/config/docker-compose/admin-token` 的行为在此不适用。


# 设置认证策略

本节介绍如何设置认证策略。

### 获取 IBM SE 的认证策略字段

在 s390x 实例上使用 [se_parse_hdr.py](se_parse_hdr.py) 获取用于认证策略的 IBM SE 字段。

```bash
python3 se_parse_hdr.py hdr.bin HKD-3931.crt

...
  ================================================
  se.image_phkh: xxx
  se.version: 256
  se.tag: xxx
  se.attestation_phkh: xxx
```

我们得到以下字段，并将在 rvps 中设置这些字段以用于认证策略。
`se.version: 256`
`se.tag: xxx`
`se.attestation_phkh: xxx`
`se.image_phkh: xxx`


### 设置认证策略

#### 生成认证策略文件
```bash
cat << EOF > ibmse-policy.rego
package policy
import rego.v1
default allow = false

converted_version := sprintf("%v", [input["se.version"]])

allow if {
    input["se.attestation_phkh"] == "xxx"
    input["se.image_phkh"] == "xxx"
    input["se.tag"] == "xxx"
    input["se.user_data"] == "xxx"
    converted_version == "256"
}
EOF
```

其中 `se.version`、`se.attestation_phkh`、`se.image_phkh` 和 `se.tag` 的取值来自[获取某个 IBM SE 镜像的 rvps 字段](#retrive-the-rvps-field-for-an-ibm-se-image)。`se.user_data` 的取值来自 [initdata](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/docs/initdata.md)。如果未使用 `initdata`，请删除 `input["se.user_data"] == "xxx"` 这一行。

#### 设置认证策略
```bash
kbs-client --url http://127.0.0.1:8080 \
  config --admin-token-file ./admin-token \
  set-attestation-policy --policy-file ./ibmse-policy.rego
```
