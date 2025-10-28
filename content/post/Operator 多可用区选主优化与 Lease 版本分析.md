---
title: Operator 多可用区选主优化与 Lease 版本分析
slug: Operator-multi-zone-master-election-optimization-and-lease-version-analysis
tags: [K8s, Git]
date: 2025-03-19T18:20:25+08:00
---

## 什么是 Lease？

Lease 对象的核心作用是表示某个实体（通常是一个 Pod 或进程）对某项资源或角色的“持有权”，并且这种持有权是有时间限制的。通过定期续约（renew），持有者可以保持其控制权。如果持有者未能续约，租约到期后，其他实体可以接管。

常见的应用场景包括：

1.  **领导选举**：在分布式系统中，确保只有一个实例（Leader）执行特定任务，其他实例作为 Follower。
2.  **资源协调**：跟踪和管理资源的临时所有权。
3.  **心跳机制**：通过续约时间（RenewTime）检测持有者是否仍然活跃。

Kubernetes 内部的一些组件（如 kube-controller-manager 和 kube-scheduler）就使用 Lease 来实现高可用性和领导选举。

## Lease 的工作原理

1.  **创建租约**：
    *   一个进程（如你的代码）创建一个 Lease 对象，并声明自己为持有者（HolderIdentity）。
2.  **续约**：
    *   持有者需要定期更新 RenewTime，证明自己仍然活跃。通常通过客户端（如 kubectl 或 Go 客户端）调用 Kubernetes API 来更新。
3.  **失效与接管**：
    *   如果持有者未能及时续约（例如进程崩溃），其他进程可以通过检查 RenewTime 和 LeaseDurationSeconds 判断租约是否过期，并尝试接管。
4.  **领导选举**：
    *   多个实例竞争同一 Lease 对象时，只有成功创建或更新它的实例成为 Leader。

## 示例场景

假设你用这个 Lease 来实现领导选举：

*   你有一个分布式应用，有 3 个 Pod：pod-1、pod-2、pod-3。
*   它们都尝试创建或更新同一个 Lease 对象（例如 my-leader-lease）。
*   pod-1 成功创建，设置 HolderIdentity: "pod-1"，成为 Leader。
*   pod-1 每 10 秒更新 RenewTime，保持领导地位。
*   如果 pod-1 崩溃，RenewTime 未更新，pod-2 检测到租约过期，接管并更新 HolderIdentity: "pod-2"。

## 现有逻辑

