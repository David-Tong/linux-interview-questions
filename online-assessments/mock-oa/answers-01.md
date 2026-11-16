# 模拟 OA 答案与评分

先完成 [试卷](mock-oa.md)。评分仅用于自测，没有 Optiver 官方通过线。

## A：24 分

| 题号 | 答案 | 解释 |
| --- | --- | --- |
| A01 | B | load 包括可运行任务和不可中断睡眠任务；CPU idle 高时不能直接归因 CPU 不足。 |
| A02 | B | 查 link count 为零但仍被进程持有的文件；还需确认它位于目标文件系统。 |
| A03 | C | 看可用内存、换页与压力趋势，不能只看 used。 |
| A04 | C | SYN 无响应不能区分路径丢包、过滤或返回路径等原因。 |
| A05 | A | 父进程 wait；父进程退出后由接管它的 reaper 回收。 |
| A06 | B | CPU 配额可能限制吞吐并放大尾延迟，不以 OOM 方式处理。 |
| A07 | B | 单个 Pod 的请求不能跨节点拼接。 |
| A08 | A | selector 必须匹配 Pod 标签。 |
| A09 | B | readiness 控制就绪状态；liveness 才可触发容器重启。 |
| A10 | A | DNS 需要考虑 UDP 和 TCP，以及 NodeLocal DNS 等实际部署。 |
| A11 | A | self-heal 可将受管理资源恢复到 Git 期望状态。 |
| A12 | C | 3 个成员的多数派是 2；当前没有进一步的成员故障余量。 |

## B：每题 10 分

通用评分：假设及证据链 2 分，命令与预期观察 3 分，替代解释 1 分，范围受控的缓解 2 分，恢复验证 2 分。不按命令数量机械给分。

以下命令是 Linux / Kubernetes 环境的排查示例，并非要求在当前电脑执行。`<pid>`、`<pod>`、`<container>`、`<node>` 等均需替换；命名空间示例用 trading。时间窗口应换成事故实际时间，节点和 cgroup 检查需相应权限。

### B01：磁盘满，但目录不大

**判断与原理。** 优先怀疑轮转后旧日志被 unlink，但日志进程仍持有 fd。目录项消失后，du 遍历路径时看不到它；只要文件仍有打开引用，相关数据块就不能按普通删除流程回收，因此 df 仍统计为已用。95G 与 12G 的差额约 83G，是值得查找的量级，但不应要求不同工具的值精确相等。inode 仅用 8%，不支持 inode 耗尽。

**第一步：确认比较对象。**

```bash
findmnt -T /var
df -hT /var
df -i /var
sudo du -xsh /var
```

findmnt 确认 `/var` 属于哪个文件系统；df 看这个文件系统的总体使用量；`du -x` 不跨文件系统。必须使用足够权限并检查 du 的错误信息，否则无权限目录也会造成低估。如果 `/var` 只是根分区的一部分，df 的已用空间还包括该分区其他目录，不能拿 `/var` 的 du 与整分区 df 直接相减后认定泄漏。

**第二步：找持有旧文件的进程。**

```bash
sudo lsof +L1
sudo ls -l /proc/<pid>/fd
sudo stat -L /proc/<pid>/fd/<fd>
```

在 lsof 中找 link count 为 0、大小较大的普通文件，记录 COMMAND、PID、FD、DEVICE、SIZE/OFF 和路径，核对设备属于目标文件系统。`/proc/<pid>/fd/<fd>` 可能指向带 `(deleted)` 的旧日志；stat 跟随 fd 查看文件大小及实际分配块。稀疏文件的逻辑大小不等于磁盘占用，多个 fd 也可能引用同一个文件，不能直接把 lsof 各行大小相加。

若发现日志服务持有一个约 80G 的 deleted 文件，且时间与轮转吻合，证据链就较强。若没有，则继续查权限漏算、同一分区其他目录、被挂载覆盖的旧数据、快照及文件系统元数据；不要为了检查被覆盖目录直接卸载生产挂载点。

