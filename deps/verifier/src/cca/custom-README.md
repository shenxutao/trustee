将 `TRUSTEE_SRC` 设置为已检出的 trustee 仓库的绝对路径：

```sh
export TRUSTEE_SRC="__set_me__"
cd "${TRUSTEE_SRC}"
```

# 构建 "grpc" 模式

> 说明：本文件是 `deps/verifier/src/cca/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../custom-文档清单.md)。

在顶层目录下执行：

```sh
make -C attestation-service build VERIFIER=cca-verifier
make -C rvps build
make -C kbs build AS_TYPE=coco-as-grpc
```

# 一次性环境准备

* Trustee 服务的工作目录

```sh
TRUSTEE_WDIR="/opt/confidential-containers"

sudo mkdir -p "${TRUSTEE_WDIR}/attestation-service/cca/opa"
sudo chown -R $(id -un):$(id -gn) "${TRUSTEE_WDIR}"
```

* `/etc/hosts`

```sh
sudo sh -c 'printf "# trustee services\n127.0.0.1 kbs rvps grpc-as\n" >> /etc/hosts'
```

* 背书（endorsement）存储

```sh
cp ${TRUSTEE_SRC}/deps/verifier/test_data/cca/conf/tastore.json "${TRUSTEE_WDIR}/attestation-service/cca/"
cp ${TRUSTEE_SRC}/deps/verifier/test_data/cca/conf/rvstore.json "${TRUSTEE_WDIR}/attestation-service/cca/"
```

* CCA OPA 策略

```sh
cp ${TRUSTEE_SRC}/deps/verifier/test_data/cca/conf/default.rego "${TRUSTEE_WDIR}/attestation-service/cca/opa/"
```


# 运行

## `kbs`

```sh
RUST_LOG=debug ${TRUSTEE_SRC}/target/release/kbs -c ${TRUSTEE_SRC}/kbs/conf/kbs-config-grpc.toml
```

## `rvps`

`rvps` 必须（MUST）在 `as` 之前启动

```sh
RUST_LOG=debug ${TRUSTEE_SRC}/target/release/rvps -c ${TRUSTEE_SRC}/kbs/conf/rvps.json
```

`rvps` 启动后，按如下方式配置所需的参考值（reference values）：

```sh
( cd ${TRUSTEE_SRC}/deps/verifier/test_data/cca/scripts && \
  RIM="rof79/EF5xp4J8tvCJaInHPSvq9IMNT5ywXj6Ww/x7T6oEyvNqJr0JTPnHtzhhfkwYmq7hhN7dpdegPDnb2sfA==" ./endorse-rim.sh )
```

## `as`

* 本地验证器（local verifier）：

```sh
CCA_CONFIG_FILE=${TRUSTEE_SRC}/deps/verifier/test_data/cca/conf/cca-config-local.json \
RUST_LOG=debug \
  ${TRUSTEE_SRC}/target/release/grpc-as \
    -c ${TRUSTEE_SRC}/deps/verifier/test_data/cca/conf/as-config.json \
    -s 127.0.0.1:50004
```

* 远程验证器（remote verifier）：

```sh
CCA_CONFIG_FILE=${TRUSTEE_SRC}/deps/verifier/test_data/cca/conf/cca-config-remote.json \
RUST_LOG=debug \
  ${TRUSTEE_SRC}/target/debug/grpc-as \
    -c ${TRUSTEE_SRC}/deps/verifier/test_data/cca/conf/as-config.json \
    -s 127.0.0.1:50004
```

## `kbc` 模拟器

* 安装 [`evcli`](https://github.com/veraison/evcli)

```sh
go install github.com/veraison/evcli/v2@latest
```

* 运行密钥代理服务客户端模拟器：

```sh
( cd ${TRUSTEE_SRC}/deps/verifier/test_data/cca/scripts && ./kbs-test-client.sh )
```
