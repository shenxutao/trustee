# CCA 校验器

> 说明：本文件是 `deps/verifier/test_data/cca/conf/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../../../custom-文档清单.md)。

| 属性 | 类型 | 说明 | 是否必需 | 默认值 |
|----|----|----|----|----|
| `type` | `remote` 或 `local` | CCA 校验器的类型 | 是 | none |

* `local`：使用本地校验器
* `remote`：使用一个远端校验器

## 本地校验器

当 `type` 为 `local` 时，可接受的 CCA 平台参考值与信任锚通过两个 JSON 文件配置，
它们使用 [ccatoken 的 store](https://github.com/veraison/rust-ccatoken/blob/main/src/store/data-model.cddl) 格式。

| 属性 | 类型 | 说明 | 是否必需 | 默认值 |
|----|----|----|----|----|
| `ta-store` | Path | CCA 信任锚存储的位置 | 是 | none |
| `rv-store` | Path | CCA 参考值存储的位置 | 是 | none |

示例

~~~json
{
    "cca-verifier": {
        "type": "local",
        "ta-store": "/etc/trustee/as/cca/ta-store.json",
        "rv-store": "/etc/trustee/as/cca/rv-store.json"
    }
}
~~~

## 远端校验器

当 `type` 为 `remote` 时，校验器的位置以及（可选的）信任锚通过以下属性提供：

| 属性 | 类型 | 说明 | 是否必需 | 默认值 |
|----|----|----|----|----|
| `address` | URI | 校验器的 URI | 是 | none |
| `ca-cert` | Path | （自定义）根 CA 证书的位置 | 否 | none |

该校验器必须实现 Veraison 的 ["challenge-response"](https://github.com/veraison/docs/tree/main/api/challenge-response)
与 ["well-known"](https://github.com/veraison/docs/tree/main/api/well-known) API。

使用 [Linaro 的公开测试校验器](http://veraison.test.linaro.org/)的部署示例：

~~~json
{
    "cca-verifier": {
        "type": "remote",
        "address": "https://veraison.test.linaro.org:8443"
    }
}
~~~

使用带有自定义根 CA 的自定义 Veraison 校验器的部署示例：

~~~json
{
    "cca-verifier": {
        "type": "remote",
        "address": "https://localhost:8443",
        "ca-cert": "/etc/trustee/as/cca/rootCA.pem"
    }
}
~~~
