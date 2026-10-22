## 开发容器（Development containers）

> 说明：本文件是 `DEVELOPMENT.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](custom-文档清单.md)。

Trustee 的开发可以通过开箱即用的开发容器来完成，这些容器由 [.devcontainer](.devcontainer) 目录提供，可让你完成以下任务：

| 任务（task）   | 路径（path） |
| -------- | ------- |
| **simple**，编辑纯文本文件  | `.devcontainer/devcontainer.json`    |
| **image**，构建和开发容器镜像、更新 devcontainer | `.devcontainer/image-builder/devcontainer.json`     |
| **code**，编写、编译和测试代码    | `.devcontainer/rust-development/devcontainer.json`    |

如果你刚接触开发容器，更多信息参见 [containers.dev](https://containers.dev)。

想快速确认你常用的 IDE 是否支持开发容器，可查看[这里](https://containers.dev/supporting)或你所用 IDE 的参考文档。


### 维护 devcontainer

#### 镜像（Images）

开发容器镜像已被固定（pin）到多架构 digest。若要更新，先启动 [simple](.devcontainer/devcontainer.json) 开发容器，添加 skopeo：

```shell
sudo apk add skopeo
```

然后为目标镜像提取多架构 digest。请注意，对于开发容器镜像，子标签（child tag）必须被移除。生成后将其加入对应的 devcontainer.json。

```shell
#!/bin/bash

# This script retrieves the manifest digest for quay.io/podman/stable:v5
# and prints the pinned multi-architecture image reference.
IMAGE="quay.io/podman/stable:v5"
# Retrieve the manifest digest using skopeo and jq
DIGEST=$(skopeo inspect docker://$IMAGE | jq -r '.Digest')
# Print the pinned image reference
echo "Pinned multi-arch image reference:"
echo "${IMAGE%@*}@${DIGEST}"
```

#### 特性（Features）

开发容器特性通过每个容器一份的[锁文件](https://github.com/devcontainers/spec/blob/main/docs/specs/devcontainer-lockfile.md)进行版本固定。如果想要更改某个开发特性集合，启动 [image](.devcontainer/image-builder/devcontainer.json) 开发容器，应用改动后运行：

```shell
devcontainer build --workspace-folder --docker-path podman --config .devcontainer/image-builder/devcontainer.json
```

#### Dependabot（依赖自动更新）

开发容器规格说明已加入 dependabot 生态体系，更多信息见[这里](https://containers.dev/guide/dependabot)。
