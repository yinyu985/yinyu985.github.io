---
title: ElasticSearch中熔断器
slug: Circuit-breaker-in-ElasticSearch
tags: [ELK]
date: '2023-11-20T21:40:25+08:00'
---

> Elasticsearch Service 提供了多种官方的熔断器（circuit breaker），用于防止内存使用过高导致 ES 集群因为 OutOfMemoryError 而出现问题。Elasticsearch 设置有各种类型的子熔断器，负责特定请求处理的内存限制。此外，还有一个父熔断器，用于限制所有子熔断器上使用的内存总量。

## Circuit breaker settings **断路器设置**

Elasticsearch contains multiple circuit breakers used to prevent operations from causing an OutOfMemoryError. Each breaker specifies a limit for how much memory it can use. Additionally, there is a parent-level breaker that specifies the total amount of memory that can be used across all breakers. Elasticsearch 包含多个断路器，用于防止操作导致 OutOfMemoryError。每个断路器都指定了其可以使用的内存量的限制。此外，还有一个父级断路器，用于指定可在所有断路器中使用的内存总量。

Except where noted otherwise, these settings can be dynamically updated on a live cluster with the cluster-update-settings API. 除非另有说明，否则可以使用 cluster-update-settings API 在实时集群上动态更新这些设置。

For information about circuit breaker errors, see Circuit breaker errors. 有关断路器错误的信息，请参阅断路器错误。

下面提到的静态和动态的意思是，这个配置参数的属性。

**动态设置（Dynamic Settings）：**

可以使用集群更新设置 API 在运行中的集群上进行配置和更新。  
在未启动或已关闭的节点上，也可以通过 `elasticsearch.yml` 文件进行本地配置。

**静态设置（Static Settings）：**

只能在未启动或已关闭的节点上通过 `elasticsearch.yml` 文件进行配置。  
必须在集群中的每个相关节点上进行设置。  
主要用于配置静态的集群设置和节点设置。  
静态设置的配置只在节点启动时生效，不会受到集群运行时的影响。

## **Parent circuit breaker 父断路器**

The parent-level breaker can be configured with the following settings: 可以使用以下设置配置父级断路器：

- `indices.breaker.total.use_real_memory`
  - (Static) Determines whether the parent breaker should take real memory usage into account (true) or only consider the amount that is reserved by child circuit breakers (false). Defaults to true.
  - （静态）确定父断路器是应考虑实际内存使用情况（true）还是仅考虑子断路器保留的内存量（false）。默认值为 true。
- `indices.breaker.total.limit`
  - (Dynamic) Starting limit for overall parent breaker. Defaults to 70% of JVM heap if `indices.breaker.total.use_real_memory` is false. If `indices.breaker.total.use_real_memory` is true, defaults to 95% of the JVM heap.
  - （动态）整体父断路器的起始限制。如果 `indices.breaker.total.use_real_memory` 为 false，默认为 JVM 堆的 70%；如果为 true，默认为 JVM 堆的 95%。

## **Field data circuit breaker 字段数据断路器**

The field data circuit breaker estimates the heap memory required to load a field into the field data cache. If loading the field would cause the cache to exceed a predefined memory limit, the circuit breaker stops the operation and returns an error. 字段数据断路器估计将字段加载到字段数据缓存中所需的堆内存。如果加载该字段会导致缓存超出预定义的内存限制，则断路器将停止操作并返回错误。

- `indices.breaker.fielddata.limit`
  - (Dynamic) Limit for fielddata breaker. Defaults to 40% of JVM heap.
  - （动态）字段数据断路器的限制。默认为 JVM 堆的 40%。
- `indices.breaker.fielddata.overhead`
  - (Dynamic) A constant that all field data estimations are multiplied with to determine a final estimation. Defaults to 1.03.
  - （动态）一个常量，所有字段数据估计值都乘以该常量以确定最终估计值。默认值为 1.03。

## **Request circuit breaker 请求断路器**

The request circuit breaker allows Elasticsearch to prevent per-request data structures (for example, memory used for calculating aggregations during a request) from exceeding a certain amount of memory. 请求断路器允许 Elasticsearch 防止每个请求的数据结构（例如，用于在请求期间计算聚合的内存）超过一定数量的内存。

- `indices.breaker.request.limit`
  - (Dynamic) Limit for request breaker, defaults to 60% of JVM heap.
  - （动态）请求断路器的限制，默认为 JVM 堆的 60%。
- `indices.breaker.request.overhead`
  - (Dynamic) A constant that all request estimations are multiplied with to determine a final estimation. Defaults to 1.
  - （动态）一个常量，所有请求估计值都乘以该常量以确定最终估计值。默认值为 1。

## **In flight requests circuit breaker 传输中请求断路器**

