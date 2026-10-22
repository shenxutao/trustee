# 更新 Hadolint（Updating Hadolint）

> 说明：本文件是 `.devcontainer/image-builder/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

如需更新，请按以下示例下载所需版本的校验和（checksum）。

```console
hadolint_version=v2.14.0
curl -L https://github.com/hadolint/hadolint/releases/download/${hadolint_version}/hadolint-linux-x86_64.sha256 -o hadolint-linux-x86_64_${hadolint_version}.sha256
```

然后相应地更新 postCreateCommand.sh。
