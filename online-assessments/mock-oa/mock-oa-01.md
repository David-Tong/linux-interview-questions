# 模拟 OA 01：Linux / Kubernetes SA

90 分钟，100 分。时间与评分是自定练习规则，不代表 Optiver。假设使用 Linux、Bash、Python 3 与支持 EndpointSlice 的 Kubernetes；所有环境输出均为合成案例。

## A. 单选题：每题 2 分，共 24 分

**A01**：16 核服务器 load average 为 35，CPU idle 70%，大量线程处于 D 状态。下一步最合理的是？

A. 增加 CPU；B. 查看 I/O 延迟、阻塞任务和内核日志；C. 提高 nice 值；D. 直接重启。

**A02**：`df -h /var` 显示 100%，`du -xsh /var` 远小于已用空间。哪项最能直接检验“已删除但仍打开的文件”假设？

A. `free -m`；B. `lsof +L1`；C. `ip route`；D. `chmod -R 777 /var`。

**A03**：主机有大量 page cache，但 `MemAvailable` 足够且无持续换页或内存 PSI 压力。最合理的判断是？

A. 必须清缓存；B. 已经 OOM；C. 仅凭 used 高不能判定内存不足；D. 禁用所有文件缓存。

**A04**：一次 TCP 连接抓包只有反复 SYN，没有 SYN-ACK。最准确的结论是？

A. 一定是 DNS；B. 一定是服务进程挂了；C. 应继续定位路由、防火墙、返回路径与监听，现有证据不能唯一归因；D. TLS 证书过期。

**A05**：进程退出后保留为 zombie，谁通常负责回收其退出状态？

A. 父进程通过 wait 系列调用；B. 给 zombie 发 SIGKILL；C. DNS 服务；D. 文件系统。

**A06**：Pod CPU request 为 500m，limit 为 1000m。节点 CPU 空闲，但容器持续撞到 CPU 配额。最可能的现象是？

A. CPU 超限必然 OOMKilled；B. 可能被 throttling，延迟增加；C. 自动提高 limit；D. request 保证线程独占物理核心。

**A07**：三个节点各剩余 1 CPU 可调度，Pod request 为 2 CPU，无其他约束。结果最可能是？

A. 跨节点凑出 2 CPU；B. Pending / Insufficient cpu；C. request 自动降为 1；D. 必定 OOM。

**A08**：Service selector 是 `app: quotes`，所有业务 Pod 标签都是 `app: quote`。最直接的问题是？

A. Service 不会选中这些 Pod；B. Pod 无法调度；C. kubelet 无法启动；D. 镜像拉取失败。

**A09**：readiness probe 连续失败、其他配置正常时，通常会发生什么？

A. 必定重启容器；B. 标为未就绪并从常规 Service 就绪后端中移除；C. 删除 Deployment；D. 增加内存。

**A10**：已有选中业务 Pod 的默认拒绝 egress 策略；现在业务 DNS 超时，其他条件不变。优先检查什么？

A. 是否允许到实际 DNS 目的地址的 UDP/TCP 53，并检查 CNI 路径；B. 删除 RBAC；C. 提高磁盘容量；D. 改 liveness。

**A11**：ArgoCD 开启自动同步和 self-heal 后，你手动修改受管理 Deployment，配置很快恢复。最合理的解释是？

A. Git 期望状态被重新应用；B. 节点时钟一定坏了；C. etcd 无法写入；D. 镜像缓存过期。

**A12**：三个 etcd 成员中一个故障，另两个健康且互通。通常还有多少故障余量？

A. 还能任意失去一个成员且继续多数派写入；B. 已失去多数派；C. 当前有多数派，再失去一个将失去多数派；D. 与成员数量无关。

## B. 故障诊断：每题 10 分，共 30 分

每题写：首要假设、至少 3 个命令及预期证据、其他可能原因、最小范围缓解、恢复验证。允许证据不足时明确说明。

### B01：磁盘满，但目录不大

`/var` 使用率 100%，inode 使用率 8%；`du -xsh /var` 为 12G，分区已用 95G。日志服务昨天轮转过，业务报 ENOSPC。你怎么确认并恢复？为什么删除更多路径可能没有帮助？

### B02：Pod 重启

业务容器 `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`，memory limit 512Mi。容器重启后查看节点，显示有 20Gi 可用内存；历史监控显示容器在终止前一分钟内存从 300Mi 增到接近 512Mi。

