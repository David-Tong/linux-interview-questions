# OA 03 答案与评分

先完成 [试卷](mock-oa-03.md)。评分仅供自测。

## A：24 分

每题 2 分：**B A A A B C B B C D C B**。

1. 平均 CPU 可掩盖单线程瓶颈。
2. 丢弃计数是线索，统计层次与具体驱动语义需要核对。
3. DNS 与实际连接目标需分层验证。
4. 块与 inode 是不同资源；小文件可耗尽 inode。
5. CLOSE_WAIT 关注本端关闭。
6. 超时表示未在预算内收到结果，不证明服务端未执行。
7. Forbidden 看授权。
8. 分位数不能直接平均得到全局分位数。
9. DNS 也需要 TCP。
10. 503 可能由代理生成。
11. 拉取错误看事件。
12. NetworkPolicy 允许规则可叠加。

## B：30 分

每题：假设 2，命令与预期证据 3，替代解释 1，缓解 2，验证 2。

以下命令中的地址、PID 和接口均需替换。curl 示例用 `--noproxy '*'` 排除环境代理干扰，适用于题目中的直接访问验证；若真实业务必须经代理，应另行验证代理路径。

### B01：发布后不停重启

**首要判断。** 应用正常要 70 秒才能完成冷启动，但 liveness 从启动后约 5 秒就探测，连续失败三次便可能触发重启。它可能在约 15 秒附近就被终止，始终无法完成 70 秒启动。实际时序受探测执行、超时、kubelet 和终止宽限期影响，不能把这个估算当作精确承诺。

**第一步：建立“启动 → 探测失败 → 被杀”的时间线。**

```bash
kubectl -n trading describe pod <pod>
kubectl -n trading get pod <pod> -o yaml
kubectl -n trading logs <pod> -c <container> --previous --timestamps
kubectl -n trading get events --field-selector involvedObject.name=<pod> --sort-by=.metadata.creationTimestamp
```

记录容器 startedAt/finishedAt、restartCount、lastState、探测失败与 Killing 事件，和应用“开始初始化/完成初始化”日志对齐。若每次在相近短时间被探针终止、从未到启动完成点，且旧版本或独立测试约 70 秒可完成，则支持启动期误杀。事件可能被聚合或过期，不能只靠一条事件精确计算全部时间。

无 OOMKilled 降低 OOM 优先级，但还需排除：端口/路径配置错、应用只监听 loopback、初始化卡死、CPU throttling 导致启动更慢、真正崩溃。即使最终退出码为 0，若应用是收到探针触发终止的 SIGTERM 后正常退出，仍可能是探针配置问题。

**第二步：分配三个探针的职责。** 假设应用真实提供以下语义端点，可配置：

```yaml
startupProbe:
  httpGet: {path: /startup, port: 8080}
  periodSeconds: 5
  timeoutSeconds: 1
  failureThreshold: 24
readinessProbe:
  httpGet: {path: /ready, port: 8080}
  periodSeconds: 5
  timeoutSeconds: 1
  failureThreshold: 1
livenessProbe:
  httpGet: {path: /live, port: 8080}
  periodSeconds: 5
  timeoutSeconds: 1
  failureThreshold: 3
```

startup 的名义预算约 120 秒，大于 70 秒且留余量；未成功前 readiness/liveness 不开始执行。readiness 反映是否可接新请求，失败使普通 Service 就绪后端不再选它；liveness 仅检测值得通过重启恢复的进程故障，不把短暂数据库不可用当作必须重启。路径不能凭空新增：若现有应用只有 `/healthz`，要先确认它的真实语义或配套开发端点，再修改配置。

**第三步：受控恢复与反向验证。** 若已影响线上，先回退已知良好版本，或在少量实例上修正探针再推进。验证至少一次完整冷启动超过旧的失败时刻、最终 Ready、重启次数稳定、业务正常；在测试环境模拟依赖短暂失败，预期 readiness 下降而非所有实例重启；模拟进程真实失活，确认 liveness 能发现。单纯把 liveness 延迟设成极大值会延后发现运行期故障，不能替代职责设计。探针机制参见 [官方容器探针说明](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes)。

### B02：连接建立快，响应很慢

**首要判断。** DNS/TCP/TLS 阶段未变、代理 upstream 耗时增加，优先查代理到应用及其下游处理路径。首字节等待包含排队、处理与返回首字节的时间，不能直接把它全部叫“数据库耗时”。CPU 30% 也可能掩盖单核热点，或者大量线程正在等待。

