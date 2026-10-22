# 速率限制

> 说明：本文件是 `kbs/docs/rate-limiting.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

`POST /kbs/v0/auth` 在设计上是不需要认证的，并且每次调用都会分配一条会话记录，
因此一个滥用的客户端可以塞满会话存储。KBS 自身并不对请求做速率限制。

针对单个客户端的限制应当放在 KBS 前置的反向代理、Ingress 控制器或负载均衡器上：

- 在代理之后，KBS 只能看到代理的地址，无法区分不同客户端。
- 代理可以看到真实的客户端地址，并且其计数器不依赖于由哪个 KBS 副本来处理请求。

这适用于所有部署形态，无论 KBS 是由 Helm chart 运行、
由 trustee-operator 运行，还是直接运行。请按所选代理文档说明的方式配置该限制。
对于 Helm chart，参见
[按客户端速率限制](../../deployment/helm-chart/README.md#per-client-rate-limiting)。
