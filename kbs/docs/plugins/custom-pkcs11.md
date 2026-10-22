# PKCS11 插件

> 说明：本文件是 `kbs/docs/plugins/pkcs11.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../custom-文档清单.md)。

PKCS11 是一份内容广泛的规范，支持多种密码学操作，主要适用于硬件安全模块（Hardware Security Module，HSM）。Trustee 实现了其中一部分特性，可供已通过远程证明的客户端使用。

## 资源存储

PKCS11 插件可以把明文资源存储到 HSM 中。由于 KBS 期望资源是明文的，我们把这些资源以通用密钥（generic secret）类型的私钥形式存放在 HSM 中。当用户向 KBS 上传资源时，该存储接口会以预期的方式把资源配置到 HSM 中。用户只需指定一个已初始化的 HSM 插槽（slot）的位置即可。用户可以使用与资源后端相同的语义来获取资源。

`GET https://<kbs_address>/kbs/v0/pkcs11/resource/<repository_name>/<type>/<tag>`

管理员也可以使用与资源后端类似的语义把资源存入 HSM。

`POST https://<kbs_address>/kbs/v0/pkcs11/resource/<repository_name>/<type>/<tag>`

## 密钥封装/解封（Key Wrapping/Unwrapping）

插件初始化时，会生成一对唯一的公钥/私钥，用于配合 HSM 对数据进行封装和解封。

### 密钥封装

为了封装数据，客户端可以发送 POST 请求到：

`POST https://<kbs_address>/kbs/v0/pkcs11/wrap-key`

并在请求体中给出明文数据。PKCS11 插件会用公钥对数据进行封装，然后把结果返回给客户端。

密钥封装需要管理员校验，因为只有管理员才应当能够为远程证明客户端封装数据。

### 密钥解封

为了解封数据，客户端可以发送 GET 请求到：

`GET https://<kbs_address>/kbs/v0/pkcs11/wrap-key`

并在请求体中给出被封装的数据。PKCS11 插件会用私钥对数据进行解封，然后把结果返回给客户端。