**第三步：最小范围恢复。** 先确认进程及服务身份、是否允许切流，再按该服务文档执行“重新打开日志”的操作。某些服务支持 reload 或指定信号，某些不支持；HUP 不是通用 reopen 命令。不支持时先移走业务流量，再受控重启持有文件的实例。只有相关打开引用都释放后，空间才有机会回收。不要直接清空未知 `/proc/.../fd`，也不要继续随机删除业务目录。

**验证与预防。** 再查 lsof，确认旧 inode 不再被持有；df 可用空间明显回升；做受控业务写入，确认 ENOSPC 消失；检查新日志路径持续有正确内容。下一次轮转也要验证 reopen 生效。完善轮转 postrotate 流程、空间告警和增长速率监控。只写“重启服务”缺少定位依据、影响控制和长期修复，不能拿满分。

### B02：Pod 重启，节点却有可用内存

**判断与原理。** 容器受自己的内存 cgroup 限制，节点尚有 20Gi 并不允许它越过 512Mi。OOMKilled 与接近 limit 的曲线共同支持容器限制相关 OOM；但“一次 OOM”不等于“程序泄漏”，137 也只是常见的 `128 + SIGKILL(9)` 表达，人工 kill、终止超时等同样可能出现。

**先按题目已知条件作答。** 本题已经明确给出 `Reason: OOMKilled`，应按“这次终止被运行时记录为 OOM”继续定位原因；不能因为 exitCode 同为 137，就把普通手工 kill 或驱逐强杀列为同等解释。单纯手工 `kill -9` 或普通驱逐强杀不会仅因退出码是 137，就被标记为 OOMKilled。若要质疑运行时记录，需要额外的冲突证据，本题没有提供。

主问题要区分的是“容器限制 OOM”与“节点全局 OOM”，以及是否存在泄漏。节点 20Gi 是重启后的快照，不能证明故障时也有 20Gi；即便故障时节点内存充足，容器仍受自身 limit 约束。当前内存曲线更支持容器限制方向，但需核对故障时证据。

**以下只回答独立追问：假设另一条告警仅给出 137，能否确定 OOM？不能。** 137 通常反映进程被 SIGKILL 结束，不记录发信号的原因。下面几种情况都可能出现 137；它们不是对前面已知 OOMKilled 案例的同等候选解释：

| 场景 | 为什么可能出现 137 | 还要看什么证据 |
| --- | --- | --- |
| 容器内存达到 cgroup 限制，触发 OOM kill | 内核 OOM killer 杀死容器主进程 | 容器 `reason: OOMKilled`、实际 limit、cgroup 与内核记录 |
| 节点全局内存不足，触发 OOM kill | 内核选择并杀死受影响的容器进程，即使它未达到自身 limit | 节点内存与内核 OOM 日志，区分全局 OOM 和 cgroup 限制 OOM |
| 手工执行 `kill -9 <pid>` | SIGKILL 直接结束容器主进程 | 操作审计、运行时记录和时间线；不能仅凭 137 标成 OOM |
| 节点压力触发 kubelet 驱逐 Pod | 若终止过程中容器主进程被 SIGKILL 杀死，可出现 137 | Pod 的 `status.reason: Evicted`、`status.message` 与驱逐事件 |
| 删除、发布或探针触发终止，进程在宽限期内未退出 | 终止流程最终强制杀死容器进程 | 终止时间线、探针事件、宽限期和应用退出日志 |

这里的手工 kill 指容器主进程；只杀一个子进程不一定导致容器退出。节点有压力也不意味着必然出现 137：进程可能正常响应终止信号，或尚未触发驱逐/OOM。节点全局 OOM 是内核杀进程，节点压力驱逐是 kubelet 终止 Pod，两者也要分开判断。关于节点压力驱逐及其终止宽限期，见 [官方说明](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)。

实际检查要区分两个层级：Pod 被驱逐时看 `status.phase: Failed`、`status.reason: Evicted` 和 message；容器 OOM 时看 `status.containerStatuses[].state.terminated.reason`，或重启后的 `lastState.terminated.reason` 是否为 `OOMKilled`。不要把 kubectl 列表中的 STATUS、Pod phase、容器 reason 和 exitCode 当成同一个字段。对本题而言，**支持 OOM 的关键是 OOMKilled 加上内存与限制证据，而不是单独的 137**。

**第一步：保留容器与 Pod 级证据。**

