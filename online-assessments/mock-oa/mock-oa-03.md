# 模拟 OA 03：Linux / Kubernetes SA

原创练习，非 Optiver 真题。90 分钟，100 分；假设 Linux、Bash、Python 3 与支持 EndpointSlice 的 Kubernetes。全部为合成案例，无需真实环境。A/B/C/D 建议用时 15/25/30/20 分钟。做完再看 [答案与评分](answers-03.md)。

## A. 单选：每题 2 分，共 24 分

**A01**：8 核主机中，一个单线程工作进程持续占满一个核心，总 CPU 利用率约 12.5%。合理判断？

A. 必须换磁盘；B. 单线程可能已经受 CPU 限制，应结合每线程数据；C. 已证明内存泄漏；D. CPU 一定不是瓶颈。

**A02**：Linux 接口接收丢包计数增长，可以直接断言什么？

A. 存在该统计口径下的接收丢弃，仍需定位驱动、队列与协议栈等环节；B. 必然是公网拥塞；C. 必然是应用解析错误；D. 必然是 DNS。

**A03**：域名连接失败，直连已知 IP 成功。最合理的判断？

A. 优先检查解析结果及客户端解析路径，并确认两次请求协议和目标一致；B. 证明 TLS 正常；C. 必然是 DNS 服务宕机；D. 必然是磁盘。

**A04**：`df -h /data` 尚余 80%，`df -i /data` 为 100%，新建空文件报 ENOSPC。优先检查？

A. 小文件数量与 inode 占用；B. DNS TTL；C. TCP 重传；D. CPU 温度。

**A05**：大量 CLOSE_WAIT 持续增长，应重点检查？

A. 路由 metric；B. 应用收到对端关闭后是否及时关闭自己的 socket；C. 交换机时钟；D. DNS 缓存。

**A06**：写请求超时后，最准确的结论是？

A. 一定未执行；B. 一定已提交；C. 执行结果可能未知，需要查询结果或依靠幂等机制；D. 必须重复执行十次。

**A07**：API 返回 Forbidden，且身份认证已成功。优先检查？

A. MTU；B. 该主体对应 verb/resource/namespace 的授权；C. 磁盘碎片；D. swap。

**A08**：两个实例的请求量不同，能否直接平均各自 p99 得到整体 p99？

A. 取较小值即可；B. 不可以，需原始样本或可合并的分布信息；C. 只要单位相同即可；D. 始终可以。

**A09**：DNS UDP 查询可用，但 TCP 53 被阻断。哪项正确？

A. 必然所有查询立即失败；B. 只有 HTTP 受影响；C. 需要 TCP 的查询或截断回退可能失败；D. 对 DNS 永无影响。

**A10**：curl 返回 HTTP 503，可以直接确定什么？

A. 必然是 DNS 失败；B. 必然是内核 OOM；C. 必然未建立 TCP；D. 某 HTTP 响应端返回了 503，来源可能是代理或应用。

**A11**：Pod 报 ImagePullBackOff，优先查看？

A. HTTP Host；B. 应用 SQL；C. describe 中拉取事件、镜像地址、认证及仓库连通性；D. readiness URL。

**A12**：两个 NetworkPolicy 均选中同一 Pod 的 ingress，分别允许来源 X、Y。允许集合通常是？

A. 必须同时来自 X 和 Y；B. 两者允许规则的并集；C. 只看后创建的；D. 自动拒绝所有。

## B. 故障诊断：每题 10 分，共 30 分

每题给出假设、至少三个命令及预期证据、替代解释、最小范围缓解、恢复验证。

### B01：发布后不停重启

新版本正常冷启动需要约 70 秒。liveness 为 HTTP `/healthz`，initialDelaySeconds=5、periodSeconds=5、failureThreshold=3；未设置 startup probe。事件有 `Liveness probe failed` 和 `Killing`，无 OOMKilled。如何证明启动期被误杀？如何设计 startup、readiness、liveness 的职责和时间预算？还需排除哪些问题？

### B02：连接建立快，响应很慢

过去五分钟 HTTP p99 从 20ms 升至 2s。客户端采样显示 DNS、TCP、TLS 阶段耗时基本不变，首字节等待显著增长。反向代理 upstream 耗时同步增长，应用 CPU 30%。如何继续区分线程池排队、下游数据库、锁竞争和存储阻塞？为什么 CPU 不高不能排除服务端瓶颈？

### B03：发布时才出现连接复位

平时错误率正常，每次滚动发布会出现少量连接复位。应用收到 SIGTERM 直接退出，最长请求持续 20 秒，terminationGracePeriodSeconds=10。readiness 与 Service 后端传播存在短暂延迟。如何观察终止时序、区分应用主动退出与超时后强杀？怎样安排停止接流量、排空和退出预算？固定 sleep 是否足以保证零错误？

## C. 编码：重建 Pod 状态，26 分

每行 `timestamp sequence pod state`；用 Python 3 标准库离线处理。

- timestamp、sequence 是 1～12 位 ASCII 非负十进制整数；pod 匹配 `[a-z][a-z0-9-]{0,31}`，state 只能为 `Pending`、`Running`、`Failed`、`Deleted`。
- 非法行、空行忽略；最多 200,000 行。pod 名在本题中是唯一实体标识，不考虑 UID 和重建复用。
- 每个 pod 取 `(timestamp, sequence)` 数值字典序最大的事件作为当前状态；键相同则取输入中最后出现的一条有效事件。
- 最终状态为 Deleted 的 pod 不输出；其余按 pod 字典序输出 `pod state timestamp sequence`。
- 给出复杂度并测试乱序、同时间不同序号、完全同键、删除、空输入及 Unicode 数字。

输入：

```text
10 1 quotes Running
9 99 quotes Pending
10 2 quotes Failed
10 2 quotes Running
11 1 gateway Deleted
3 1 gateway Running
4 1 worker Pending
```

输出：

```text
quotes Running 10 2
worker Pending 4 1
```

## D. 容量与滚动发布推演：20 分

一个 Deployment replicas=3，每 Pod request 2 CPU、4Gi；三个节点 allocatable 各为 4 CPU、8Gi。其他固定工作负载在每个节点分别请求 2 CPU、4Gi，且不能移动或缩容。当前业务的三个 Pod 各占一个节点。没有其他亲和性、taint、配额或卷约束。

发布设置 `maxUnavailable: 0`、`maxSurge: 1`，要求整个发布期间至少三个业务 Pod 可用。新旧版本 requests 相同。所有调度判断按 requests 计算；不能超售这些 requests。终止中的 Pod 在释放资源前仍占用容量。

1. 计算当前各节点、整个集群的剩余可调度资源，解释为何发布可能停住，而不是直接认定控制器故障。（5 分）
2. 说明把 maxSurge 改成 2、把 maxUnavailable 改成 1，分别是否解决问题，是否满足目标。（5 分）
3. 给出一种满足目标的容量方案，计算至少需要的新增可调度资源和放置条件；说明为什么仅增加分散的小碎片资源可能不够。（5 分）
4. 写出验证命令、发布前提、失败时停止/回滚标准。为什么修改 requests 以“骗过调度器”不能替代实际容量规划？（5 分）
