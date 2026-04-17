---
title: StatefulSet 的 OnDelete 策略下，currentRevision 为什么一直不更新？
slug: statefulset-ondelete-currentrevision
tags: [K8s, Operator]
date: 2026-04-17T13:20:00+08:00
---

最近遇到一个很典型的云原生问题：业务明明已经完成了 Pod 重建，结果实例变更还是被平台拦住了。继续往下看，发现不是 Operator 逻辑有问题，也不是业务没重启成功，而是 StatefulSet 在 `OnDelete` 策略下，一个存在很多年的状态收敛问题被 PaaS 的检测逻辑放大了。<!--more-->

这篇文章不打算聊八卦，也不打算纯吐槽。我想把这件事按技术问题的方式讲清楚：

1. 现象是什么
2. 根因在哪一层
3. 这是不是 Kubernetes 的既定设计
4. 临时应该怎么处理
5. 长期到底该推动谁来修
6. 截至现在，上游社区有没有进展

## 问题现象

实例在做部署、重启这类变更时，有概率出现这样一种情况：

- Pod 实际上已经被替换过
- 新 Pod 的 `controller-revision-hash` 也是新的
- 但是 StatefulSet 的 `status.currentRevision` 仍然停留在旧值
- `status.updateRevision` 则已经是新值

一旦平台新增了类似下面这种检测：

```text
currentRevision != updateRevision
```

就会把这个 StatefulSet 直接判定为 `not ready`，从而阻塞变更流程。

问题就出在这里。

如果这是业务没升级成功，那这个检测没有问题。  
但如果 Pod 实际已经是新版本，只是 StatefulSet 的 `status` 没有收敛，那平台拿这个字段做强校验，就会产生误判。

## 为什么我们一定会踩这个坑

因为我们的服务不是靠 StatefulSet 自己做滚动更新，而是靠 Operator 控制 Pod 重启和数据迁移。

这意味着 StatefulSet 只能使用 `OnDelete`，而不能切到 `RollingUpdate`。  
因为在这个场景里，Pod 什么时候删、删哪个 Pod、删完之后要不要做数据迁移，决定权都在 Operator 手里，不在原生 StatefulSet controller 手里。

也正因为如此，这个问题不是一个“边角 case”，而是只要你的架构是：

- StatefulSet
- `OnDelete`
- Operator 接管滚动过程
- 平台又依赖 StatefulSet 的 revision 状态做准入判断

那它迟早会暴露出来。

## 排查到最后，根因其实很明确

网上一查，这不是新问题。

2019 年 Kubernetes 社区就有人提了这个 issue：