```go
// Copyright 2018 The Operator-SDK Authors
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
package main

import (
	"context"
	"time"

	"github.com/operator-framework/operator-sdk/pkg/k8sutil" // Operator-SDK 提供的工具包，用于获取运行环境信息（如命名空间、Pod 信息）
	corev1 "k8s.io/api/core/v1"                              // Kubernetes Core v1 API，包含 Pod、ConfigMap 等资源定义
	apierrors "k8s.io/apimachinery/pkg/api/errors"          // Kubernetes API 错误处理包
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"          // Kubernetes 元数据定义包，包含 ObjectMeta 等
	"k8s.io/apimachinery/pkg/util/wait"                    // 提供等待和退避逻辑的工具
	crclient "sigs.k8s.io/controller-runtime/pkg/client"   // controller-runtime 提供的通用客户端，用于操作 Kubernetes 资源
	"sigs.k8s.io/controller-runtime/pkg/client/config"     // 获取 Kubernetes 配置（如 kubeconfig）
)

// maxBackoffInterval 定义了尝试成为 Leader 时，最大退避等待时间（16秒）。
// 在多次尝试失败后，等待时间会逐渐增加，但不会超过这个值。
const maxBackoffInterval = time.Second * 16

// leaderBecome 函数确保当前 Pod 成为其命名空间内的 Leader。
// 如果代码运行在集群外（本地），会跳过领导选举直接返回 nil。
// 它通过创建一个带有当前 Pod 作为 OwnerReference 的 ConfigMap 来实现领导选举。
// ConfigMap 的名称是 lockName，同一时间只能有一个同名 ConfigMap 存在，成功创建者即为 Leader。
// 当 Leader Pod 终止时，垃圾回收机制会删除 ConfigMap，其他 Pod 可以接管。
func leaderBecome(ctx context.Context, lockName string) error {
	log.Info("Trying to become the leader.") // 日志：尝试成为 Leader

	// 获取当前 Operator 运行的命名空间
	ns, err := k8sutil.GetOperatorNamespace()
	if err != nil {
		// 如果无法获取命名空间（可能是本地运行或无权限）
		if err == k8sutil.ErrNoNamespace || err == k8sutil.ErrRunLocal {
			log.Info("Skipping leader election; not running in a cluster.") // 日志：不在集群中，跳过选举
			return nil
		}
		return err // 其他错误，返回
	}

	// 获取 Kubernetes 配置（通常来自 kubeconfig 或 ServiceAccount）
	config, err := config.GetConfig()
	if err != nil {
		return err // 配置获取失败，返回错误
	}

	// 创建 controller-runtime 的客户端，用于操作 Kubernetes 资源
	client, err := crclient.New(config, crclient.Options{})
	if err != nil {
		return err // 客户端创建失败，返回错误
	}

	// 获取当前 Pod 的 OwnerReference，表示当前 Pod 是谁
	owner, err := myOwnerRef(ctx, client, ns)
	if err != nil {
		return err // 获取 OwnerReference 失败，返回错误
	}

	// 检查是否已存在由当前 Pod 拥有的 ConfigMap（可能是进程重启的情况）
	existing := &corev1.ConfigMap{}
	key := crclient.ObjectKey{Namespace: ns, Name: lockName} // ConfigMap 的键（命名空间+名称）
	err = client.Get(ctx, key, existing)
	switch {
	case err == nil: // ConfigMap 已存在
		// 检查现有 ConfigMap 的 OwnerReferences
		for _, existingOwner := range existing.GetOwnerReferences() {
			if existingOwner.Name == owner.Name { // 如果 Owner 是当前 Pod
				log.Info("Found existing lock with my name. I was likely restarted.") // 日志：发现已有锁，可能是重启
				log.Info("Continuing as the leader.")                           // 日志：继续作为 Leader
				return nil                                                      // 直接返回，当前 Pod 已是 Leader
			}
			log.Info("Found existing lock", "LockOwner", existingOwner.Name) // 日志：发现其他 Pod 持有的锁
		}
	case apierrors.IsNotFound(err): // ConfigMap 不存在
		log.Info("No pre-existing lock was found.") // 日志：没有预先存在的锁
	default: // 其他错误
		log.Error(err, "Unknown error trying to get ConfigMap") // 日志：获取 ConfigMap 时发生未知错误
		return err
	}

	// 创建一个新的 ConfigMap，用作领导锁
	cm := &corev1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:            lockName,           // ConfigMap 名称，由 lockName 指定
			Namespace:       ns,                 // 命名空间
			OwnerReferences: []metav1.OwnerReference{*owner}, // 设置当前 Pod 为 Owner
		},
	}

	// 尝试创建 ConfigMap，成为 Leader
	backoff := time.Second // 初始退避时间为 1 秒
	for {                  // 无限循环，直到成功或上下文取消
		err := client.Create(ctx, cm) // 尝试创建 ConfigMap
		switch {
		case err == nil: // 创建成功
			log.Info("Became the leader.") // 日志：成功成为 Leader
			return nil                     // 返回，当前 Pod 成为 Leader
		case apierrors.IsAlreadyExists(err): // ConfigMap 已存在（其他 Pod 是 Leader）
			existingOwners := existing.GetOwnerReferences() // 获取当前 ConfigMap 的 Owner
			switch {
			case len(existingOwners) != 1: // ConfigMap 应该只有一个 Owner
				log.Info("Leader lock configmap must have exactly one owner reference.", "ConfigMap", existing)
			case existingOwners[0].Kind != "Pod": // Owner 必须是 Pod
				log.Info("Leader lock configmap owner reference must be a pod.", "OwnerReference", existingOwners[0])
			default: // 正常情况：ConfigMap 由一个 Pod 拥有
				leaderPod := &corev1.Pod{}
				key = crclient.ObjectKey{Namespace: ns, Name: existingOwners[0].Name} // 当前 Leader Pod 的键
				err = client.Get(ctx, key, leaderPod)                          // 获取 Leader Pod 信息
				switch {
				case apierrors.IsNotFound(err): // Leader Pod 已删除
					log.Info("Leader pod has been deleted, waiting for garbage collection do remove the lock.") // 日志：等待垃圾回收删除 ConfigMap
				case err != nil: // 获取 Pod 失败
					return err
				case isPodEvicted(*leaderPod) && leaderPod.GetDeletionTimestamp() == nil: // Leader Pod 被驱逐但未标记删除
					log.Info("Operator pod with leader lock has been evicted.", "leader", leaderPod.Name) // 日志：Leader 被驱逐
					log.Info("Deleting evicted leader.")                                          // 日志：删除被驱逐的 Leader
					err := client.Delete(ctx, leaderPod)                                          // 删除 Leader Pod
					if err != nil {
						log.Error(err, "Leader pod could not be deleted.") // 日志：删除失败
					}
				case leaderPod.GetDeletionTimestamp() != nil: // Leader Pod 已被标记删除（新增逻辑）
					log.Info("Operator pod with leader lock has been terminating/completed.", "leader", leaderPod.Name) // 日志：Leader Pod 正在终止
					log.Info("Deleting a-lock cm.")                                                       // 日志：删除 ConfigMap
					errCm := client.Delete(ctx, cm)                                                       // 删除 ConfigMap
					if errCm != nil {
						log.Error(errCm, "Leader pod could not be deleted.") // 日志：删除 ConfigMap 失败
					}
					err := client.Delete(ctx, leaderPod) // 删除 Leader Pod
					if err != nil {
						log.Error(err, "Leader pod could not be deleted.") // 日志：删除 Pod 失败
					}
				default: // Leader Pod 仍然活跃
					log.Info("Not the leader. Waiting.") // 日志：当前不是 Leader，继续等待
				}
			}
			// 等待一段时间后重试，带有随机抖动（Jitter）避免所有 Follower 同时竞争
			select {
			case <-time.After(wait.Jitter(backoff, .2)): // 等待 backoff 时间（带 20% 随机抖动）
				if backoff < maxBackoffInterval { // 如果退避时间未达到最大值
					backoff *= 2 // 退避时间翻倍
				}
				continue // 继续下一次循环
			case <-ctx.Done(): // 上下文取消
				return ctx.Err() // 返回上下文错误
			}
		default: // 创建 ConfigMap 时发生其他错误
			log.Error(err, "Unknown error creating ConfigMap") // 日志：未知错误
			return err
		}
	}
}

// myOwnerRef 返回当前运行 Pod 的 OwnerReference。
// 它依赖环境变量 POD_NAME（通过 downwards API 设置）来识别当前 Pod。
func myOwnerRef(ctx context.Context, client crclient.Client, ns string) (*metav1.OwnerReference, error) {
	// 获取当前 Pod 的信息
	myPod, err := k8sutil.GetPod(ctx, client, ns)
	if err != nil {
		return nil, err // 获取失败，返回错误
	}

	// 构造 OwnerReference
	owner := &metav1.OwnerReference{
		APIVersion: "v1",              // API 版本
		Kind:       "Pod",             // 资源类型
		Name:       myPod.ObjectMeta.Name, // Pod 名称
		UID:        myPod.ObjectMeta.UID,  // Pod 的唯一标识符
	}
	return owner, nil
}

// isPodEvicted 检查 Pod 是否被驱逐（Evicted）。
// Pod 被驱逐时，Phase 为 Failed，且 Reason 为 "Evicted"。
func isPodEvicted(pod corev1.Pod) bool {
	podFailed := pod.Status.Phase == corev1.PodFailed // Pod 状态为 Failed
	podEvicted := pod.Status.Reason == "Evicted"       // 原因是 Evicted
	return podFailed && podEvicted                     // 两者都满足则为被驱逐
}
```