**第一步：将时间分解并对齐请求。**

```bash
curl --noproxy '*' -sS -o /dev/null --max-time 10 \
  -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n' \
  https://api.example/<safe-path>
```

这些值主要是从请求开始累计的时间，TCP 阶段可近似看 connect−namelookup，TLS 看 appconnect−connect；连接复用、重定向和代理会改变解释。TTFB 减 appconnect 也只是近似的 TLS 后等待，仍含请求传输等时间。对多次请求、多个实例采样，按 request ID 关联代理与应用 trace；不要把不同请求的 p99 数字直接相减当成某阶段 p99。

**第二步：从排队到下游逐层定位。**

```bash
pidstat -u -w -d -p <pid> 1
top -H -p <pid>
iostat -xz 1
sudo ss -tanp
```

配合应用指标观察线程池 active/max、队列长度、队列等待、数据库池 active/idle/wait、请求处理跨度。系统命令提供资源侧旁证，不能代替应用队列指标。

| 方向 | 预期证据 | 区分方法 |
| --- | --- | --- |
| 线程池排队 | worker 已满、队列等待增长 | 看 worker 实际在执行还是等下游；排队可能是后果 |
| 数据库或连接池 | trace 的数据库跨度变长、池等待或慢查询/锁等待增加 | 分清“等借连接”与“已发 SQL 等结果” |
| 应用锁竞争 | 多次线程栈停在同一锁，持锁线程或热点吻合 | 等锁不一定吃 CPU，自旋锁则可能吃 CPU |
| 存储阻塞 | 线程 I/O 等待、磁盘延迟/队列与请求延迟同步 | 关联具体设备和访问，不能仅凭磁盘 %util 判满 |
| 单线程或局部实例瓶颈 | 某线程/实例饱和，聚合 CPU 却低 | 按线程、实例及请求类型拆分 |

同时排查代理队列、请求体/响应变大、热点键和少数慢实例。必要时对一个实例短时取线程栈或 profiler，先评估开销。

**缓解与验证。** 若数据库已满，盲目扩线程或连接池会把更多请求压到数据库；更合理的是限并发、限流、隔离慢接口或回滚。若只有一个坏实例，可先移出接流量范围再诊断；若是锁或代码退化则回滚对应版本。恢复标准包括线程/连接池等待与队列回落、相近负载下 p99 和成功率恢复、下游无新增压力，并验证积压没有继续增长。

### B03：发布时才出现连接复位

**首要判断。** 应用收到 SIGTERM 立即退出，会切断在途请求；最长请求 20 秒而 grace 只有 10 秒，则即使应用愿意等待也可能没有足够预算。readiness/后端传播又不是瞬间完成，开始终止后仍可能短暂收到流量。要分清“应用主动过早退出”和“宽限期耗尽被强杀”，两者修复点不同。

**第一步：收集同一 Pod 的完整终止时序。**

```bash
kubectl -n trading get pod <pod> -o yaml
kubectl -n trading describe pod <pod>
kubectl -n trading logs <pod> -c <container> --timestamps --since=15m
kubectl -n trading get endpointslices -l kubernetes.io/service-name=<service> -o yaml
kubectl -n trading get pods -w
```

在发布前开启日志集中采集，避免 Pod 删除后证据丢失；`logs --previous` 主要针对同 Pod 的上次容器，不保证能找回已删除旧 Pod。记录 deletionTimestamp、preStop 起止、SIGTERM 到达、readiness 改变、最后新请求、在途计数归零、进程退出、EndpointSlice ready/terminating 条件及代理摘流量时间。跨节点时间线也要核对时钟偏差。

| 观察到的时间线 | 判断方向 |
| --- | --- |
| SIGTERM 后立即退出，仍有在途请求，未到 grace 上限 | 应用主动终止过早 |
| 应用持续等待，到 grace 附近被杀，在途仍未归零 | 预算不足、请求无界或 drain 未完成 |
| 已停止接收新连接，但旧 keep-alive 仍发新请求 | 连接级排空协议不完整 |
| EndpointSlice 已变化，代理仍向旧实例发请求 | 传播、代理缓存或长连接行为需检查 |

137 不是强杀原因的唯一证据，需要事件、内核记录及时间相关性排除 OOM 等情况。还要确认容器 PID 1 能收到并转发信号，shell 包装脚本吞信号也会导致终止超时。

