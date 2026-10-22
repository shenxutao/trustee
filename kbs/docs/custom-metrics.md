# KBS Prometheus 指标

> 说明：本文件是 `kbs/docs/metrics.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

密钥代理服务（KBS）在 `/metrics` HTTP 端点上暴露 Prometheus 指标，
该端点与 KBS 本身使用相同的端口（见 KBS 配置文件中 `http_server` 小节的 `sockets` 项，默认为 8080）。


`/metrics` 端点本身被排除在请求指标的采集之外，
以避免监控流量使数据产生偏斜。

## 可用指标

### 构建信息

| 指标 | 类型 | 标签 | 说明 |
|--------|------|--------|-------------|
| `kbs_build_info` | Gauge | `version`, `git_hash`, `build_date` | KBS 二进制的构建信息。取值是占位值（恒为 `1`）。构建信息位于标签取值中。|

### HTTP 服务指标

| 指标 | 类型 | 说明 |
|--------|------|-------------|
| `kbs_http_requests_total` | Counter | 发往 KBS 协议端点的 HTTP 请求总数。 |
| `kbs_http_request_duration_seconds` | Histogram | 请求处理时长的分布，单位为秒。分桶：0.5ms、1ms、5ms、10ms、50ms、500ms、1s。 |
| `kbs_http_request_size_bytes` | Histogram | 请求体大小的分布，单位为字节。指数分桶：32、128、512、2048 与 8192 字节。 |
| `kbs_http_response_size_bytes` | Histogram | 响应体大小的分布，单位为字节。分桶与请求大小相同。 |
| `kbs_http_active_connections` | Gauge | 当前正在处理的 HTTP 连接数。 |

### 资源访问指标

| 指标 | 类型 | 标签 | 说明 |
|--------|------|--------|-------------|
| `kbs_resource_reads_total` | Counter | `resource_path` | 资源读取操作的计数。`resource_path` 标签包含完整的资源路径（例如 `repository/type/tag`）。 |
| `kbs_resource_writes_total` | Counter | `resource_path` | 资源写入操作的计数。`resource_path` 标签包含完整的资源路径。 |

### 策略评估指标

| 指标 | 类型 | 说明 |
|--------|------|-------------|
| `kbs_policy_evaluations_total` | Counter | KBS 策略评估的总次数。 |
| `kbs_policy_approvals_total` | Counter | 被 KBS 策略批准的请求总数。 |
| `kbs_policy_violations_total` | Counter | 被 KBS 策略拒绝的请求总数。 |
| `kbs_policy_errors_total` | Counter | KBS 策略评估过程中出现错误的总次数。 |

### 证明指标

| 指标 | 类型 | 标签 | 说明 |
|--------|------|--------|-------------|
| `kbs_attestation_requests_total` | Counter | - | 证明请求的总数。 |
| `kbs_attestation_successes_total` | Counter | `tee_type` | 按 TEE 类型划分的成功证明总数 |
| `kbs_attestation_failures_total` | Counter | `tee_type` | 失败证明的总数。 |
| `kbs_attestation_errors_total` | Counter | - | 证明处理过程中出现错误的总数（例如畸形请求）。 |

### 认证指标

| 指标 | 类型 | 说明 |
|--------|------|-------------|
| `kbs_auth_requests_total` | Counter | 认证（RCAR 握手）请求的总数。 |
| `kbs_auth_successes_total` | Counter | 成功的认证请求总数。 |
| `kbs_auth_errors_total` | Counter | 认证处理过程中出现错误的总数。 |

## Prometheus 配置示例

要从 KBS 抓取指标，请在你的 Prometheus 配置中添加一个 job：

```yaml
scrape_configs:
  - job_name: 'kbs'
    static_configs:
      - targets: ['kbs-host:8080'] # 替换为与你 KBS HTTP 服务 socket 配置相匹配的 host:port
    scheme: https  # 如果使用 insecure_http 则为 http
    tls_config:
      insecure_skip_verify: true  # 仅在使用自签名证书时使用
```

## Kubernetes 部署

在 Kubernetes 中部署 KBS 时，你可以为 Prometheus Operator 配置一个 `ServiceMonitor`：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kbs
  labels:
    app: kbs
spec:
  selector:
    matchLabels:
      app: kbs
  endpoints:
    - port: 8080 # 替换为与你 KBS HTTP 服务 socket 配置相匹配的端口
      path: /metrics
      interval: 30s
```