### 现有逻辑总结

1.  **目标**：
    *   通过创建一个 ConfigMap（名称为 a-lock）来实现领导选举，成功创建者成为 Leader。
    *   ConfigMap 的 OwnerReference 指向当前 Pod，当 Pod 删除时，ConfigMap 会被垃圾回收。
2.  **流程**：
    *   **检查环境**：如果不在集群中运行，跳过领导选举。
    *   **检查现有锁**：如果已有由当前 Pod 拥有的 ConfigMap，直接恢复为 Leader。
    *   **创建锁**：尝试创建新的 ConfigMap，如果失败（已有其他 Leader），检查当前 Leader 状态。
    *   **处理失效**：
        *   如果 Leader Pod 已删除，等待垃圾回收。
        *   如果 Leader Pod 被驱逐或标记删除，主动删除 Pod 和 ConfigMap。
        *   否则，等待并重试（带有退避机制）。

### 与 Lease 的对比

*   **相似点**：都用于领导选举，依赖 Kubernetes API 的原子性操作。
*   **不同点**：
    *   Lease 是专门为领导选举设计的轻量资源，内置续约机制（RenewTime）。
    *   ConfigMap 是通用资源，通过 OwnerReference 和垃圾回收间接实现接管，逻辑更复杂。
*   **请求开销**：
    *   Lease 需要定期续约和查询。
    *   此代码使用轮询和退避，Follower 需反复尝试创建 ConfigMap，也会产生多次请求。

