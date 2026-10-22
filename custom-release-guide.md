# KBS 发布指南（KBS Release Guide）

> 说明：本文件是 `release-guide.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](custom-文档清单.md)。

以下步骤是发布 kbs 的指南。请注意，可以使用一个
[辅助脚本](hack/release-helper.sh)
来自动化第 2 步和第 3 步。

1. 确定一个有效的候选发布版本（release candidate）。这一步部分是做健全性检查。
例如，假设你想要最近的那个提交，先检查最后一个被合并的 PR 的测试，
确认所有测试都通过了（或者其中的失败在该上下文中是可接受的）。
类似地，找到由该 PR 生成的 ghcr kbs 镜像。

2. 复制你在第 (1) 步中确定的 ghcr 镜像。新的副本将成为发布版本。
例如，如果你对 ghcr 中的 "latest" 镜像满意，并且假设你要发布 v0.8.2，
那么 ghcr 包的映射关系如下：
```
staged-images/kbs:latest -> key-broker-service:built-in-as-v0.8.2
staged-images/kbs-grpc-as:latest -> key-broker-service:v0.8.2
staged-images/kbs-client:latest -> kbs-client:v0.8.2
staged-images/kbs-ita-as:latest -> key-broker-service:ita-as-v0.8.2
staged-images/rvps:latest -> reference-value-provider-service:v0.8.2
staged-images/coco-as-grpc:latest -> attestation-service:v0.8.2
staged-images/coco-as-restful:latest -> attestation-service:rest-v0.8.2
```

3. 为该发布创建一个 PR。这个 PR 需要更新 kubernetes 的
kustomization.yaml 文件：`kbs/config/kubernetes/base/kustomization.yaml`。
其中的 `newTag` 应当被提升（bump）以反映新版本，例如
`built-in-as-v0.8.2`。

4. 在该 PR 通过所有测试并被合并之后，使用 kbs 的 github 页面
创建发布（release）。发布创建完成后，github 会自动
用新的发布版本在 main 分支上打标签。