- [kubernetes/kubernetes#73492](https://github.com/kubernetes/kubernetes/issues/73492)

issue 里描述得非常直接：

- StatefulSet 使用 `OnDelete`
- Pod 被手工删除后，确实会按新模板重建
- `updatedReplicas` 这类状态也会变化
- 但 `currentRevision` 不会推进到 `updateRevision`

这就说明一件事：  
**不是 Pod 没更新，而是 StatefulSet controller 没把 status 维护完整。**

这一步其实很关键。因为很多人看到 `currentRevision != updateRevision`，第一反应会怀疑：

- 是不是 Operator 自己没删干净
- 是不是 Pod 里混了老版本
- 是不是业务没完成升级

但如果你去看 Pod 的 `controller-revision-hash`，再结合社区 issue，就会发现锅不在这一层。

## 这是不是 Kubernetes 的既定设计

不是。

这个问题必须拆开说，不然很容易混在一起。

### 第一层：OnDelete 的确是“手工触发更新”

Kubernetes 官方文档对 `OnDelete` 的定义很明确：

- StatefulSet controller 不会自动更新 Pod
- 必须由用户或者外部控制器删除 Pod
- Pod 被删除后，才会按最新模板重新创建

这一点没有争议。

### 第二层：status 仍然应该由 controller 维护

争议不在 `OnDelete` 会不会自动滚动，而在 `status.currentRevision` 到底该不该更新。

我的结论很明确：**该更新，而且本来就不该由用户维护。**

原因很简单，`currentRevision` 和 `updateRevision` 都在 StatefulSet 的 `status` 里，而 Kubernetes 的 API 设计是：

- `spec` 表示期望状态
- `status` 表示系统观测状态

也就是说，`status` 的职责天然属于 controller。

如果一个字段放在 `status` 里，最后却需要业务方自己 patch 才能收敛，那这不是“设计如此”，而是 controller 在某条路径上没有把状态维护完整。

所以这个问题的本质，不是：

> OnDelete 就是这样，你应该自己维护 currentRevision

而是：

> OnDelete 下 StatefulSet controller 没有把 currentRevision 正常推进到 updateRevision

这两个结论看起来只差一句话，但责任边界完全不同。

## 为什么这个 bug 会把人绕进去

因为它不是那种“集群马上炸掉”的 bug。

如果这是一个会导致 Pod 起不来、PVC 丢失、apiserver 崩掉的问题，那它的优先级会非常高。  
但它偏偏是一个“状态字段不准确”的问题。

在很多环境里，这种问题甚至不会立刻暴露，因为：

- 业务照样能跑
- Pod 确实也更新了
- 没有人去盯 `currentRevision`

但一旦平台把这个字段当成 readiness 的强依据，问题性质就变了。

原本只是 status 不够准确，最后却会演变成：

- 业务已经完成升级
- Operator 也按顺序完成了 Pod 替换
- 平台因为一个 stale 的状态字段，判定整个变更不通过

这也是为什么我后来觉得，这个问题真正该背锅的，不只有 Kubernetes，上层平台的检测策略也有问题。

## 临时解决方案

先说结论：我最后写了一个脚本，用来 patch StatefulSet 的 `/status` 子资源，把 `currentRevision` 对齐到 `updateRevision`。

它不是长期方案，但在确认问题就是这个已知 bug 的前提下，可以作为临时兜底手段。

脚本如下：

```bash
#!/bin/bash

# 使用方法：sh fix_statefulset_current_revision.sh <namespace> <statefulset_name>
# 注意事项: 使用前一定要明确是 OnDelete 的升级策略 bug 导致 currentRevision 没有正确更新

if [ "$#" -ne 2 ]; then
    echo "使用前一定要明确是 OnDelete 的升级策略 bug 导致 currentRevision 没有正确更新"
    echo "用法: $0 <namespace> <statefulset_name>"
    exit 1
fi

namespace="$1"
statefulset_name="$2"

# 获取 API Server 地址
APISERVER=$(kubectl config view --minify --raw | grep server | awk '{ print $2 }')

# 获取 Token
TOKEN_NAME=$(kubectl get sa -n elasticsearch elasticsearch-operator-product -o jsonpath='{.secrets[0].name}')
TOKEN=$(kubectl get secret -n elasticsearch $TOKEN_NAME -o jsonpath='{.data.token}' | base64 --decode)

# 获取 StatefulSet 的 status.updateRevision
update_revision=$(kubectl get sts -n $namespace $statefulset_name -o jsonpath='{.status.updateRevision}')

# StatefulSet 的补丁 URL
resource_url="/apis/apps/v1/namespaces/$namespace/statefulsets/$statefulset_name/status"

# 生成 JSON 补丁
patch=$(cat <<EOF
{
  "status": {
    "currentRevision": "$update_revision"
  }
}
EOF
)

# 输出当前状态
echo "修改前的 StatefulSet status:"
kubectl get sts -n $namespace $statefulset_name -o yaml | grep -i "revision"

# 应用补丁
echo "开始修改 StatefulSet status..."
curl -XPATCH \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/merge-patch+json" \
     --data "$patch" \
     "$APISERVER$resource_url" --insecure

# 输出更新后的状态
echo "修改后的 StatefulSet status:"
kubectl get sts -n $namespace $statefulset_name -o yaml | grep -i "revision"
```

这个脚本做的事情其实很简单：

1. 读取目标 StatefulSet 当前的 `status.updateRevision`
2. 调用 `/status` 子资源接口
3. 把 `status.currentRevision` patch 成相同值

从技术上说，这比直接改 etcd 要克制得多，因为它至少还是走 Kubernetes API Server。  
但从职责边界看，它依然只是 workaround，本质上是在替 StatefulSet controller 补账。

所以这个脚本能解决问题，不代表这个动作合理到可以长期保留。

## 这个脚本什么时候能用，什么时候不能用

我自己的边界是这样的。

### 可以用的前提

必须先确认以下几点：

- StatefulSet 的更新策略确实是 `OnDelete`
- Pod 已经全部按新模板重建
- Pod 的 `controller-revision-hash` 已经和 `updateRevision` 对齐
- 业务实例本身已经进入健康状态
- 唯一不一致的就是 StatefulSet 的 `status.currentRevision`

只有在这种情况下，这个 patch 才是在“修正一个错误状态”。

### 不应该乱用的情况

如果你还没有确认 Pod 是否真的都升级了，就直接 patch `currentRevision`，那就不是修状态，而是在伪造状态。

这两者差别很大。

所以我不建议把这个脚本包装成一个看起来很万能的“修复工具”，它更适合被定义成：

> 在确认命中 Kubernetes 已知 bug 之后，用于解除平台误判的临时处置脚本

## 长期方案应该怎么选

我觉得真正靠谱的方向只有两个。

### 方案一：PaaS 修检测逻辑

这是我最认同的方案。

原因不复杂：  
既然 `OnDelete` 下 `currentRevision` 长期存在失真风险，那平台就不应该拿：

```text
currentRevision != updateRevision
```

直接作为强门禁。

更合理的做法应该是：

- 如果是 `OnDelete`，降低这条规则的优先级
- 或者直接跳过这条规则
- 或者改查 Pod 实际的 `controller-revision-hash`
- 再配合 `readyReplicas`、`updatedReplicas` 去做判断

因为在这个问题里，真正不可靠的不是 Pod 状态，而是 StatefulSet 的这两个 revision 字段关系。

### 方案二：Operator 做托底

如果 PaaS 侧一时改不了，那第二个可接受的方案，就是把这段 patch 逻辑收进 Operator。

也就是：

- Operator 先判断 Pod 是否已经全部完成切换
- 如果确认全部切换完成，但 StatefulSet 的 `currentRevision` 仍然没更新
- 再由 Operator 统一 patch `/status`

这样至少责任边界还是清楚的。

不是业务用户自己改 status，  
而是自家控制器对上游已知缺陷做补偿。

## 上游进展到哪了

我又去核对了一遍资料，截至 `2026-04-17`，结论是：

**这个问题仍然没有在 Kubernetes 主线里正式修复发布。**

可以确认的信息有这些：

- 2019-01-29，这个问题被正式提出来  
  [kubernetes/kubernetes#73492](https://github.com/kubernetes/kubernetes/issues/73492)
- 这个 issue 到今天还是 `Open`
- 标签里仍然能看到 `kind/bug`、`needs-triage`、`sig/apps`
- issue 页面已经关联到了新的修复 PR  
  [kubernetes/kubernetes#136833](https://github.com/kubernetes/kubernetes/pull/136833)

这说明两件事：

第一，这不是一个“社区不承认”的问题。  
第二，这也还不是一个“已经修完，等发版就行”的问题。

换句话说，上游并不是完全没动，但也还远没到可以指望近期彻底落地的程度。

## 这不是只有我们遇到的问题

这件事之所以值得写，不只是因为我们踩到了，而是因为很多做 Operator 的项目，其实都在和同一个语义缺口打交道。

我查到几个比较有代表性的例子：

- Elastic 早年在讨论是否切到 StatefulSet 方案时，就明确讨论过 `OnDelete` 和 revision 跟踪的问题  
  [elastic/cloud-on-k8s#1173](https://github.com/elastic/cloud-on-k8s/issues/1173)
- Elastic 的用户社区里，2021 年就有人因为 `currentRevision` 不更新而监控误报  
  [ECK Statefulset currentRevision not updated](https://discuss.elastic.co/t/eck-statefulset-currentrevision-not-updated/263155)
- VictoriaMetrics Operator 在 2022 年的 changelog 里，直接提到了修复 StatefulSet `currentRevision` / `updateRevision` 的问题  
  [VictoriaMetrics Operator changelog](https://docs.victoriametrics.com/operator/changelog/)

所以这不是“你们业务写挂了”，而是：

**只要你用 StatefulSet + OnDelete + 外部控制器接管滚动，这个问题迟早会浮出水面。**

## 我对这个问题的最终判断

现在回头看，这件事我会这样定义：

它不是一个“用户自己维护 status”的设计问题，  
也不是一个“单纯吐槽 Kubernetes 老 bug”的情绪问题。

它本质上是一个跨层问题：

- Kubernetes 在 `OnDelete` 路径下存在 status 收敛缺口
- Operator 出于业务要求必须接管更新流程
- PaaS 又把一个已知不稳定的字段当成了强检测依据

这三件事叠在一起，才会最终表现成“实例变更被阻塞”。

如果只盯着表象，很容易把问题看成：

> StatefulSet 没 ready

但如果把链路往下拆，就会发现更准确的说法其实是：

> 平台把一个上游已知存在缺陷的状态字段，当成了严格的发布门禁

这两个说法差别很大。前者是在看现象，后者是在做归因。

## 总结

最后用几句话收一下：

- `OnDelete` 不会自动滚动更新 Pod，这没问题
- 但 `status.currentRevision` 长期不更新，不是“用户自己维护 status”的设计
- 这本质上是 StatefulSet controller 在 `OnDelete` 路径上的状态收敛缺口
- 临时可以通过 patch `/status` 来解除平台误判，但这只是 workaround
- 更合理的长期方案，要么是 PaaS 修检测逻辑，要么是 Operator 做托底
- 截至 `2026-04-17`，上游问题仍未正式修复发布

如果你也在平台里用了基于 StatefulSet revision 的 ready 检测，而且工作负载又依赖 `OnDelete`，建议尽快检查一下。  
这种问题平时很安静，一旦被平台规则激活，往往就是在最不想被卡住的时候卡住。

## 参考链接

- Kubernetes StatefulSet 文档：<https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/>
- Kubernetes issue #73492：<https://github.com/kubernetes/kubernetes/issues/73492>
- Kubernetes PR #136833：<https://github.com/kubernetes/kubernetes/pull/136833>
- Elastic Cloud on Kubernetes issue #1173：<https://github.com/elastic/cloud-on-k8s/issues/1173>
- Elastic 讨论帖：<https://discuss.elastic.co/t/eck-statefulset-currentrevision-not-updated/263155>
- VictoriaMetrics Operator changelog：<https://docs.victoriametrics.com/operator/changelog/>