## 为什么基于 ConfigMap 的方案在主机房断电后表现不佳？（另一个 Operator 没能及时成为 Leader）

### 1. ConfigMap 依赖垃圾回收机制

*   **机制**：
    *   在现有代码中，ConfigMap 的 OwnerReference 指向 Leader Pod。当 Leader Pod 被删除时，Kubernetes 的垃圾回收器（Garbage Collector, GC）会自动删除 ConfigMap，从而允许新的 Pod 成为 Leader。
*   **主机房断电的影响**：
    *   如果主机房断电，运行 Leader Pod 的节点会突然下线。此时，Kubernetes 集群需要检测到节点不可用，并将该节点上的 Pod 标记为删除（DeletionTimestamp）。
    *   **问题**：节点失联后，Kubernetes 不会立即认为 Pod 已删除，而是依赖 **Node Controller** 和 **Taint-based Eviction** 等机制来更新状态，这个过程可能需要几分钟甚至更久。
*   **延迟来源**：
    *   **Node NotReady 检测**：Master 检测节点失联通常依赖 node.kubernetes.io/not-ready 和 node.kubernetes.io/unreachable 污点，默认超时（node-monitor-grace-period）是 40 秒。
    *   **Pod Eviction 超时**：Pod 被驱逐的默认超时（eviction-hard 或 eviction-soft）可能配置为几分钟（常见默认值如 5 分钟）。
    *   **垃圾回收延迟**：即使 Pod 被标记删除，GC 删除 ConfigMap 也有一定延迟（通常几秒到几分钟，取决于集群负载和 GC 调度）。

### 2. 代码中的轮询和退避逻辑

*   **机制**：
    *   现有代码使用轮询方式尝试创建 ConfigMap，如果失败（IsAlreadyExists），会检查现有 Leader Pod 状态，并以指数退避（backoff）等待。
    *   退避时间从 1 秒开始，每次翻倍，直到最大值 maxBackoffInterval = 16 秒。
*   **主机房断电的影响**：
    *   当 Leader Pod 的节点断电后，Follower Pod 检测到 ConfigMap 仍存在，但无法立即接管。它会进入等待循环，每次等待时间逐渐增加（1s → 2s → 4s → 8s → 16s）。
    *   **问题**：如果 GC 迟迟未删除 ConfigMap，Follower 的等待时间会累积。例如，等待 5 次退避的总时间是 1 + 2 + 4 + 8 + 16 = 31 秒，但如果需要更多次（因为 GC 延迟），时间会更长。