1. 已知运行时将这次终止记录为 OOMKilled，为什么节点有可用内存，容器仍可能发生 OOM？当前证据更支持哪种原因？如何进一步区分容器 cgroup 限制 OOM 与节点全局 OOM？重启后的节点内存读数能否代表故障当时？
2. 这次 OOM 能否证明应用内存泄漏？给出排查、受控缓解和恢复验证步骤。
3. **独立追问：假设另一条告警只有 `Exit Code: 137`，尚未取得 reason、事件或日志。** 此时能否确定 OOM？手工 `kill -9`、驱逐中的强制终止等应如何鉴别？不要将这个信息不足的假设与前面已给出 OOMKilled 的案例混为一谈。

### B03：跨节点失败

同节点 Pod IP 互通，跨节点 Pod IP 连接超时；Service 也间歇失败。两个节点均 Ready，部署 Cilium。最近更改了交换机 MTU。如何区分 MTU、底层路由、防火墙、CNI 和策略问题？先小包成功、大包失败是否足以确定根因？

## C. 编码：日志窗口聚合，26 分

实现 Python 3 程序，从 stdin 读取日志，向 stdout 输出结果；不得依赖第三方包。输入可能乱序。**离线处理即可**，无需实现实时流系统。

输入每行四列，以一个或多个空白分隔：`timestamp_ms service status latency_us`。

- timestamp_ms、latency_us 是非负十进制整数，字符仅为 ASCII 0–9。
- service 匹配 `[a-z][a-z0-9-]{0,31}`；status 为 100–599 的三位 ASCII 十进制数。
- 列数或字段不合法的整行忽略；空行忽略；不去重，重复日志是独立请求。
- 按 `(timestamp_ms // 60000, service)` 分组，窗口为左闭右开的 60 秒。
- 每个分组分别统计三个值：`count` 是有效请求总数；`5xx_count` 是其中 status 为 500～599 的请求数；`p99_us` 是这个分组所有有效请求的 p99 延迟，单位为微秒（包括成功请求和错误请求）。
- p99 使用 nearest rank 方法：先将组内所有 `latency_us` 从小到大排序，再计算位置 `rank = ceil(0.99 * count)`，最后取排序后的第 `rank` 个值。`ceil` 表示向上取整，位置从 **1** 开始计数；如果用 Python 列表读取，对应下标为 `rank - 1`。不进行插值，也不是把某个延迟乘以 0.99。
- 按窗口起始毫秒升序，再按 service 字典序输出；没有有效记录时不输出任何行。
- 输出格式：`window_start_ms service count 5xx_count p99_us`。不输出标题或调试日志。
- 最多 200,000 行，有效整数最多 15 位；允许保存全部有效延迟。

例如，一个分组有 5 条有效请求，排序后的延迟为 `10, 20, 30, 40, 100`，其中 2 条请求的 status 为 503，其余为 200：

```text
count     = 5
5xx_count = 2
rank      = ceil(0.99 × 5) = ceil(4.95) = 5
p99_us    = 第 5 个延迟 = 100
```

如果一个分组有 100 条请求，则取排序后的第 99 个延迟；有 200 条请求，则取第 198 个延迟。因此 p99 并不总是最大值。可以用整数公式 `rank = (99 * count + 99) // 100` 计算位置，避免浮点数舍入影响。

输入示例：

```text
60000 quotes 200 30
0 quotes 200 10
59999 quotes 503 90
1 gateway 500 40
bad row
60001 quotes 200 50
2 quotes 200 -1
```

期望输出：

```text
0 gateway 1 1 40
0 quotes 2 1 90
60000 quotes 2 0 50
```

提交程序，并说明复杂度。至少自行测试：空输入、全非法输入、59999/60000 边界、100 条延迟时的 p99、乱序、多服务、重复记录和 Unicode 数字。

## D. 配置排错：20 分

应用实际监听 `0.0.0.0:8080`，`GET /ready` 返回 200；镜像可用。以下资源在 `trading` namespace，未启用 NetworkPolicy。Git 仓库中的这份配置由开启 self-heal 的 ArgoCD 管理。忽略 namespace 与 image 的创建问题。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quotes
spec:
  replicas: 2
  selector:
    matchLabels:
      app: quotes
  template:
    metadata:
      labels:
        app: quotes
    spec:
      containers:
        - name: quotes
          image: example.invalid/quotes:v1
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8081
---
apiVersion: v1
kind: Service
metadata:
  name: quotes
spec:
  selector:
    app: quote
  ports:
    - port: 80
      targetPort: 8081
```

写出三处修正、完整修正后的关键字段、验证命令与成功条件。说明为什么仅修改 containerPort 无法修复进程监听端口；怎样通过 GitOps 发布，以及验证失败后怎样回滚。
