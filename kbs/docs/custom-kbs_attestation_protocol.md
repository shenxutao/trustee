# KBS 证明协议

> 说明：本文件是 `kbs/docs/kbs_attestation_protocol.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../custom-文档清单.md)。

密钥代理服务（KBS）证明协议定义了机密 guest 中的密钥代理客户端（KBC）
与一个受信任的密钥代理服务（KBS）之间的通信。
该协议使用简单、通用且可扩展的
"Request-Challenge-Attestation-Response"（RCAR）方法，
以完成 guest 证明与 secret 注入。

# 简介

KBS 与 KBC 之间证明的目的在于确认 **KBC 所在的平台是否处于预期的安全状态**，
即它是否运行在一个值得信赖的 HW-TEE 中，并且其软件栈是被度量且可验证的。

在本文档中，HW-TEE 证明过程是应用层的语义，
它被定义为一个简单、通用且可扩展的 `RCAR` 协议。
由 HW-TEE 生成的临时非对称密钥用于加密响应输出载荷，
并使用标准的 [HTTP Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
机制来避免多次证明带来的性能问题：KBC 通常需要多个资源，
从而会向 KBS 发起多次请求。如果每次都要走完整的证明流程，
会非常耗时，并给 guest 带来延迟影响。

为了保证 KBS 的易用性与安全完整性，我们将使用
[HTTPS](#http-integration) 作为传输协议来承载本文档中设计的应用层语义。
这是因为 HTTPS 为 KBC 提供了一种认证 KBS 身份的手段，
从而有效避免恶意攻击者劫持 KBS 地址来冒充 KBS 并欺骗 KBC。
为了实现这一点，KBS 的公钥需要以有效的方式传递给 KBC，
而公钥分发的具体方式不在本文档范围内。

需要注意的是，仅靠 HTTPS 提供的机密性保护不足以满足 KBS 协议的安全要求。
因此，如上所述，需要使用 HW-TEE 生成的密钥来加密并保护这些机密数据。

# Request-Challenge-Attestation-Response 语义

KBS 所定义的证明语义是一个简单且可扩展的四步模型，
它使用 JSON 结构来组织信息。如下所示：

1. **Request（请求）**：KBC 向 KBS 发送初始的 `Request` 载荷，
   以便向 KBS 证明自身身份，并最终请求资源。
2. **Challenge（挑战）**：在收到初始 `Request` 载荷后，
   KBS 以 `Challenge` 载荷作为响应。这是 KBS 向 KBC 发送证明挑战的方式。
   除了证明挑战之外，KBS 还会向 KBC 发送一个会话标识，作为 HTTP Cookie。
   在证明成功之后，该会话标识可用于跳过第 2 步和第 3 步。
3. **Attestation（证明）**：KBC 以一份证明证据来回应第 2 步中的证明挑战，
   以证明其环境（HW-TEE）是安全可靠的。KBC 向 KBS 发送一个 `Attestation` 载荷，
   其中包含证明证据以及由 HW-TEE 生成的临时公钥。
4. **Response（响应）**：在证明成功之后，KBS 在对 `/kbs/v0/attest` 端点的响应中
   返回一个 `AttestationToken`。随后，KBC 可以通过向资源相关端点发送 HTTP `GET`
   请求来请求资源。KBS 以 JWE 格式的加密资源数据作为响应。
   在第 2 步中由 KBS 生成的 HTTP Cookie 的有效期内，
   KBC 可以直接向 KBS 请求资源或服务，而无需再走第 2 步和第 3 步。

# 载荷

## `Request`（请求）

<a id="request"></a>

请求的载荷格式如下：

```json
{
  /* KBS protocol version number used by KBC */
  "version": "0.1.1",
  /*
   * Type of HW-TEE platforms where KBC is located,
   * e.g. "tdx" or "snp", etc.
   */
  "tee": "$tee",
  /* Reserved fields to support some special requests sent by HW-TEE. */
  "extra-params": {}
}
```

> **注意：** `/*...*/` 形式的注释在 JSON 中并不合法，不得使用。

- `version`

KBC 所支持的协议版本号。KBS 需要根据该字段判断该 KBC 能否正常通信。

- `tee`

用于声明 KBC 所在 HW-TEE 平台的类型。目前已知的 HW-TEE 平台有：

| TEE            | 说明                                                       |
|----------------|-------------------------------------------------------------------|
| `az-snp-vtpm`  | Microsoft Azure AMD SNP VTPM（Azure 上的 AMD SNP + vTPM）                                      |
| `az-tdx-vtpm`  | Microsoft Azure TDX VTPM（Azure 上的 Intel TDX + vTPM）                                          |
| `snp`          | AMD SNP（AMD SEV-SNP）                                                           |
| `sgx`          | Intel SGX（Intel Software Guard Extensions）                                                         |
| `tdx`          | Intel TDX（Intel Trust Domain Extensions）                                                         |
| `cca`          | Arm Confidential Compute Architecture（Arm 机密计算架构）                             |
| `csv`          | China Secure Virtualization（Hygon CSV）                                       |
| `se`           | IBM Z Secure Execution（IBM Z 安全执行）                                            |
| `tpm`          | Trusted Platform Module（可信平台模块）                                           |
| `sample`       | 仅用于测试证明服务的取值                                          |

最新的列表参见[此处](https://github.com/confidential-containers/kbs-types/blob/main/src/lib.rs)。

- `extra-params`

在运行时证明场景（Intel TDX 与 SGX、AMD SEV-SNP）中，
`extra-params` 字段不被使用，因此被设置为空字符串。
不过，对于某些特殊 HW-TEE 平台的证明，该字段可能被用于传递一些特定信息。
例如，某些证明遵循 Diffie–Hellman 密钥交换协议，
先建立安全通道再传递机密消息（例如 AMD SEV(-ES) 的预证明）。

`extra-params` 也可以携带一个 `attestation-policy-selector`，
用于选择评估本次会话证据的证明服务策略：

```json
"extra-params": { "attestation-policy-selector": "alice" }
```

KBS 会把每个 `attestation-policy-selector` 映射到一个或多个证明策略，
因此可接受的取值是特定于某次部署的，并且 KBC 必须事先知晓。
未映射的 `attestation-policy-selector` 会被拒绝，而省略该字段则会选择一个默认策略。
请注意，并非所有后端证明服务都支持多个策略，例如 CoCo AS 目前只支持一个策略，
而 ITA 支持多个。

选择一个 `attestation-policy-selector` 本身并不会让 KBC 获得任何权利，
因为 KBS 仍然决定每次评估（appraisal）可以释放什么。
不过，它确实改变了所生成令牌的含义，
因此依赖方不应把所有令牌都一视同仁。
[Attestation Results Token](#attestation-results-token) 会指明实际应用的策略，
即 `attestation-policy-selector` 所解析到的那个策略，而不是 `attestation-policy-selector` 本身，
这样资源策略就可以要求在释放资源之前必须具备某次特定的评估。

## `Challenge`（挑战）

<a id="challenge"></a>

如果 KBC 没有持有任何由 KBS 生成的 HTTP Cookie，或者 Cookie 已过期，
那么 KBC 不能直接请求资源，KBS 会对 KBC 的请求返回错误。
KBC 必须首先完成证明流程，并向 KBS 发送一个初始的 [`Request`](#request) 载荷。
随后 KBS 会以如下格式的证明挑战作为响应：

```json
{
  /* Evidence freshness. */
  "nonce": "$nonce",
  /* Extra parameters to support some special HW-TEE attestation. */
  "extra-params": {}
}
```

> **注意：** `/*...*/` 形式的注释在 JSON 中并不合法，不得使用。

- `nonce`

传递给 KBC 的新鲜性数值。KBC 需要把它放入下一步发送给 KBS 的证据中，以防止重放攻击。

- `extra-params`

保留的额外参数字段，用于在某些特定 HW-TEE 需要被证明时传递 KBS 提供的附加信息。

## `Attestation`（证明）

<a id="attestation"></a>

在收到证明挑战后，KBC 会从 HW-TEE 平台构建一份证明证据，并组织成如下载荷：

```json
{
  "init-data": {
    "format": "json",
    "body": "..."
  },
  "runtime-data": {
    "nonce": "$nonce",
    "tee-pubkey": {
      "kty": "EC",
      "crv": "$curve_name",
      "alg": "$key_algorithm",
      "x": "$x_coordinate",
      "y": "$y_coordinate"
    }
  },
  "tee-evidence": {
    "primary_evidence": {},
    "additional_evidence": "{}"
  }
}
```

> **注意：** `init-data` 字段是可选的。`tee-pubkey` 可以是 RSA 或 EC 格式（参见 [Key Format](#key-format) 小节）。`additional_evidence` 是一个 JSON 字符串，表示从 TEE 类型到证据对的映射。

> **注意：** `/*...*/` 形式的注释在 JSON 中并不合法，不得使用。

KBS 通过 KBC 在承载 `Attestation` 载荷的 HTTP 请求中所包含的 HTTP Cookie，
把证明证据与对应的证明挑战匹配起来。

- `init-data`
  Init data 是一个与某 TEE 实例关联的**可选（OPTIONAL）**结构化文档，通常包含被度量的配置设置。

- `init-data.format`
  `init-data.body` 字段中 init data 的格式。目前支持 `json` 与 `toml`。
  序列化逻辑请参考[该文档](./initdata.md)。

- `init-data.body`
  init data 的明文。

- `runtime-data`
  Runtime data 是一个结构化的 JSON 文档，该数据的哈希被绑定到 HW-TEE 证据上，
  从而确保密码学意义上的完整性与真实性。它必须包含：
  - `nonce`：第 2 步中从 KBS 收到的 `Challenge.nonce` nonce 字符串。
  - `tee-pubkey`：由运行在 HW-TEE 中的 KBC 生成的、JWK 格式的公钥。

- `runtime-data.tee-pubkey`

在 KBC 收到证明挑战之后，会在 HW-TEE 中生成一个临时的非对称密钥对。
私钥保存在 HW-TEE 中。公钥及其描述信息会被导出并放入 `tee-pubkey` 字段，
与证明证据一起发送给 KBS。
`tee-pubkey` 字段的哈希必须被包含在 HW-TEE 证据的自定义字段中，并由 HW-TEE 硬件签名。
该公钥在下一次 KBC 收到来自 KBS 的证明挑战之前一直有效。

`tee-pubkey` 遵循 [JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517) 格式，
可以是 RSA 或椭圆曲线（EC）格式，如 [key format](#key-format) 小节所述。

- `tee-evidence`

证明证据由 KBC 执行环境中的 HW-TEE 平台软硬件生成。
`tee-evidence` 是一个复合证据结构，包含：
  - `primary_evidence`：主要的 TEE 证据。其格式取决于具体 TEE，通常由证明服务定义。
  - `additional_evidence`：客户端内次级 TEE 设备的附加证据。这是一个 JSON 字符串，表示从 TEE 类型到（TEE 类别, TEE 证据）对的映射，以字符串形式表示以避免序列化不一致。

复杂的 guest 可能由不止一种证明方以及不止一种格式的 TEE 证据来表示。
来自多个证明方的 TEE 证据可以被组合在一起。
组合与拆解证据的方法应当在证明代理与证明服务或 KBS 之间共享。

通常，KBS 不会解析或分析证明证据，
而是把它转发给证明服务进行校验。
在处理组合证据时，KBS 可能负责拆开多份硬件证据，
并把它们分别转发给证明服务。
或者，也可以由证明服务负责拆包组合证据。
无论哪种方式，证据的格式都不在本文档范围内。

## `Response`（响应）

KBS 协议定义了两类响应：

1. **Attestation Results Token（证明结果令牌）**：在证明成功时由 `/kbs/v0/attest` 端点返回。
2. **Resource Response（资源响应）**：在请求资源时返回，使用 JWE 格式加密。

### Attestation Results Token（证明结果令牌）

<a id="attestation-results-token"></a>

在证明成功之后，`/kbs/v0/attest` 端点会在响应体中返回一个 `AttestationToken`：

```json
{
  "token": "<JWT_token_string>"
}
```

该令牌是一个序列化的 JSON Web Token（JWT），其中包含证明结果与背书。
关于该令牌的详细信息，参见 [Attestation Token](../../attestation-service/docs/attestation_token.md)
以及 [simple token claim in JSON](../../attestation-service/docs/example.token.json)。

### Resource Response（资源响应）

<a id="resource-response"></a>

在证明成功之后，KBC 可以通过向 KBS 发送 HTTP `GET` 请求来请求资源。

如果 KBS 批准了该请求，它会向 KBC 返回一个资源响应载荷，
该载荷遵循 [JSON Web Encryption](https://www.rfc-editor.org/rfc/rfc7516) 的
[flattened serialization format](https://www.rfc-editor.org/rfc/rfc7516#section-7.2.2)：

``` json
{
  "protected": "$jose_header",
  "encrypted_key": "$encrypted_key",
  "aad": "$aad",
  "iv": "$iv",
  "ciphertext": "$ciphertext",
  "tag": "$tag"
}