*   **延迟来源**：
    *   **退避累积**：退避逻辑本身会拖慢接管速度。
    *   **未优化检测**：代码依赖主动轮询，而非实时监听（Watch），无法立即感知 ConfigMap 删除。

### 3. Kubernetes 集群的容错机制

*   **机制**：
    *   Kubernetes 在设计时倾向于“谨慎”处理节点失联，避免误判（false positive）。这意味着在确认节点和 Pod 真正失效前，会有较长的宽限期。
*   **主机房断电的影响**：
    *   如果整个主机房断电，多个节点可能同时失联，Master 的 Node Controller 和 API Server 负载会激增，导致状态更新变慢。
    *   **问题**：Pod 和 ConfigMap 的状态更新可能被推迟，进一步延长新 Leader 上任时间。
*   **延迟来源**：
    *   **Node Controller 过载**：大规模节点失效可能导致状态同步延迟。
    *   **API Server 压力**：大量资源状态变更可能使 API 调用变慢。

### 4. 代码中的改进逻辑（DeletionTimestamp）

*   **机制**：
    *   现有代码新增了检查 leaderPod.GetDeletionTimestamp() != nil，当 Leader Pod 被标记删除时，主动删除 ConfigMap 和 Pod。
*   **主机房断电的影响**：
    *   这个逻辑依赖 Pod 被标记删除，但断电后 Pod 状态更新依赖 Node Controller，可能需要几分钟。
    *   **问题**：如果节点失联时间过长，DeletionTimestamp 不会立即设置，主动删除 ConfigMap 的逻辑无法触发。
*   **延迟来源**：
    *   **状态更新延迟**：节点失联到 Pod 被标记删除的时间（默认可能 5-10 分钟）。

#### 断电场景：15 分钟的延迟虽然不常见，但并非完全不合理，尤其在以下条件下：

1.  **Node Eviction 超时较长**：如果集群配置了较长的宽限期（如 kube-controller-manager 的 `--pod-eviction-timeout=5m` 或更高），Pod 状态更新可能延迟 5-10 分钟。
2.  **GC 延迟**：大规模断电后，GC 处理大量资源，调度延迟可能累加到几分钟。
3.  **退避累积**：Follower 的退避逻辑在极端情况下可能执行多次（比如 10-20 次），每次 16 秒，总计 160-320 秒（约 3-5 分钟）。
4.  **集群负载**：如果 Master 或 API Server 因断电事件过载，状态同步可能进一步推迟。
5.  **估算**：
    *   Node 失联检测：5 分钟（默认宽限期）。
    *   Pod 驱逐 + GC：2-5 分钟。
    *   Follower 退避：3-5 分钟。

### 与 Lease 方案的对比

*   **Lease 的优势**：
    *   **主动续约**：Leader 定期更新 RenewTime，Follower 可通过时间戳判断租约是否过期，无需依赖 GC。
    *   **更快接管**：租约过期后（通常 15-30 秒），Follower 可立即尝试接管。
    *   **典型延迟**：在断电场景下，接管时间通常在几十秒到 1-2 分钟（取决于 LeaseDurationSeconds 和网络恢复）。
*   **现有 ConfigMap 方案**：
    *   接管时间受限于 Node 状态更新和 GC，异常场景下可能高达数十分钟。

## Lease 资源在 Kubernetes 中的版本局限性分析

## 背景

