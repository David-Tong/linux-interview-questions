# 模拟 OA 02：Linux / Kubernetes SA

原创练习，非 Optiver 真题。90 分钟，100 分；假设 Linux、Bash、Python 3 与支持 EndpointSlice 的 Kubernetes。全部为合成案例，无需真实环境。A/B/C/D 建议用时 15/25/30/20 分钟。做完再看 [答案与评分](answers-02.md)。

## A. 单选：每题 2 分，共 24 分

**A01**：通过环境变量引用 ConfigMap 的容器，在 ConfigMap 修改后通常需要什么才能使用新值？

A. 重新创建/启动相应容器，常通过 rollout；B. 修改 Service；C. 无需任何动作即热更新环境变量；D. 等待 DNS TTL。

**A02**：init container 一直失败，业务容器尚未启动，应先看？

A. init 容器状态、事件与对应日志；B. 业务容器最近请求日志；C. Ingress 证书；D. HPA CPU。

**A03**：TCP 大量 TIME_WAIT 最准确的含义？

A. 是连接关闭的正常状态之一，需结合连接率、端口和资源压力判断；B. 一定由服务端产生；C. 一定泄漏；D. 必须全局关闭。

**A04**：Service 中有两个后端，其中一个错误率很高。哪种验证更有帮助？

A. 只看聚合平均值；B. 立即删除所有 Pod；C. 分后端比较请求与指标，核对故障是否集中于某实例；D. 只查看 Deployment 名称。

**A05**：同一目的 IP 有匹配的 /16 和 /24 路由，其他条件正常。优先选择？

A. /16；B. 最早添加的；C. /24，最长前缀匹配；D. 随机。

**A06**：PDB 能保证什么？

A. 节点断电时副本不减少；B. 代替多副本；C. 任意删除 Pod 都会被阻止；D. 对遵循 eviction API 的自愿驱逐施加可用性约束。

**A07**：Pod 的多个普通容器通常共享什么？

A. 相同环境变量；B. Pod 网络命名空间和端口空间；C. 相同根文件系统；D. 必然相同 PID 命名空间。

**A08**：startup probe 尚未成功时，配置的 liveness/readiness probe 通常如何？

A. 删除 Pod；B. 仍按原计划抢先执行；C. 修改 Service；D. 等待 startup probe 成功后再开始。

**A09**：SIGTERM 与 SIGKILL 的区别是？

A. SIGTERM 可被处理，SIGKILL 不可被捕获或忽略；B. SIGKILL 保证立即完成磁盘写入；C. 二者均能被捕获；D. SIGTERM 必然立即退出。

**A10**：测试 HTTPS 时要强制连接指定 IP，同时保持原域名的 SNI 与证书验证，最合适的是？

A. 只改成 `https://192.0.2.10/`；B. `curl --resolve api.example:443:192.0.2.10 https://api.example/`；C. 永久加 `-k`；D. 改成 ping。

**A11**：Pod Pending，事件为 `untolerated taint`，首先应？

A. 提高 memory limit；B. 核对目标节点用途、taint 与 Pod toleration 是否符合部署意图；C. 修改 DNS；D. 删除所有 taint。

**A12**：`vmstat 1` 的后续采样持续出现较高 si/so，应用延迟同步恶化。最合理的下一步？

A. 结合内存压力、进程用量与工作集检查换页影响；B. 立即删 swap 文件；C. 证明网卡损坏；D. 只调大磁盘分区。

## B. 故障诊断：每题 10 分，共 30 分

每题写出首要假设、至少三个命令及预期证据、替代解释、受控缓解和恢复验证。不能只列命令。

### B01：平均 CPU 低，行情处理仍堆积

16 核节点 CPU 平均利用率 8%，某进程一个工作线程接近 100% 单核。输入速率未变，队列长度持续增长；新版本增加了解析逻辑，存储延迟未明显改变。如何区分单线程计算瓶颈、锁等待及线程被限制到少数核心？为什么直接增加 Pod memory limit 没有证据支持？

### B02：容量够，但无法创建文件

`/data` 块使用率 20%，inode 使用率 100%。任务每秒产生数千个不足 1KiB 的临时文件，正常应在十分钟后删除；清理任务两天未成功。业务有保留最近一小时文件的要求。如何确定主要目录、恢复写入并防止复发？说明直接运行 `rm -rf /data/*` 的问题。

### B03：仅一台机器访问失败

同子网两台客户端访问 `api.example:443`，A 正常，B 超时。B 解析为旧 IP，A 解析为新 IP；新 IP 上证书与域名匹配。如何区分 `/etc/hosts`、本地解析缓存、递归 DNS 缓存及应用自带缓存？如何绕开解析做受控验证，同时保留 SNI 和证书校验？

## C. 编码：并发连接峰值，26 分

用 Python 3 标准库读取每行 `service start_ms end_ms`。

- service 匹配 ASCII `[a-z][a-z0-9-]{0,31}`，时间为 1～12 位 ASCII 非负十进制整数，start_ms < end_ms。
- 非法行、空行忽略；重复行表示独立连接。最多 200,000 行。
- 连接活跃区间为 `[start_ms, end_ms)`；同一时刻结束的连接不与该时刻开始的连接重叠。
- 每个 service 输出 `service peak earliest_ms`，其中 earliest_ms 为首次达到最大并发的时刻。
- service 字典序输出；空结果不输出。给出复杂度与边界测试。

输入：

```text
quotes 0 10
quotes 5 10
quotes 10 20
gateway 3 4
quotes 12 15
bad 9 9
```

输出：

```text
gateway 1 3
quotes 2 5
```

## D. 滚动发布与探针修复：20 分

命名空间 trading 中已有两副本 Deployment，下面为待发布的关键字段。镜像可用、资源足够。应用监听 8080，冷启动最久 75 秒，期间三个端点均返回 503；启动后 `/startup` 永久返回 200，`/live` 只反映本进程是否可工作，`/ready` 反映能否接新请求。所有路径由应用真实实现。

```yaml
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 0
  template:
    spec:
      containers:
        - name: quotes
          image: registry.example/quotes:v2
          livenessProbe:
            httpGet: {path: /ready, port: 8080}
            periodSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet: {path: /live, port: 8080}
            periodSeconds: 5
```

发布目标：滚动更新过程至少保留两个可用副本；给冷启动留出大于 75 秒的探测预算；依赖短暂故障只停止接流量，不触发 liveness 重启。写出修复后的完整关键字段，解释所选值，并写发布验证和 GitOps 回滚步骤。本题假设没有节点故障；说明该发布策略不能保证哪些故障场景。