```bash
kubectl -n trading describe pod <pod>
kubectl -n trading get pod <pod> -o yaml
kubectl -n trading logs <pod> -c <container> --previous --timestamps
kubectl -n trading top pod <pod> --containers
```

逐个容器核对 lastState、reason、exitCode、finishedAt、restartCount 与实际 resources；不能把 sidecar 的状态当成业务容器。`--previous` 查上一次容器的日志，但应用被直接杀死时可能没有最后一条错误日志。top 是当前采样，重启后的低内存不能反证重启前没有峰值。还要区分容器 OOM 后在原 Pod 内重启与 Pod 因节点压力被 Evicted、控制器另建副本。

**第二步：区分限制 OOM 与节点压力。**

```bash
kubectl describe node <node>
sudo journalctl -k --since '-15 min'
cat /proc/<container-host-pid>/cgroup
```

看节点 MemoryPressure、内核 OOM 记录、受影响进程与时间。根据容器运行时和宿主机挂载布局找到实际 cgroup v2 目录，再读 `memory.max`、`memory.current`、`memory.events`；512Mi 对应 536870912 字节。关注 max、oom、oom_kill 计数的变化，并与日志时间关联。oom_kill 单独增加不能唯一证明是该 cgroup 的 limit 触发，需结合内核记录及其他计数；重建容器后旧 cgroup 也可能已不存在。v1 环境使用对应指标，不能硬套 v2 文件名。

| 假设 | 更有区分力的证据 |
| --- | --- |
| 容器限制不足 | 容器逼近 memory.max，相关限制/oom 证据吻合，节点压力不突出 |
| 节点压力或全局 OOM | 节点内存低、多个负载受影响、内核记录或驱逐事件支持 |
| 应用泄漏 | 相似负载下存活对象/内存基线持续增长，空闲或正常回收后仍不回落 |
| 合理工作集变大 | 并发、批次、缓存或输入规模增长可解释峰值，负载回落后恢复或稳定 |

**第三步：缓解并验证根因。** 先依据变更关联回滚、限流或减小批次/并发。有容量依据时可临时提高 limit，同时评估节点总需求与 request；单改 request 不会提高内存硬上限。再用运行时 heap profile、分配统计或对象数量检查泄漏，区分堆、native 内存、缓存和其他记账内容。

在相当或更高的负载下观察足够长时间：restartCount 不再增加，内存峰值有余量、基线不持续抬升、节点无新增压力、成功率与 p99 恢复。观察窗口应覆盖原先触发故障的周期；“重启后一分钟正常”不能证明泄漏修复。

### B03：同节点可通，跨节点失败

**假设已有以下业务资源**，都在 `trading` 命名空间，不需要创建任何资源：

| 资源 | 所在节点 | IP / 端口 |
| --- | --- | --- |
| client：发请求的 Pod | node-a | 10.244.1.10 |
| quotes-a：业务 Pod | node-a | 10.244.1.20:8080 |
| quotes-b：业务 Pod | node-b | 10.244.2.20:8080 |
| quotes：Service | 后端为 quotes-a、quotes-b | 10.96.0.100:80 → Pod:8080 |

两个业务 Pod 都提供 `/ready`。节点 IP 假设为 node-a `192.0.2.11`、node-b `192.0.2.12`，CNI 使用 Cilium。下面地址仅用于说明，实际排查时替换。假设 client 可用 curl 和 ping；节点可用 ip、tcpdump，相关命令需对应权限。

**1. 先确认 Pod 的 IP 和位置。**

```bash
kubectl -n trading get pods -o wide
```

含义：列出 Pod、IP、所在节点。确认 client 与 quotes-a 同节点，quotes-b 在另一节点。

**2. 从同一个 client 分别访问两个 Pod。**

```bash
# 同节点：client → quotes-a
kubectl -n trading exec client -- curl --noproxy '*' -v --connect-timeout 3 --max-time 5 http://10.244.1.20:8080/ready

# 跨节点：client → quotes-b
kubectl -n trading exec client -- curl --noproxy '*' -v --connect-timeout 3 --max-time 5 http://10.244.2.20:8080/ready
```

`exec client --` 表示在 client 容器里执行后面的命令；`-v` 显示连接过程；`--noproxy '*'` 不走代理；连接最多等 3 秒，整个请求最多 5 秒。