Kubernetes 中的 Lease 资源（隶属 coordination.k8s.io API 组）用于节点心跳和领导选举等协调机制，是一个不算核心但也不可忽视的功能。然而，官方文档（[https://kubernetes.io/docs/reference/](https://kubernetes.io/docs/reference/)）对 Lease 的版本演进记录极为有限：

*   未明确说明何时引入 v1beta1（Beta）。
*   未记录 v1 何时成为稳定版（GA）。
*   仅在 API 参考中提供当前版本（比如 1.29）的字段说明，历史信息几乎为零。

这种文档缺失迫使我们通过自行测试和源码分析来确定 Lease 的版本支持情况。下文记录了这一过程，包括使用的命令、每步的意义，以及最终结论。

---

## 查找过程与命令记录

### 1. 检查集群支持的 API 版本

```bash
kubectl get --raw /apis/coordination.k8s.io | python -m json.tool
```

*   **意义**：通过 Kubernetes API Server 的 `/apis/coordination.k8s.io` 端点，获取当前集群支持的 coordination.k8s.io API 版本（比如 v1beta1 或 v1）。`python -m json.tool` 美化 JSON 输出，便于阅读。
*   **目的**：验证运行时环境是否支持 Lease 的某个版本，但无法揭示历史。

### 2. 查看客户端和服务器版本

```bash
kubectl version --short
```

*   **意义**：显示 kubectl 客户端和 API Server 的版本（如 v1.16.15 或 v1.20.11），确保测试环境明确。
*   **目的**：为后续测试提供版本上下文，避免混淆。

### 3. 克隆 Kubernetes 源码

```bash
git clone https://github.com/kubernetes/kubernetes.git
```

*   **意义**：获取 Kubernetes 完整源码仓库，包含所有版本的历史记录。
*   **目的**：由于文档不足，直接从源码追溯 Lease 的引入时间。

### 4. 搜索包含 "lease" 的文件

```bash
grep -lER '\blease' ./* | grep -vE '(docs|test|e2e|vendor)'
```

*   **意义**：
    *   `grep -lER '\blease' ./*`：递归搜索当前目录下所有包含单词 "lease" 的文件，`-l` 只输出文件名，`\b` 确保匹配完整单词。
    *   `grep -vE '(docs|test|e2e|vendor)'`：排除文档、测试和依赖目录，聚焦核心代码。
*   **目的**：定位 Lease 定义的关键文件（如 `staging/src/k8s.io/api/coordination/v1beta1/types.go` 和 `v1/types.go`）。

### 5. 切换到最新分支

```bash
git checkout master
```

*   **意义**：确保从最新代码开始分析，包含所有历史提交。
*   **目的**：避免从某个旧提交开始遗漏后续变更。

### 6. 查找 v1beta1 的首次出现

```bash
git log --follow --name-only -- staging/src/k8s.io/api/coordination/v1beta1/types.go | tail
```

*   **意义**：
    *   `--follow`：追踪文件重命名历史。
    *   `--name-only`：只显示提交涉及的文件名。
    *   `| tail`：取最后几行，找到最早提交。
*   **结果**：

```text
commit f38e952f4eb3a5f4996a2a41bd0eedc98b6637de
Author: wojtekt <wojtekt@google.com>
Date:   Tue May 22 16:34:25 2018 +0200

    Add coordination API group with Lease type

staging/src/k8s.io/api/coordination/v1beta1/types.go
```

*   **目的**：确定 v1beta1 的代码引入点。

### 7. 检查 f38e952f 的版本

```bash
git tag --contains f38e952f4eb3a5f4996a2a41bd0eedc98b6637de
```

*   **意义**：列出包含此提交的所有 Tag，揭示它首次出现在哪个版本。
*   **结果**：包含 v1.12.0 及后续版本。
*   **目的**：确认 v1beta1 的代码从 1.12.0 开始存在。

### 8. 验证 f38e952f 的文件状态

```bash
git checkout f38e952f4eb3a5f4996a2a41bd0eedc98b6637de
ls -l staging/src/k8s.io/api/coordination/v1beta1/types.go
ls -l staging/src/k8s.io/api/coordination/v1/types.go
```

*   **意义**：
    *   `git checkout`：切换到该提交。
    *   `ls -l`：检查文件是否存在。
*   **结果**：
    *   `v1beta1/types.go` 存在。
    *   `v1/types.go` 不存在。
*   **目的**：确认 f38e952f 只引入了 v1beta1，无 v1。

### 9. 检查 1.12.0 的文件状态

```bash
git checkout v1.12.0
ls -l staging/src/k8s.io/api/coordination/v1/types.go
ls -l staging/src/k8s.io/api/coordination/v1beta1/types.go
```

*   **意义**：验证 1.12.0 版本的文件状态。
*   **结果**：同上，v1beta1 有，v1 无。
*   **目的**：进一步确认 1.12.0 只包含 v1beta1。

### 10. 查找 v1 的首次出现（尝试 1）

```bash
git log --follow --diff-filter=A --name-only -- staging/src/k8s.io/api/coordination/v1/types.go | tail
```

*   **意义**：
    *   `--diff-filter=A`：只显示文件添加的提交。
    *   `--follow`：追踪重命名。
*   **结果**：错误输出 v1beta1/types.go（f38e952f）。
*   **问题**：`--follow` 误将 v1 历史追溯到 v1beta1。

### 11. 查找 v1 的首次出现（尝试 2）

```bash
git log --follow --diff-filter=A --name-only -- staging/src/k8s.io/api/coordination/v1/types.go
```

*   **意义**：完整输出，避免 `tail` 截断。
*   **结果**：仍错误指向 v1beta1。
*   **问题**：同上，`--follow` 干扰。

### 12. 查找 v1 的首次出现（修正）

```bash
git log --diff-filter=A --name-only -- staging/src/k8s.io/api/coordination/v1/types.go
```

*   **意义**：去掉 `--follow`，直接找 `v1/types.go` 的添加。
*   **结果**：

```text
commit 73d14dede6bd6915b608f5fd86c20c9a322847be
Author: wojtekt <wojtekt@google.com>
Date:   Wed Dec 19 16:22:05 2018 +0100

    Promote Lease API to v1

staging/src/k8s.io/api/coordination/v1/types.go
```

*   **目的**：准确找到 v1 的代码引入点。

### 13. 检查 73d14dede 的版本

```bash
git tag --contains 73d14dede6bd6915b608f5fd86c20c9a322847be
```

*   **意义**：确认 v1 引入的版本。
*   **结果**：包含 v1.14.0 及后续版本。
*   **目的**：锁定 v1 从 1.14.0 开始存在。

---

## 结论

通过源码分析和命令追溯，Lease 资源的版本支持情况如下：

*   **coordination.k8s.io/v1beta1**：
    *   **代码引入**：1.12.0（f38e952f，2018-05-22）。
    *   **运行支持**：1.13.0（Alpha，PR #64503，默认关闭，需 feature gate）。
    *   **说明**：1.12.0 仅定义代码，未启用；1.13.0 通过 KEP-589 实现 Alpha 支持。
*   **coordination.k8s.io/v1**：
    *   **代码引入**：1.14.0（73d14dede，2018-12-19）。
    *   **Beta**：1.14.0（默认启用）。
    *   **稳定（GA）**：1.17.0（官方文档和社区确认）。
    *   **说明**：1.14.0 添加 v1 剧本并提升为 Beta，1.17.0 正式稳定。
*   **局限性**：
    *   官方文档未明确记录版本演进（Alpha、Beta、GA 时间点）。
    *   需通过源码（f38e952f 和 73d14dede）和 PR（#64503、#71735）推断。
    *   早期版本（如 1.13）需手动测试（kind 或 minikube）确认运行支持。

---

## 补充说明

*   **文档不足**：Kubernetes 文档对 Lease 的版本历史几乎无记录，CHANGELOG 提及零散，KEP 和 PR 信息分散，导致用户需自行挖掘。
*   **建议**：可向 Kubernetes 社区提交 PR（如改进 website 仓库的 API 参考），补充版本演进细节。
*   **验证方法**：使用 kind 创建历史版本集群（如 kindest/node:v1.13.12），测试 `kubectl apply` 是否支持 v1beta1 或 v1。