```

上述 JWE JSON 字段定义如下：

``` rust
let jose_header_string = format!(r#"{{"alg": "{}","enc": "{}"}}"#, alg, enc);
let jose_header = base64_url::encode(&jose_header_string);
let encrypted_key = base64_url::encode(enc_kbs_symkey);
let aad = base64_url::encode(additional_authenticated_data);
let iv = base64_url::encode(initialization_vector);
let ciphertext = base64_url::encode(response_output);

// tag is optional and depends on the encryption algorithm.
tag = base64_url::encode(authentication_tag);

```

- `protected.alg`

用于加密 `encrypted_key` 中加密密钥的算法。
由于该密钥是使用 HW-TEE 公钥加密的，`alg` 必须与
[`Attestation`](#attestation) 的 `tee-pubkey` 字段中所描述的值相同。

- `protected.enc`

用于加密 KBS 服务 API 输出的加密算法。

- `ciphertext`

KBS 服务 API 的输出。它必须使用 KBS 生成的临时密钥加密。

- `aad`（如果使用 AEAD 则为必需）

AEAD 操作的一个输入，它受到完整性保护但不被加密。
依据 [JSON Web Encryption](https://www.rfc-editor.org/rfc/rfc7516)，
AAD 字段应当由 `ASCII(BASE64URL(UTF8(JWE Protected Header)))` 计算得到。

- `iv`

密码学原语的输入，用于提供初始状态。
如果 `enc` 所描述的算法不需要它，该字段留空。

- `encrypted_key`

被加密的对称密钥，用于加密 `ciphertext`。
该密钥使用 HW-TEE 的公钥、按 `alg` 中定义的算法加密。

- `tag`

用于认证密文的认证标签。如果 `enc` 所描述的算法不需要它，该字段留空。

## Key Format（密钥格式）

<a id="key-format"></a>

### Public Key（公钥）

[`Attestation`](#attestation) 的 `runtime-data.tee-pubkey` 字段与
[Token Resource](#attestation-results-token) 的 `tee-pubkey` 声明项
都包含一个由 HW-TEE 生成的非对称密钥对中的公钥。

该字段遵循 [JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517) 格式，
并同时支持 RSA 与椭圆曲线（EC）两种密钥类型。

#### RSA Key Format（RSA 密钥格式）

``` json
{
  "kty": "RSA",
  "alg": "$key_algorithm",
  "n": "$pubkey_modulus",
  "e": "$pubkey_exponent"
}
```

#### Elliptic Curve Key Format（椭圆曲线密钥格式）

``` json
{
  "kty": "EC",
  "crv": "$curve_name",
  "alg": "$key_algorithm",
  "x": "$x_coordinate",
  "y": "$y_coordinate"
}
```

`kty` 字段决定使用哪种格式。对于 RSA 密钥，`n` 与 `e` 分别表示模数与指数。
对于 EC 密钥，`crv` 指定曲线名称（例如 "P-256"、"P-384"），
而 `x` 与 `y` 是公钥点的坐标。

# HTTP Integration（HTTP 集成）

<a id="http-integration"></a>

KBS 使用 HTTPS 传输协议，以上述证明消息作为 HTTP 请求与响应的载荷进行交换。

这里描述的基于 HTTP 的 KBS 证明协议的第一步，
允许一个证明方向一个受信任的密钥代理服务证明自身身份。
第二步是证明方向 KBS 请求一个或多个受保护的资源。
这些资源由 KBS 依据绑定到 HTTP Cookie 的认证结果返回给证明方。

## Authentication（认证）

<a id="authentication"></a>

在能够请求一个受保护的资源（密钥、令牌等）之前，
证明方必须先向远端的 KBS 证明自身身份。
只有在成功完成身份认证之后，它才能请求这些资源。

KBS 通过两个端点提供该认证服务：

1. `/kbs/v0/auth` 只接受 `POST` 请求，其请求体是一个 [KBS Request](#request)
   JSON 载荷。该端点用于证明方发起证明协议并向 KBS 证明自身身份。
   KBS 的 HTTP 响应由以下内容组成：
    1. 一个 `Set-Cookie` 头部，取值为 `kbs-session-id=<session>`。KBS 通过该 Cookie 跟踪证明方及其证明结果。
    2. 一个供证明方接受的证明挑战。它是响应的内容，取值为一个 [KBS Challenge](#challenge) JSON 载荷。
    3. 在出现错误的情况下（例如 KBS 基于 `version` 兼容性拒绝了 [KBS Request](#request)），返回一个 HTTP 状态码为 401（`Unauthorized`）的响应，并附带 ErrorInformation JSON 载荷。

2. `/kbs/v0/attest` 只接受 `POST` 请求，其请求体是一个 [KBS Attestation](#attestation)
   JSON 载荷，并且头部包含一个 `Cookie`，其取值为第 1.i 步中收到的值。
   这是证明方回应第 1.ii 步中收到的证明挑战的方式。如果证明成功，
   KBS 会以 [Attestation Results Token](#attestation-results-token) 回应该请求。

```
┌──────────┐                                          ┌─────┐
│ Attester │                                          │ KBS │
└────┳─────┘                                          └──┳──┘
     ┃                 ┌─────────────────┐               ┃
     ┃─────────────────│POST /kbs/v0/auth│──────────────▶┃
     ┃                ┌┴─────────────────┴┐              ┃
     ┃                │Header: No Cookie  │              ┃
     ┃                │Body: Request      │              ┃
     ┃                └───────────────────┘              ┃
     ┃                     ┌────────┐                    ┃
     ┃◀────────────────────│Response│────────────────────┃
     ┃      ┌──────────────┴────────┴───────────────┐    ┃
     ┃      │Header: "Set-Cookie kbs-session-id=123"│    ┃
     ┃      │Body: Challenge                        │    ┃
     ┃      └───────────────────────────────────────┘    ┃
     ┃                                                   ┃
     ┃                                                   ┃
     ┃                                                   ┃
     ┃                                                   ┃
     ┃                                                   ┃
     ┃                ┌───────────────────┐              ┃
     ┃────────────────│POST /kbs/v0/attest│─────────────▶┃
     ┃        ┌───────┴───────────────────┴───────┐      ┃
     ┃        │Header: "Cookie kbs-session-id=123"│      ┃
     ┃        │Body: Attestation                  │      ┃
     ┃        └───────────────────────────────────┘      ┃
     ┃                     ┌────────┐                    ┃
     ┃◀────────────────────│Response│────────────────────┃
     ┃      ┌──────────────┴────────┴───────────────┐    ┃
     ┃      │Header: "Set-Cookie kbs-session-id=123"│    ┃
     ┃      │Body: Attestation Results Token        │    ┃
     ┃      └───────────────────────────────────────┘    ┃
     ┃                                                   ┃
     ┃                                                   ┃
     ┃                                                   ┃
     ▽                                                   ▽
               Figure 1: KBS Authentication Phase

```

## Resource Requests（资源请求）

[认证](#authentication)阶段是整个 KBS 协议的第一步，
在这一步中证明方向某个 KBS 实现证明自身身份。
这引出了第二个阶段，在该阶段中已通过认证的证明方可以向 KBS 请求受保护的资源。

证明方可以通过向 KBS 提供 HTTP cookie，或者提供作为证明 HTTP 响应一部分返回的
Attestation Results token，来访问受保护的资源。
前者通常生命周期较短，一般用于一次性的、同步的资源请求。
对于更长的 KBS 会话，使用后者更为方便。

令牌存放在证明会话中（而不是在 `Authorization` 头部中重新发送），
但 KBS 在使用前仍会校验该 JWT。参见
[Attestation Token Verification](./attestation_token_verification.md)
以了解信任链配置以及针对各个证明服务的指引。

### HTTP Cookie Authentication（HTTP Cookie 认证）

KBS 实现会跟踪证明结果，并把它们绑定到一个 cookie 标识上。
在第二个阶段，KBS 可以通过把证明方在资源请求消息中包含的 cookie 标识
映射到其证明结果，来决定某个特定资源是否可以释放给该证明方。

为了从 KBS 请求一个受保护的资源，证明方向一个资源相关端点发送 `GET` 请求。
如果证明方被允许访问该资源，KBS 会以一个 HTTP 响应回应这个 `GET` 请求，
其内容为 [Resource Response](#resource-response) JSON 载荷（JWE 格式）：

```
┌──────────┐                                         ┌─────┐
│ Attester │                                         │ KBS │
└───┳──────┘                                         └──┳──┘
    ┃            ┌─────────────────────────┐            ┃
    ┃────────────│GET /kbs/v0/<resource_id>│───────────▶┃
    ┃      ┌─────┴─────────────────────────┴────┐       ┃
    ┃      │Header: "Cookie kbs-session-id=123" │       ┃
    ┃      │Body: N/A                           │       ┃
    ┃      └────────────────────────────────────┘       ┃
    ┃                    ┌────────┐                     ┃
    ┃◀───────────────────│Response│─────────────────────┃
    ┃     ┌──────────────└────────┘───────────────┐     ┃
    ┃     │Header: "Set-Cookie kbs-session-id=123"│     ┃
    ┃     │Body: Response                         │     ┃
    ┃     └───────────────────────────────────────┘     ┃
    ┃                                                   ┃
    ┃                                                   ┃
    ┃                                                   ┃
    ▽   Figure 2: KBS Resource Request Phase - Session  ▽
```

对受保护资源的请求可能由于三种原因失败：

1. 请求方未通过认证，因此无法提供正确的 cookie 标识。KBS 实现会返回一个 HTTP 状态码为 401（`Unauthorized`）的响应。
2. 证明方已通过认证，但请求了一个它无权获得的资源。KBS 实现会返回一个 HTTP 状态码为 403（`Forbidden`）的响应。
3. 所请求的资源不存在。KBS 实现会返回一个 HTTP 状态码为 404（`Not Found`）的响应。

### Attestation Results Token Authentication（证明结果令牌认证）

作为认证过程的一部分，请求方会生成一个 TEE 非对称密钥对。
在证明成功之后，KBS 会发放一个令牌，其中包含针对该 TEE 公钥的证明结果与背书。

为了从 KBS 请求一个受保护的资源，证明方向一个资源相关端点发送 `GET` 请求，
并把该令牌作为 bearer 授权。如果该令牌通过校验，
KBS 会以一个 HTTP 响应回应这个 `GET` 请求，
其内容为 [Resource Response](#resource-response) JSON 载荷（JWE 格式）。
[Resource Response](#resource-response) JSON 载荷中的对称密钥
将由该令牌中的公钥封装（wrap）。

Bearer 令牌校验所用的 KBS 信任锚在 KBS 配置文件的
`[attestation_token]` 小节中配置。参见
[Attestation Token Verification](./attestation_token_verification.md)。

```
┌──────────┐                                         ┌─────┐
│ Attester │                                         │ KBS │
└───┳──────┘                                         └──┳──┘
    ┃            ┌─────────────────────────┐            ┃
    ┃────────────│GET /kbs/v0/<resource_id>│───────────▶┃
    ┃      ┌─────┴─────────────────────────┴────┐       ┃
    ┃      │Header: Authorization Bearer        │       ┃
    ┃      │Body: N/A                           │       ┃
    ┃      └────────────────────────────────────┘       ┃
    ┃                    ┌────────┐                     ┃
    ┃◀───────────────────│Response│─────────────────────┃
    ┃     ┌──────────────└────────┘───────────────┐     ┃
    ┃     │Header: No Cookie                      │     ┃
    ┃     │Body: Response                         │     ┃
    ┃     └───────────────────────────────────────┘     ┃
    ┃                                                   ┃
    ┃                                                   ┃
    ┃                                                   ┃
    ▽    Figure 3: KBS Resource Request Phase - Token   ▽
```

对受保护资源的请求可能由于三种原因失败：

1. 请求方未通过认证，因此无法提供有效的令牌。KBS 实现会返回一个 HTTP 状态码为 401（`Unauthorized`）的响应。
2. 证明方已通过认证，但请求了一个它无权获得的资源。KBS 实现会返回一个 HTTP 状态码为 403（`Forbidden`）的响应。
3. 所请求的资源不存在。KBS 实现会返回一个 HTTP 状态码为 404（`Not Found`）的响应。

### Secret Resource（机密资源）

KBS 使用如下路径格式来定位 secret 资源：

```
/kbs/v0/resource/<repository>/<type>/<tag>
```

其中 URL 路径参数为：

- `<repository>`：这与容器镜像仓库（`docker.io/{repository}/{image_name}:{tag}`）的概念类似，
  用于方便用户管理不同的资源组。其名称应当完全由用户设定。
- `<type>`：用于区分不同的资源类型。
- `<tag>`：用于区分不同的资源实例。

是否成功回应证明方针对某个具体资源实例的资源请求，
取决于 KBS 及其底层的证明服务。
该决定通常同时基于证明证据、结果以及为给定证明方配置的策略。

#### Resource Registration（资源注册，实验性）

向 `/kbs/v0/resource/<repository>/<type>/<tag>` 发送一个携带资源内容的 POST 请求，
可以把该资源注册进 KBS。

### Attestation Results Token（证明结果令牌）

已通过认证的证明方还可以在 `/kbs/v0/attest` 的响应体中从 KBS 收到一个证明令牌。
证明方可以使用该证明结果令牌向外部服务（又称依赖方）请求额外的资源。

所提供的证明结果令牌遵循 [JSON web token](https://datatracker.ietf.org/doc/html/rfc7519)
标准与格式。

##### Header（头部）

这是一个 [JOSE](https://jose.readthedocs.io/en/latest/) 头部的 Base64url 编码。

它是 JSON 格式的，同时包含令牌标准格式（`typ`）与令牌签名算法（`alg`）。
例如：

```json
{
  "typ": "JWT",
  "alg": "RS256"
}
```

##### Payload（载荷）

这是一组 JSON 格式声明项的 Base64url 编码，如下例所示：

``` json
{
  "exp": 1568187398,
  "iat": 1568158598,
  "iss": "https://xxxxxx",
  "jwk": $token-pubkey,

  "tee-pubkey": $pubkey,
  "tcb-status": $claims,
  "evaluation-report": $report,
}
```

令牌载荷被划分为注册声明项集合与私有声明项集合。
KBS 也可以包含额外的公共声明项集合。

注册声明项集合必须包含 `exp`、`iat`'`iss` 和 `jwk` 名称，
它们分别声明该令牌的过期时间、签发时间与签发者（KBS URL 地址）：

| 字段 | 说明                          | 类型                    | 示例              |
|-------|--------------------------------------|-------------------------|----------------------|
| `exp` | 过期时间                      | 自纪元起的秒数 | `99991231235959`     |
| `iat` | 签发时间                         | 自纪元起的秒数 | `180322235959`       |
| `iss` | 签发者                               | KBS URL                 | `https://my-kbs.io/` |
| `jwk` | 用于校验令牌签名的公钥 | RSA Json Web Key        |                      |

自定义声明项集合必须包含来自 `Attestation Service` 的证明结果，
其中包括 TCB 状态与度量值。

自定义声明项集合还可以包含一个 `tee-pubkey` 声明项。
该声明项指代 KBC 随证明证据一起发送的 HW-TEE 公钥，
它在该令牌的有效期内有效。`tee-pubkey` 遵循
[JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517) 格式，
可以是 RSA 或椭圆曲线（EC）格式，如 [Key Format](#key-format) 小节所述。

`evaluation-report` 是 `Attestation Service` 策略引擎的输出，
即 AS 策略对 TEE 证据的评估意见。

当 KBC 使用该令牌向依赖方服务 API 请求资源或服务时，
用于加密输出载荷的对称密钥可以由所提供的 `tee-pubkey` 封装。

### Set Attestation Policy（设置证明策略）

KBS 的用户可以通过以下端点设置一个证明校验策略：

```
/kbs/v0/attestation-policy
```

POST 请求的载荷形如：

```json
{
  "type": "rego",
  "policy_id": "default",
  "policy": "<Base64EncodedPolicy>"
}
```

其中 `type` 是策略格式（例如 `rego` 或 `opa`），`policy_id` 提供策略 ID，
而 `policy` 是 base64 编码的策略内容。
只有通过认证的用户才能向该端点发送 POST 请求。
KBS 通过请求中必须包含的、由用户私钥签名的 JSON Web Token（JWT）来校验用户身份。

### Set Resource Policy（设置资源策略）

KBS 的用户可以通过以下端点设置一个资源策略：

```
/kbs/v0/resource-policy
```

POST 请求的载荷形如：

```json
{
  "policy": "<Base64EncodedPolicy>"
}
```

其中 `policy` 是 base64 编码的策略内容。
只有通过认证的用户才能向该端点发送 POST 请求。
KBS 通过请求中必须包含的、由用户私钥签名的 JSON Web Token（JWT）来校验用户身份。

##### Signature（签名）

使用令牌头部中所描述的算法，KBS 对令牌头部与令牌载荷的拼接结果
（以点号 `.` 分隔）进行签名。令牌签名就是该签名的 Base64url 编码字符串。

#### HTTP Response（HTTP 响应）

该令牌包含在 `/kbs/v0/attest` 的 `POST` HTTP 响应体中：

```json
{
  "token": <token>
}
```

一个序列化后的令牌按如下方式构造：

``` rust
let jwt_header = base64_url::encode(r#"{{"alg": "RS256","typ": "JWT"}}"#);

let jwt_claims_string = format!(r#"{{"exp": {}, "iat": {}, "iss": {}, {}}}}"#, exp, iat, iss, claims);
let jwt_claims = base64_url::encode(&jwt_claims_string);

let jws_signature_input = format!("{}.{}", jwt_header, jwt_claims);
let jws_signature = base64_url::encode(rs256_key_pair.sign(&jws_signature_input));

let serialized_token = format!("{}.{}.{}", jwt_header, jwt_claims, jwt_signature);
```

## Error information（错误信息）

除了使用标准 HTTPS 状态码来表示返回的错误类型之外，
还需要提供对证明错误的详细描述。
为此使用了 [Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807) 格式：

```json
{
  "type": "https://github.com/confidential-containers/kbs/errors/<problem-type>",
  "detail": "$detail"
}
```
> **注意：** `type` 字段是一个标识字符串，而不是一个可用的
> 超链接。该 URL 不可解析；它的存在是为了给错误类型提供命名空间，
> 并遵循 RFC 7807 约定。

`<problem-type>` 是以下取值之一，定义于
[`kbs/src/error.rs`](https://github.com/confidential-containers/trustee/blob/main/kbs/src/error.rs)：

| 错误类型                      | HTTP 状态码 | 说明                                                          |
| -------------------------------- | ----------- | ---------------------------------------------------------------------|
| `AdminAuth`                     | 401         | 管理类 API 端点的管理员认证失败。      |
| `AttestationError`              | 401         | 证明过程中发生错误。                    |
| `HTTPFailed`                    | 401         | HTTP 服务初始化失败。                                |
| `HTTPSFailed`                   | 401         | HTTPS 服务初始化失败。                                |
| `InvalidRequestPath`            | 404         | 请求路径不对应一个有效资源。            |
| `JweError`                      | 401         | 构造 JWE 加密的资源响应失败。                 |
| `PluginManagerInitialization`   | 401         | 插件管理器初始化失败。                             |
| `PluginNotFound`                | 404         | 未找到所请求的插件。                                  |
| `PluginInternalError`           | 401         | 插件内部发生错误。                          |
| `PolicyDeny`                    | 403         | 被资源策略拒绝访问，通常是由于证明结果所致。  |
| `PolicyEngine`                  | 401         | 策略引擎中发生错误。                              |
| `ResourceNotFound`              | 404         | 某个插件报告所请求的资源不存在。        |
| `SerdeError`                    | 401         | JSON 载荷的序列化或反序列化失败。                   |
| `TokenNotFound`                 | 401         | 请求中未找到证明令牌。                       |
| `TokenVerifierError`            | 401         | 所提供的证明令牌校验失败。                  |

## OpenAPI Description（OpenAPI 描述）

KBS 的 HTTP 端点与载荷
[正式描述](./kbs.yaml)在
一个符合 [OpenAPI](https://www.openapis.org/) 规范的格式中。

# 致谢

以下个人对本协议的开发起到了重要作用：

* Ding Ma (@Xynnn007)
* James O. D. Hunt (@jodh-intel)
* Jia Zhang (@jiazhang0)
* Jiale Zhang (@jialez0)
* Samuel Ortiz (@sameo)
* Sergio Lopez (@slp)
* Thomas Fossati (@thomas-fossati)
* Tobin Feldman-Fitzthum (@fitzthum)