| 结果 | 怎么理解 |
| --- | --- |
| 两次都返回 HTTP 200 | 这两条 HTTP 路径当前正常 |
| 同节点 200、跨节点超时 | 复现题目现象，继续第 3 步 |
| 跨节点立即 `Connection refused` | 查目标是否监听 8080，或是否有主动拒绝规则 |
| 已返回 HTTP 404/503 | 已有 HTTP 响应，继续查路径、应用或代理，不能只说网络不通 |

**3. 检查节点间路由和接口 MTU。** 在 node-a 的终端执行，不是在本地电脑执行：

```bash
ip route get 192.0.2.12
ip -d link show
```

第一条看去 node-b 走哪个接口、网关和源地址；第二条看接口、隧道信息及 MTU。没有路由或走错接口就先修路由。节点之间能通不代表 Pod 网络一定能通，还要继续看 Cilium 的跨节点路径。

**4. 比较不同大小的包。** 从 client 向 quotes-b 探测：

```bash
kubectl -n trading exec client -- ping -4 -M do -s 1200 -c 3 10.244.2.20
kubectl -n trading exec client -- ping -4 -M do -s 1400 -c 3 10.244.2.20
```

`-4` 用 IPv4；`-M do` 要求不分片；`-s` 指 ICMP 数据大小；`-c 3` 发三次。IPv4 无选项时，还要加 28 字节 IP/ICMP 头，所以 1400 对应约 1428 字节的 IP 包。

小包成功、大包失败，或看到 `message too long`、`Frag needed`，就优先核对近期 MTU 修改和封装开销。若两种大小都超时，也可能只是 ICMP 被过滤，不能直接认定 MTU 错误。

**5. 在两个节点同时抓包，找包丢在哪里。**

在 node-a 执行：

```bash
sudo tcpdump -ni any -c 50 'host 10.244.2.20 and tcp port 8080'
```

在 node-b 执行：

```bash
sudo tcpdump -ni any -c 50 'host 10.244.1.10 and tcp port 8080'
```

含义：查看这两个 Pod 之间 TCP 8080 的流量，最多抓 50 个包。两端开始抓后，重新执行第 2 步的跨节点 curl；没抓满时按 Ctrl+C 停止。

| 抓到什么 | 下一步 |
| --- | --- |
| node-a 有 SYN，node-b 没看到 | 查节点间路由、封装、过滤；先确认没抓错位置 |
| node-b 收到 SYN、发出 SYN-ACK，node-a 没收到 | 查返回路径和过滤 |
| 收到 RST | 查目标监听端口和 REJECT 规则 |
| 握手完成，传输数据时反复重传 | 结合 MTU、丢包及应用状态继续定位 |

SYN 是连接请求，SYN-ACK 是服务端回应，RST 是复位。隧道封装时，物理网卡上可能只有节点 IP 间的外层包；如果 Pod IP 过滤器抓不到，再用下面的节点 IP 过滤器查看外层流量：

```bash
# node-a 上执行
sudo tcpdump -ni any -c 50 'host 192.0.2.12'
```

**6. 检查 Cilium 和网络策略。**

```bash
kubectl -n kube-system get pods -l k8s-app=cilium -o wide
kubectl -n kube-system logs <cilium-pod> --since=10m
kubectl -n trading get networkpolicy -o yaml
kubectl -n trading get ciliumnetworkpolicy -o yaml
kubectl get ciliumclusterwidenetworkpolicy -o yaml
```

第一条找到 node-a/node-b 对应的 Cilium Pod；第二条把 `<cilium-pod>` 换成实际名字，查看故障附近日志，两端都看。后三条看 Kubernetes 策略、Cilium 命名空间策略和集群策略，检查是否允许 client 到业务 Pod 的 TCP 8080。若某类策略资源不存在，API 会提示不识别资源类型，不代表网络本身坏了。日志中没报错也不能证明没有策略丢包。

**7. 单独检查 Service。**

```bash
kubectl -n trading exec client -- curl --noproxy '*' -v --connect-timeout 3 --max-time 5 http://10.96.0.100:80/ready
kubectl -n trading get svc quotes -o yaml
kubectl -n trading get endpointslices -l kubernetes.io/service-name=quotes -o yaml
```