The in-flight requests circuit breaker allows Elasticsearch to limit the memory usage of all currently active incoming requests on transport or HTTP level from exceeding a certain amount of memory on a node. The memory usage is based on the content length of the request itself. This circuit breaker also considers that memory is not only needed for representing the raw request but also as a structured object which is reflected by default overhead. 传输中请求断路器允许 Elasticsearch 限制传输或 HTTP 级别上所有当前处于活动状态的传入请求的内存使用量，使其不超过节点上的特定内存量。内存使用量基于请求本身的内容长度。此断路器还认为，内存不仅需要表示原始请求，而且还需要作为结构化对象，默认开销会反映出来。

- `network.breaker.inflight_requests.limit`
  - (Dynamic) Limit for in-flight requests breaker, defaults to 100% of JVM heap. This means that it is bound by the limit configured for the parent circuit breaker.
  - （动态）对于正在处理的请求的断路器限制，默认为 JVM 堆的 100%。这意味着它受到父断路器配置限制的约束。
- `network.breaker.inflight_requests.overhead`
  - (Dynamic) A constant that all in-flight requests estimations are multiplied with to determine a final estimation. Defaults to 2.
  - （动态）一个常量，将所有正在进行的请求估计值相乘以确定最终估计值。默认值为 2。

## **Accounting requests circuit breaker 请求内存计算断路器**

The accounting circuit breaker allows Elasticsearch to limit the memory usage of things held in memory that are not released when a request is completed. This includes things like the Lucene segment memory. 请求内存计算断路器允许 Elasticsearch 限制内存中保留的对象的内存使用量，这些对象在请求完成时未被释放。这包括如 Lucene 段内存之类的内容。

- `indices.breaker.accounting.limit`
  - (Dynamic) Limit for accounting breaker, defaults to 100% of JVM heap. This means that it is bound by the limit configured for the parent circuit breaker.
  - （动态）请求内存计算断路器的限制，默认为 JVM 堆的 100%。这意味着它受为父断路器配置的限制的约束。
- `indices.breaker.accounting.overhead`
  - (Dynamic) A constant that all accounting estimations are multiplied with to determine a final estimation. Defaults to 1.
  - （动态）一个常数，所有的内存使用估计都会与之相乘，以得出最终的估计值。默认值为 1。

## **Script compilation circuit breaker 脚本编译断路器**

Slightly different than the previous memory-based circuit breaker, the script compilation circuit breaker limits the number of inline script compilations within a period of time. 与之前基于内存的断路器略有不同，脚本编译断路器限制了一段时间内内联脚本编译的次数。

See the "prefer-parameters" section of the scripting documentation for more information. 有关详细信息，请参阅脚本文档的“prefer-parameters”部分。

- `script.max_compilations_rate`
  - (Dynamic) Limit for the number of unique dynamic scripts within a certain interval that are allowed to be compiled. Defaults to 150/5m, meaning 150 every 5 minutes.
  - （动态）限制特定时间间隔内允许编译的唯一动态脚本的数量。默认为 150/5m，表示每 5 分钟 150 次。

## **Regex circuit breaker 正则表达式断路器**

Poorly written regular expressions can degrade cluster stability and performance. The regex circuit breaker limits the use and complexity of regex in Painless scripts. 编写不当的正则表达式会降低集群的稳定性和性能。正则表达式断路器限制了 Painless 脚本中正则表达式的使用和复杂性。

- `script.painless.regex.enabled`
  - (Static) Enables regex in Painless scripts. Accepts:
    - `limited` (Default): Enables regex but limits complexity using the `script.painless.regex.limit-factor` cluster setting.
      - 启用正则表达式，但使用 `script.painless.regex.limit-factor` 集群设置限制复杂性。
    - `true`: Enables regex with no complexity limits. Disables the regex circuit breaker.
      - 启用没有复杂性限制的正则表达式。禁用正则表达式断路器。
    - `false`: Disables regex. Any Painless script containing a regular expression returns an error.
      - 禁用正则表达式。任何包含正则表达式的 Painless 脚本都会返回错误。
- `script.painless.regex.limit-factor`
  - (Static) Limits the number of characters a regular expression in a Painless script can consider. Elasticsearch calculates this limit by multiplying the setting value by the script input’s character length.
  - （静态）限制 Painless 脚本中的正则表达式可以匹配的字符数。Elasticsearch 通过将设置值乘以脚本输入的字符长度来计算此限制。

例如，输入 `foobarbaz` 的字符长度为 9。如果 `script.painless.regex.limit-factor` 为 6，则 `foobarbaz` 上的正则表达式最多可以匹配 54（9 × 6）个字符。如果表达式超过此限制，它将触发正则表达式断路器并返回错误。

Elasticsearch only applies this limit if `script.painless.regex.enabled` is `limited`. 仅当 `script.painless.regex.enabled` 为 `limited` 时，Elasticsearch 才会应用此限制。