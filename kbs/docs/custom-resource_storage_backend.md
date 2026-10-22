# 资源存储后端

> 说明：本文件是 `kbs/docs/resource_storage_backend.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

KBS 通过一个由 Rust trait 定义的 `StorageBackend` 抽象来存储机密资源。
`StorageBackend` 接口可以针对不同的存储后端（例如数据库或本地文件系统）分别实现。

[KBS 配置文件](./config.md)
决定 KBS 使用哪个资源后端。默认为本地文件系统（`LocalFs`）。

### 本地文件系统后端

在使用默认的本地文件系统后端实现时，每个资源文件都映射到一个 KBS 资源 URL。
文件路径到 URL 的转换规则定义如下：

| 资源文件路径  | 资源 URL |
| ------------------- | -------------- |
| `file://<$(KBS_REPOSITORY_DIR)>/<repository_name>/<type>/<tag>`  |  `https://<kbs_address>/kbs/v0/resource/<repository_name>/<type>/<tag>`  |

KBS 的根文件系统资源路径同样在 KBS 配置文件中指定，
默认值为 `/opt/confidential-containers/kbs/repository`。

### 阿里云 KMS

[Alibaba Cloud KMS](https://www.alibabacloud.com/en/product/kms?_p_lc=1)（又称 Aliyun KMS）
也可以作为 KBS 的资源存储后端。
在这种模式下，资源会以[通用凭据（generic secrets）](https://www.alibabacloud.com/help/en/kms/user-guide/manage-and-use-generic-secrets?spm=a2c63.p38356.0.0.dc4d24f7s0ZuW7)的形式存放在一个 [KMS 实例](https://www.alibabacloud.com/help/en/kms/user-guide/kms-overview?spm=a2c63.p38356.0.0.4aacf9e6V7IQGW)中。
一个 KBS 可以在其启动配置的 `repository_config` 字段中配置指定的 KMS 实例。配置参见[该文档](./config.md#repository-configuration)。
这些材料可以在 KMS 实例的 [AAP](https://www.alibabacloud.com/help/en/kms/user-guide/manage-aaps?spm=a3c0i.23458820.2359477120.1.4fd96e9bmEFST4) 中找到。
在被访问时，资源 URI `kbs:///repo/type/tag` 会被转换为名为 `tag` 的通用凭据。这意味着 `repo/type` 字段会被忽略。

### AWS Secrets Manager（AWS 密钥管理服务）

[AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
也可以作为 KBS 的资源存储后端。构建 KBS 时需要启用 `aws` 特性开关。
资源以 Secrets Manager 的 secret 形式存储，并通过 `GetSecretValue` 获取。

资源 URI `kbs:///repo/type/tag` 会被转换为 `SecretId = tag` 的 Secrets Manager secret。
`repo/type` 部分会被忽略 —— 与阿里云 KMS 后端的行为一致。`tag` 既可以是 secret 名称，
也可以是完整的 secret ARN。

该后端是只读的。写入与删除会返回错误 —— 请通过 AWS API 来供给和轮换 secret。

### Google Cloud Secret Manager（GCP 密钥管理服务）

[Google Cloud Secret Manager](https://cloud.google.com/secret-manager/docs/overview)
也可以作为 KBS 的资源存储后端。构建 KBS 时需要启用 `gcp` 特性开关。
资源以 Secret Manager 的 secret 形式存储，并通过 `AccessSecretVersion` 获取。

资源 URI `kbs:///repo/type/tag` 会被转换为 Secret Manager 版本
`projects/<project_id>/secrets/<tag>/versions/latest`。
`repo/type` 部分会被忽略，并且会提供最新的已启用版本 ——
与 AWS 和阿里云 KMS 后端的行为一致。

凭据通过
[Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials)（ADC）解析：
即 `GOOGLE_APPLICATION_CREDENTIALS` 环境变量、`gcloud auth
application-default login`，或 GCE/GKE/Cloud Run 元数据服务器（工作负载身份）。

该后端是只读的。写入与删除会返回错误 —— 请通过 GCP API 来供给和轮换 secret。

你可以针对一个真实的项目做冒烟测试：

```bash
gcloud secrets create my-test --replication-policy=automatic
echo -n "hello" | gcloud secrets versions add my-test --data-file=-
# 以 storage_backend_type = "gcp" 以及 project_id 配置 KBS，然后获取
# 资源 kbs:///default/default/my-test
```

### HashiCorp Vault / OpenBao 后端

[Vault KV secrets engine 后端](./vault_kv.md)