第一条请求 Service IP；第二条核对 selector、80 → 8080 的端口映射；第三条核对实际后端 IP、端口及 ready 状态。

如果跨节点 Pod IP 已失败，Service 选到该后端时也可能失败，先修跨节点路径。若两个 Pod IP 都成功而 Service 失败，再查 Service 配置和转发。不要通过重复访问 Service 来代替固定 Pod IP 测试，因为每次可能选到不同后端。

**恢复与验证。** 证据指向 MTU 就小范围修正或回退 MTU 变更；指向路由、策略则只改对应配置。修复后重复同节点、跨节点、Service 请求，并检查实际业务成功率和重传是否恢复。Node Ready 不能证明跨节点 Pod 网络正常，小包成功也不能单独证明 MTU 无误。


## C：26 分

评分：字段校验 5，分组与边界 5，计数和 nearest-rank p99 6，排序及格式 4，边界测试 4，复杂度解释 2。

参考实现：

```python
import re
import sys

digits = re.compile(r"[0-9]{1,15}")
service_pattern = re.compile(r"[a-z][a-z0-9-]{0,31}")
status_pattern = re.compile(r"[1-5][0-9]{2}")


def summarize(lines):
    groups = {}
    for line in lines:
        fields = line.split()
        if len(fields) != 4:
            continue
        timestamp, service, status, latency = fields
        if not (
            digits.fullmatch(timestamp)
            and service_pattern.fullmatch(service)
            and status_pattern.fullmatch(status)
            and digits.fullmatch(latency)
        ):
            continue
        key = (int(timestamp) // 60000 * 60000, service)
        if key not in groups:
            groups[key] = [0, []]
        groups[key][0] += int(status) >= 500
        groups[key][1].append(int(latency))

    for (window, service), (errors, latencies) in sorted(groups.items()):
        latencies.sort()
        count = len(latencies)
        rank = (99 * count + 99) // 100
        yield f"{window} {service} {count} {errors} {latencies[rank - 1]}"


if __name__ == "__main__":
    for row in summarize(sys.stdin):
        print(row)
```

若 N 是有效记录数、G 是分组数、L 是输入字符数，时间为 O(L + N log N + G log G)，空间为 O(N + G)，另加读取单行空间。各组排序之和不超过 O(N log N)。

边界期望：空输入或全非法输入为空输出；59999 属于窗口 0，60000 属于窗口 60000；同组延迟 1–100 的 p99 为 99；重复记录计两次；全角数字如 `１２` 不合法。nearest rank 不是线性插值，也不是永远取最大值。

## D：20 分

修正三处字段各 3 分，解释 containerPort 2 分，验证 5 分，GitOps 与回滚 4 分。

```yaml
# Deployment.spec.template.spec.containers[0]
ports:
  - containerPort: 8080
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
---
# Service.spec
selector:
  app: quotes
ports:
  - port: 80
    targetPort: 8080
```

containerPort 声明不会修改程序实际监听地址和端口，也不会自动修正 probe 或 Service。这里程序已监听 8080，应让两条访问路径都指向它。

验证：

```bash
kubectl -n trading get pods -l app=quotes --show-labels
kubectl -n trading describe pod <pod>
kubectl -n trading get svc quotes -o yaml
kubectl -n trading get endpointslices -l kubernetes.io/service-name=quotes -o yaml
kubectl -n trading rollout status deployment/quotes
```

预期 Pod Ready、EndpointSlice 地址匹配业务 Pod、就绪标记正常且端口 8080。再从允许的集群客户端请求 `http://quotes.trading.svc.cluster.local/ready`，预期 200，并直接测试 Pod IP:8080 以区分应用与 Service 路径。已有工具的 Pod 可用于发请求；不要假设所有业务镜像自带 curl。

在 Git 修改、审查 diff、走既有检查与发布流程，再让 ArgoCD 同步。检查 Synced/Healthy 并验证业务；这两个状态不能替代业务探测。若失败，revert 到已知良好 Git 版本后重新同步并验证。仅 `kubectl edit` 或 `rollout undo` 可能被 self-heal 覆盖。