**第二步：设计排空状态机。** 进入 drain 后先标记不接新业务并触发 readiness 失败，让流量层停止选中它；在实测传播窗口内仍应有策略处理已经到达或在连接中排队的请求，不能一标记就粗暴退出。然后停止新请求进入、完成允许的在途请求，处理 HTTP keep-alive/HTTP2 GOAWAY 或其他协议的关闭语义，关闭资源并退出。readiness 失败不会自动替应用完成连接排空。

可让 preStop 调应用的 drain 接口，或让 SIGTERM 处理器实现配套逻辑。preStop 的时间计入整体宽限期，常规流程是 hook 结束后才发送 TERM，因此“hook sleep 20 秒再给应用 20 秒、总 grace 仍 20 秒”不成立。设计示例：实测传播上界约 5 秒，随后允许最长 20 秒在途请求，再留 10 秒清理余量，可先以 35 秒或更大 grace 测试；这只是基于这些假设的预算，不是通用保证。具体生命周期机制见 [官方 Pod 终止流程](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)。

**第三步：验证与回滚。** 在测试环境持续发短请求、接近 20 秒的请求和长连接，反复滚动更新，观察终止实例在途请求是否归零、是否正常退出、客户端复位是否消失、代理是否继续发新请求。还要测试慢于预算的请求有明确超时/错误策略，不能无限等待。线上从少量实例推进，错误率或在途截断恶化就暂停发布并回退相关版本/配置。固定 sleep 只能覆盖一个估计窗口，不能保证所有流量层传播、长连接和请求长度，因此不能单独承诺零错误。

## C：26 分

校验 5，比较键与乱序 7，同键最后事件 4，删除语义 4，输出/复杂度/测试 6。

```python
import re
import sys

num_re = re.compile(r"[0-9]{1,12}")
pod_re = re.compile(r"[a-z][a-z0-9-]{0,31}")
states = {"Pending", "Running", "Failed", "Deleted"}
latest = {}
for line in sys.stdin:
    f = line.split()
    if len(f) != 4:
        continue
    ts, seq, pod, state = f
    if not (num_re.fullmatch(ts) and num_re.fullmatch(seq)
            and pod_re.fullmatch(pod) and state in states):
        continue
    key = (int(ts), int(seq))
    if pod not in latest or key >= latest[pod][0]:
        latest[pod] = (key, state)

for pod in sorted(latest):
    (ts, seq), state = latest[pod]
    if state != "Deleted":
        print(pod, state, ts, seq)
```

必须保存 Deleted 墓碑，不能立即删除字典项，否则后读到旧事件会错误复活。比较整数，不能比较时间字符串。n 行、p 个有效 pod，字段长度有界，时间 O(n + p log p)，空间 O(p)。测试旧 Running 在新 Deleted 后到达仍不输出；同键 Deleted 后接 Running 则输出 Running；前导零按数值比较；非法行不能覆盖有效状态。

## D：20 分

1. 每节点固定负载 2 CPU/4Gi，加业务 2 CPU/4Gi，剩余均为 0；全局剩余也为 0。surge 新 Pod 无处调度，而 maxUnavailable=0 不允许控制器先缩旧副本，发布卡住有充分的容量解释。（5 分）
2. maxSurge=2 允许更多额外 Pod，却不创造资源，仍卡住。maxUnavailable=1 允许先缩掉一个旧副本并释放资源，有机会推进，但会允许仅两个业务 Pod 可用，违反目标。（5 分）
3. 至少增加能在一个符合约束的节点上放下一个 2 CPU/4Gi Pod 的可调度容量。例如新增 allocatable 至少 2 CPU/4Gi、且这部分未被其他 requests 占用的节点；实际规划还需系统开销与余量。新增 Pod Ready 后再缩一个旧 Pod，等待其资源释放后继续迭代。把 2 CPU/4Gi 分散为两个各仅余 1 CPU/2Gi 的节点不能调度单个新 Pod。（5 分）
4. `kubectl describe node` 看 allocatable/requests；`kubectl describe pod <pending>` 确认 FailedScheduling；`kubectl get deployment <name> -o yaml` 核对策略和副本；`kubectl get pods -o wide`、`kubectl rollout status deployment/<name> --timeout=300s` 观察进度，结合可用副本、EndpointSlice、错误率和延迟。容量及镜像验证通过后再发布。新 Pod 无法调度、探针失败、可用副本不达标或业务指标恶化时停止推进，按 GitOps 流程回退并核对状态。虚减 requests 会改变调度和资源竞争保障，不创造真实算力与内存。（5 分）
