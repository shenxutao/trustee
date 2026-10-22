# 使用自签名证书启用 HTTPS

> 说明：本文件是 `kbs/docs/self-signed-https.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

本指南将达成以下目标
- 生成一个私钥，以及该私钥公钥部分的自签名 HTTPS 证书。
- 使用该私钥与证书启动 KBS
- 使用 KBS 客户端工具访问该 KBS HTTPS 服务

## 生成自签名证书

```bash
# 编辑一份 crt 配置。你可以把以下各项改成任意你想要的值
cat << EOF > localhost.conf
[req]
default_bits       = 2048
default_keyfile    = localhost.key
distinguished_name = req_distinguished_name
req_extensions     = req_ext
x509_extensions    = v3_ca

[req_distinguished_name]
countryName                 = Country Name (2 letter code)
countryName_default         = CN
stateOrProvinceName         = State or Province Name (full name)
stateOrProvinceName_default = Zhejiang
localityName                = Locality Name (eg, city)
localityName_default        = Hangzhou
organizationName            = Organization Name (eg, company)
organizationName_default    = localhost
organizationalUnitName      = organizationalunit
organizationalUnitName_default = Development
commonName                  = Common Name (e.g. server FQDN or YOUR name)
commonName_default          = localhost
commonName_max              = 64

[req_ext]
subjectAltName = @alt_names

[v3_ca]
subjectAltName = @alt_names

[alt_names]
DNS.1   = localhost
DNS.2   = 127.0.0.1
EOF

# 生成私钥与自签名证书
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout localhost.key \
  -out localhost.crt \
  -config localhost.conf \
  -passin pass:
```
## 生成资源获取密钥对与 admin 令牌

```bash
openssl genpkey -algorithm ed25519 > private.key
openssl pkey -in private.key -pubout -out public.pub
b64url() { openssl base64 -A | tr '+/' '-_' | tr -d '='; }
header='{"alg":"EdDSA","typ":"JWT"}'
iat=$(date +%s); exp=$((iat + 315360000))
payload="{\"issuer\":\"admin\",\"subject\":\"admin\",\"audiences\":[],\"iat\":${iat},\"exp\":${exp}}"
h64=$(printf '%s' "$header" | b64url)
p64=$(printf '%s' "$payload" | b64url)
sig=$(printf '%s' "${h64}.${p64}" | openssl pkeyutl -sign -inkey private.key -rawin | b64url)
printf '%s.%s.%s\n' "$h64" "$p64" "$sig" > admin-token
```

## 启动 KBS 服务
准备一份 `kbs-config.toml`
```bash
cat << EOF > kbs-config.toml
[http_server]
sockets = ["0.0.0.0:8080"]
private_key = "/etc/key.pem"
certificate = "/etc/cert.pem"
insecure_http = false

[admin]
authorization_mode = "InsecureAllowAll"

[attestation_token]
insecure_header_jwk = true

[policy_engine]
policy_path = "/opa/confidential-containers/kbs/policy.rego"

[attestation_service]
type = "coco_as_builtin"
work_dir = "/opt/confidential-containers/attestation-service"
policy_engine = "opa"

    [attestation_service.attestation_token_broker]
    type = "Ear"
    duration_min = 5

    [attestation_service.rvps_config]
    type = "BuiltIn"

    storage_type = "LocalFs"

[[plugins]]
name = "resource"
type = "LocalFs"
dir_path = "/opt/confidential-containers/kbs/repository"
EOF
```

使用 docker 运行 KBS-built-in-as
```bash
docker run -it --rm \
  -v $(pwd)/kbs-config.toml:/etc/kbs-config.toml \
  -v $(pwd)/localhost.key:/etc/key.pem \
  -v $(pwd)/localhost.crt:/etc/cert.pem \
  -v $(pwd)/public.pub:/etc/public.pub \
  --env RUST_LOG=debug \
  -p 8080:8080 \
  kbs:coco-as \
  kbs --config-file /etc/kbs-config.toml
```

`kbs:coco-as` 由 `docker build -t kbs:coco-as . -f kbs/docker/Dockerfile` 构建而来，
也可以使用来自 https://github.com/confidential-containers/kbs/pkgs/container/staged-images%2Fkbs 的暂存镜像。

## 使用客户端工具访问

```bash
echo testdata > dummy_data
kbs-client --cert-file localhost.crt \
  --url https://localhost:8080 \
  config \
  --admin-token-file admin-token \
  set-resource \
  --resource-file dummy_data \
  --path default/test/dummy
```

其结果为
```plaintext
Set resource success
```

表明操作成功。

**端口映射非常重要，因为证书中的 FQDN 被设为 `localhost`。** 我们必须确保客户端工具上使用的 URI 与证书 CommonName 中的取值一致。
