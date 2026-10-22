# SPIRE 委派身份插件

> 说明：本文件是 `kbs/docs/plugins/spire.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../custom-文档清单.md)。

本插件允许 Trustee 与现有的 SPIRE agent/server 协作，向机密工作负载提供 SVID。
具体而言，本插件使用 SPIRE Agent 的委派身份 API（Delegated Identity API），该 API 接收一组
selector 并返回一个 SVID。
本插件从证明令牌（attestation token）中的 Validated Identifiers 字段里解析出这些 selector，
并将 SVID 作为加密资源返回给虚拟机。
该插件同时也提供信任包（trust bundle）。

从较高层面看，机密 SPIRE 的目标是提供一种有密码学保证支撑的 SVID。
换言之，SPIFFE-ID 所代表的属性，必须反映飞地（enclave）中真正在运行的工作负载，
并且应当存在对此的密码学证据。

这些保证是通过一种名为 Validated Identifiers（已校验标识符）的抽象来实现的。
Validated Identifiers 是 Trustee 证明令牌的扩展，其中包含 guest 的可证明属性。
Validated Identifiers 由证明策略（attestation policy）从硬件证据和 Init-Data 明文中提取出来。
这些标识符的有效性取决于多个组件。

在 Confidential Containers 中，大多数 validated identifier 都根植于 Kata Agent 策略，
因为它定义了哪些工作负载可以在 podvm 内运行。
首先，Kata Agent 策略在启动之前生成。
它定义了这个 pod 的样貌。
Kata Agent 策略被加入 Init-Data，并在启动时被度量。
这就把一个 podvm 绑定到了某个 pod 定义上。
当某个工作负载向 Trustee 请求 SVID 时，证明策略会判定 TCB 状态，并从 Init-Data 中提取出 Validated Identifiers。
本插件随后把这些标识符格式化为 selector，并传递给 SPIRE Agent。

请注意，本插件并不会确认 TCB 状态。应当使用 KBS 策略来检查 TCB 状态是否为肯定（affirming）结论。
如果 guest 的整体 TCB 状态没有被检查，
就无法判断该 podvm 是否遵守了 Kata Agent 策略。
这样一来，Validated Identifiers 也就不可信了。

你也可以把本插件用于其他运行时，只要相应调整证明策略，使其提取出必要的 Validated Identifiers 即可。
请确保该运行时本身确实遵循这些标识符。

本插件以 podvm 的粒度收集 selector。
用 Kubernetes 的术语来说，这意味着它只能生成 Pod 级别的 selector。

通过配置，本插件可以生成与 SPIRE k8s workload attestor 所生成的 selector 外观一致的 selector，
这样工作负载就可以使用既有的 workload entry，透明地运行在 CoCo 内部。
另一方面，本插件也可以显式标注这些 selector 是由 Trustee 生成的。参见下面的配置项。

## 配置

|----------------------------|--------|------------------------------------------------------------------------------------|-----------|
| `admin_socket_path`       | String | 指向 SPIRE Agent 管理套接字（委派身份 API）Unix socket 的文件系统路径    | -         |
| `attested_selector_value` | String | 始终会被添加的标记型 selector `trustee:<value>` 所使用的值                  | `true`    |
| `k8s_selector_prefix`     | String | 已知的 k8s workload attestor selector 键所使用的 selector 类型                   | `k8s`     |
| `trustee_selector_prefix` | String | 标记型 selector 以及任何其他标识符键所使用的 selector 类型            | `trustee` |

请注意，`admin_socket_path` 应当指向 SPIRE Agent 的管理 UDS。这与 workload API 的 UDS 不是同一个东西，通常需要通过 SPIRE Agent 配置文件显式启用。

本插件把 selector 分为两组：与 k8s workload attestor 匹配的那些，以及不匹配的那些。
这两组的前缀/类型可以分别独立设置。
默认情况下，与 k8s workload attestor 可能创建的 selector 相匹配的那些 selector，会使用与 SPIRE 相同的类型。
其他 selector 则使用前缀 `trustee`。
本插件支持嵌套层级最多为一层的 validated identifier（例如某个标识符带有一个字符串数组）。
此外，总会额外添加一个 selector，用于指明这些 selector 是由 Trustee 生成的。
该 selector 的值是可配置的，允许你发出一个专属于自己 Trustee 实例的 selector。

## 安装配置

为了连接到 SPIRE Agent 的管理套接字，本插件自身需要一个 SVID。
因此，必须存在一个可供 KBS 进行远程证明的 SPIRE Agent。
根据 Trustee 的部署方式，你可以为此选择使用 k8s workload attestor 或者 unix attestor。
无论哪种情况，你都需要创建一个反映 Trustee 身份的 workload entry。
你还需要在 SPIRE Agent 配置文件中，把这个 workload entry 的 SPIFFE-ID
指定为 `authorized_delegate`。

请注意，SPIRE Agent 的管理套接字会自动解析到 SPIFFE-ID 的连接。
该插件无需再单独去访问 workload API。

## 使用

插件配置完成后，向 SPIRE 插件请求一个 `x509-svid` 资源。
该资源通过类似 `kbs+spire:///x509-svid` 这样的资源 URI 来标识。
