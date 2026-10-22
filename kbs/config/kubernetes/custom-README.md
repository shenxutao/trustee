# 在 Kubernetes 上部署密钥代理服务（Key Broker Service）

> 说明：本文件是 `kbs/config/kubernetes/README.md` 的简体中文译文，原文保持英文不变。总索引见仓库根目录 [custom-文档清单.md](../../../custom-文档清单.md)。

本文将介绍如何在一个 Kubernetes 集群上部署 KBS（内置 Attestation Service）。

> :warning: **请注意，下面的清单与说明并未涵盖 KBS 部署中的全部有状态资源。** 如果 Pod 被重启或被重新调度，对部署所做的变更可能会丢失；并且服务副本（replication）可能导致行为不一致。对于生产环境部署，请考虑使用持久化卷（persistent volume）。

## 密钥

创建一份你希望通过这个 KBS 实例对外提供的密钥：

```bash
echo "This is my super secret" > overlays/key.bin
```

或者，如果部署在 IBM Secure Execution 上，则运行：

```bash
echo "This is my super secret" > overlays/ibm-se/key.bin
```

如果你有不止一份密钥，请把它们复制到 `config/kubernetes/overlays` 目录，并按下面所示将这些文件加入到 `overlays/kustomization.yaml` 中：

```yaml
...
- name: keys
  files:
  - key.bin
  - secret.key
  - password.txt
  ...
```

## 定义 KBS 仓库（repository）

在默认配置下，密钥会被存放在 `reponame/workload_key/` 目录中。如果你希望定义一个不同的仓库，请对 `overlays/patch.yaml` 文件做相应的修改。

## 可选：使用 Ingress 暴露 KBS

如果你希望通过 Ingress 暴露 KBS，请运行以下命令：

```bash
export KBS_INGRESS_CLASS="REPLACE_ME"
export CLUSTER_SPECIFIC_DNS_ZONE="REPLACE_ME"
export KBS_INGRESS_HOST="kbs.${CLUSTER_SPECIFIC_DNS_ZONE}"

pushd overlays
envsubst <ingress.yaml >ingress.yaml.tmp && mv ingress.yaml.tmp ingress.yaml
kustomize edit add resource ingress.yaml
popd
```

如果你使用的是 AKS，一种做法是启用集群中的 **approuting** 插件（更多信息见[此处](https://learn.microsoft.com/en-us/azure/aks/app-routing)），并将上面的环境变量设置为：
* `KBS_INGRESS_CLASS="webapprouting.kubernetes.azure.com"`
* **approuting** 插件不会创建托管的集群 DNS 区域，因此你需要自行创建该区域并将其关联到 **approuting**（更多信息见[此处](https://learn.microsoft.com/en-us/azure/aks/app-routing-dns-ssl#create-a-public-azure-dns-zone)）。然后将 `CLUSTER_SPECIFIC_DNS_ZONE` 设置为所创建的区域名称。
* 如果你不想要集群 DNS 区域，可以导出 `KBS_INGRESS_HOST="\"\""` 并使用 Ingress 的公网 IP：
  ```bash
  kubectl get service -n app-routing-system nginx -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
  ```

## 可选：使用非发布版镜像

有时可能希望使用一个并非仓库中所设定的镜像（通常是最新发布镜像）来部署 KBS。若要把部署改为使用某个暂存构建（staging build），请用 `kustomize` 设置镜像：

```bash
pushd base
kustomize edit set image kbs-container-image=ghcr.io/confidential-containers/staged-images/kbs:65ee7e1acccd13dcb515058e71c5f8bfb4281e35
popd
```

可用的镜像标签可以在 [CoCo packages listing](https://github.com/orgs/confidential-containers/packages?repo_name=trustee) 中找到。

## 可选：使用 Nodeport 暴露 KBS

如果你希望通过 Nodeport 暴露 KBS 服务，请导出以下环境变量：

```bash
export DEPLOYMENT_DIR=nodeport
```

部署完成 KBS 之后，你可以使用该服务的 nodeport 与 Kubernetes 节点的 IP 来访问 KBS。可以通过运行以下命令生成 KBS 的 URL：

```bash
echo $(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}'):$(kubectl get svc kbs -n coco-tenant -o jsonpath='{.spec.ports[0].nodePort}')
```

## 部署 KBS

通过运行以下命令部署 KBS：

```bash
./deploy-kbs.sh
```

当在启用了 [IBM Secure Execution](https://www.ibm.com/docs/en/linux-on-systems?topic=management-secure-execution) 的环境中部署 trustee 时，由于需要用到 IBM SE 验证器（verifier），还需要一个环境变量 `IBM_SE_CREDS_DIR`，它指向一个包含在 IBM Secure Execution 上进行远程证明所需的额外文件的目录：

```
$ export IBM_SE_CREDS_DIR=/path/to/your/directory
$ tree $IBM_SE_CREDS_DIR
/path/to/your/directory
├── certs
│   ├── DigiCertCA.crt
│   └── ibm-z-host-key-signing-gen2.crt
├── crls
│   └── ibm-z-host-key-gen2.crl
├── hdr
│   └── hdr.bin
├── hkds
│   └── HKD-3931-0275D38.crt
└── rsa
    ├── encrypt_key.pem
    └── encrypt_key.pub
5 directories, 7 files
```

详情请查阅[该文档](https://github.com/confidential-containers/trustee/tree/main/deps/verifier/src/se)。

> [!NOTE]
> 对于在非 TEE 的 s390x 环境上使用 sample 验证器运行 trustee 的非生产场景，不需要这个额外的 `IBM_SE_CREDS_DIR` 环境变量。

## 检查部署

运行以下命令来检查 KBS 是否部署成功：

```bash
kubectl -n coco-tenant get pods
```

成功的结果类似下面这样：

```console
$ kubectl -n coco-tenant get pods
NAME                  READY   STATUS    RESTARTS   AGE
kbs-bdffc8dd4-jv2kr   1/1     Running   0          7m30s
```

本次部署中同时也会部署一个 Kubernetes 服务，你可以通过它访问 KBS：

```console
$ kubectl -n coco-tenant get service
NAME   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kbs    ClusterIP   10.0.210.190   <none>        8080/TCP   4s
```

## 删除 KBS

```bash
$ kubectl delete -k ${DEPLOYMENT_DIR}/
```

或者，如果运行在 IBM Secure Execution 上，则运行：

```bash
$ kubectl delete -k ${DEPLOYMENT_DIR}/ibm-se/ && kubectl delete pv test-local-pv
```
