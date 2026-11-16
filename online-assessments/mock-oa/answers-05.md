# OA 05 答案与评分

先完成 [试卷](mock-oa-05.md)。评分仅供自测。

## A：24 分

每题 2 分：**B D D A D C C C A B A C**。

1. rename 不改变已打开 fd 引用的文件对象。
2. 进程级 fd 限制应与系统级限制分开检查。
3. 采样和聚合会掩盖短时峰值。
4. Deployment 模板必须匹配 selector。
5. 默认管道状态取最后一个命令；pipefail 可暴露前级失败。
6. ready endpoint 不证明数据路径可达。
7. 用时间相关性与连续采样排查，不能靠单个容量数值。
8. 单调时钟适合持续时间，墙上时钟可能被校正。
9. RST 不等于应用亲自拒绝。
10. 0640 的 other 无权限。
11. 信号可能仍待进程从不可中断等待中返回后生效。
12. 执行还受挂载选项、ACL、安全模块等约束。

## B：30 分

每题：假设与因果链 2，命令/观测及证据 3，替代解释 1，受控缓解 2，恢复验证 2。

以下示例以 Linux、trading 命名空间为背景，PID、TID、Pod 名称和指标名需按实际系统替换。应用指标是需要采集的证据，不假设所有运行时都提供同一命令。

### B01：数据库恢复后，服务仍雪崩

**先解释重试放大。** 题目说每层最多尝试三次，包含初次调用。如果 A 调 B、B 调 C、C 调数据库都在每次上游尝试中耗尽三次，则一个顶层逻辑请求最多带来 `3 × 3 × 3 = 27` 次数据库尝试。实际倍数受成功时刻、取消和代码路径影响，不能见到三层就断言每次一定 27 次。数据库恢复后，旧请求、重复尝试和新请求可能继续竞争连接池，形成恢复期过载。

**第一步：量出放大和积压所在。**

| 具体观测 | 预期证据与用途 |
| --- | --- |
| 按逻辑请求 ID/幂等键关联各层日志和 trace | 看一个业务操作产生多少 attempt，是否跨层重复重试 |
| 分层 RPS、retry_count、数据库查询量 | 顶层业务量不变而下游尝试增加，支持重试放大 |
| 连接池 active/max、等待人数、借连接耗时 | 区分连接池排队与数据库执行慢 |
| 数据库慢查询、锁等待、连接数、资源利用率 | 排除数据库实际上仍慢或发生锁堆积 |
| 各层 deadline、单次超时、取消记录 | 确认调用者已超时后，下游是否还继续工作 |

这些是可操作的检查项；本题未给数据库类型，不应编造通用 SQL。主机侧可用 `pidstat -u -w -p <pid> 1`、`ss -tanp` 辅助看进程与连接，但看不到逻辑重试次数，不能替代应用证据。还要排除真实流量增长、定时补偿任务、数据库缓存尚冷及历史事务锁未释放。

**第二步：先停止制造额外工作。** 在入口限制并发和排队长度，对低优先级请求快速拒绝；在确定可恢复的层统一管理重试，避免三层独立耗尽预算。只对适合重试的错误采用有界尝试、指数退避和抖动，设整体 retry budget，必要时用熔断隔离异常下游。不要通过无限扩连接池把排队搬到数据库，也不要让客户端在收到拒绝后立即无界重试。

**第三步：按总 deadline 分配超时。** 例如总预算 2 秒，应把排队、连接、执行、返回和退避都算进去，向下游传播剩余截止时间；如果剩余预算不足一次合理调用，就不再启动新尝试。内层单次超时宜留出向外返回和清理的余量。取消是需要实际支持和验证的机制，客户端超时不代表已提交的数据库事务被撤销。

**第四步：单独处理扣款结果未知。** 同一个业务扣款使用稳定幂等键；服务端需要持久化的唯一约束/去重记录，并让业务写入与去重结果原子协调。仅在进程内缓存“处理过”不能应对重启、并发或事务部分完成。重试相同键应返回原结果或明确的处理中状态；相同键但不同参数应拒绝。若调用外部支付方，还要使用其幂等接口与交易查询/对账机制。

超时的历史扣款先查交易状态、按业务流水对账，不换新键盲目补扣；若尚无幂等保证，先暂停自动重试并进行受控核对。验证不仅看错误率下降，还要看业务 RPS 相当时下游尝试倍数、队列、池等待回归，数据库无持续积压，并核对无重复扣款、无漏记和未知状态得到处理。

### B02：连接越来越少，进程 fd 接近上限

**首要判断。** `accept` 需要新的 fd，进程已接近 4096，与进程级限制耗尽一致。系统级文件句柄仍有余量，说明不能只靠修改系统总上限解决。重启会关闭全部 fd，因此短暂恢复支持“资源积累”，但不能单凭这一点证明泄漏。

**第一步：确认实际进程限制与增长趋势。**

```bash
cat /proc/<pid>/limits
sudo ls -1 /proc/<pid>/fd | wc -l
sudo lsof -nP -p <pid>
sudo ss -tanp
```

limits 应显示该服务实际 soft/hard nofile；当前终端的 `ulimit -n` 不是目标进程的限制。fd 计数是近似快照，采样期间可能变化。隔一段时间重复采样并记录 RPS、活跃请求、fd 数、连接数和部署时间，判断增长是否随负载恢复。lsof 的输出还包括 cwd、映射等记录，不能直接用 lsof 行数当 fd 总数。

**第二步：按 fd 类型分流。**

| 观察 | 支持的方向 | 进一步确认 |
| --- | --- | --- |
| CLOSE_WAIT 持续增长 | 对端已关闭，本端未及时 close | 找对应代码路径、异常分支及连接生命周期 |
| ESTABLISHED 随真实并发上升，随后回落 | 容量不足或正常高并发 | 比较并发、请求耗时、池上限与 fd 预算 |
| 大量长时间闲置 socket | 池配置、超时或连接泄漏 | 查池中 active/idle 数、最后活动时间、连接创建栈 |
| 普通文件、pipe 或其他 fd 增长 | 非网络资源泄漏 | 查看 fd 链接与打开/关闭调用的配对 |

TIME_WAIT 通常已没有由该进程持有的活跃 socket fd，不能把全机 TIME_WAIT 数直接当成进程 fd 泄漏。必要时只对一个实例做短时 syscall/运行时追踪，观察 open/socket/accept 与 close 是否平衡；跟踪有开销，不能无边界抓全量生产调用。

**第三步：缓解与根治。** 有其他副本时，先切流再滚动重启或回滚异常版本；限制过高并发、重试和下游等待。若证据表明真实并发需要更多 fd，可评估 socket 内存、下游容量和系统上限后临时提高限制。通过 systemd 等启动器配置的 LimitNOFILE 通常需正确重启服务才作用于新进程，改另一个 shell 的 ulimit 不会追溯改变该服务。

泄漏要修复正常路径及异常路径的资源释放，给连接池和超时设合理界限。只升上限会把同一增长曲线延后触顶。恢复验证应覆盖超过原先六小时的运行周期和峰值负载：fd 能稳定或在低负载时回落、accept 错误消失、吞吐与 p99 恢复，同时没有把压力转移到下游。

### B03：节点维护卡住

**首要判断与算式。** 当前健康副本为 2、要求至少 2，允许中断数为 0；PDB 拒绝自愿 eviction 是按预期保护可用性。第三副本 Pending 不计为健康，无法增加预算。目标不是“让 drain 命令尽快完成”，而是在不低于两个 Ready 的条件下维护。

**第一步：核对预算、放置与容量。**

```bash
kubectl -n trading get pdb <pdb> -o yaml
kubectl -n trading get deployment <deployment> -o yaml
kubectl -n trading get pods -l app=<app> -o wide
kubectl -n trading describe pod <pending-pod>
kubectl describe node <node>
```

PDB 重点看 selector、currentHealthy、desiredHealthy、disruptionsAllowed 及 observedGeneration 是否已反映当前 spec。核对 PDB 是否选到了预期工作负载，两个 Ready 是否真的落在维护节点/其他节点，第三副本 FailedScheduling 是否确因资源 requests。还要排除 affinity、反亲和、配额、PVC 拓扑和节点 taint；不能只给集群总内存加一个数就认为一定能调度。

**第二步：先具备维护容量。** 将维护节点 cordon，防止新负载调度上去；按声明配置扩到三个副本，并在其他节点扩容或安全释放可调度资源，让第三个实例 Ready 且业务探测通过。已存在的第三个 Pending 副本不必重复再扩，应先解决其放置。

```bash
kubectl cordon <maintenance-node>
kubectl -n trading get pods -l app=<app> -o wide
kubectl -n trading get pdb <pdb>
```

当三个健康副本都被正确计入时，minAvailable=2 通常允许一次驱逐。若维护节点上有两个业务副本，必须逐个驱逐，并等其他节点的替代副本 Ready、预算恢复后再继续，不能一次失去两个。drain 还可能被其他业务的 PDB、存储或系统 Pod 阻塞，需分别处理。

**第三步：按预算 drain、维护与恢复。** 经前提检查后执行既有维护命令，例如 `kubectl drain <maintenance-node> --ignore-daemonsets --timeout=10m`；若遇到本地临时数据、无控制器 Pod 等阻塞，先核对含义，不随意追加删除数据或绕开 eviction 的选项。题目要求两个 Ready 时，`--disable-eviction` 绕过 PDB 或把 minAvailable 降为 1 都会降低保护；`--force` 也不是“修复 PDB”的通用参数。

无法提供第三个健康副本、预算没有恢复、错误率恶化或副本跌到阈值时暂停维护，保留节点状态并处理容量/健康问题。完成后检查节点 Ready、网络与存储健康，再按计划 uncordon；业务稳定且无需维护冗余后，通过受管配置恢复副本数。持续监测可用副本、EndpointSlice 与业务成功率；PDB 不防节点突然断电，也不保证每个 Ready 实例都真正能完成业务。预算字段和驱逐约束可参照 [官方 PDB 文档](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)。

## C：26 分

校验 6，分组排序 5，合并逻辑（含相接与嵌套）8，输出 3，测试与复杂度 4。

```python
import re
import sys
from collections import defaultdict

host_re = re.compile(r"[a-z][a-z0-9-]{0,31}")
num_re = re.compile(r"[0-9]{1,12}")
groups = defaultdict(list)
for line in sys.stdin:
    fields = line.split()
    if len(fields) != 3:
        continue
    host, a, b = fields
    if not (host_re.fullmatch(host) and num_re.fullmatch(a)
            and num_re.fullmatch(b)):
        continue
    start, end = int(a), int(b)
    if start < end:
        groups[host].append((start, end))

for host in sorted(groups):
    intervals = sorted(groups[host])
    start, end = intervals[0]
    for a, b in intervals[1:]:
        if a <= end:
            end = max(end, b)
        else:
            print(host, start, end)
            start, end = a, b
    print(host, start, end)
```

n 为有效行数，时间 O(n log n)，空间 O(n)。`[1,10)` 加 `[2,3)` 必须仍为 `[1,10)`；`[1,5)` 加 `[5,9)` 应为 `[1,9)`；不同 host 不合并。空输入、全非法、13 位数字和全角数字均应正确处理。

## D：20 分

四处定位与修复各 3 分，共 12；验证、发布与回滚 6；解释 200 不充分 2。

```nginx
upstream app_backend {
    server 127.0.0.1:8080;
}
server {
    listen 80;
    server_name api.example;
    location /api/ {
        proxy_pass http://app_backend;
        proxy_set_header Host $http_host;
    }
}
```

proxy_pass 不带 URI 部分，保留本题原 URI；原配置尾部 `/` 会替换匹配的 `/api/` 前缀。`$http_host` 保留收到的 Host（含可能的端口）；若需求为规范化主机名，可讨论 `$host` 的不同语义。

先保存旧配置，直连 `curl -H 'Host: api.example' http://127.0.0.1:8080/api/orders`，再 `nginx -t`，成功后受控 `nginx -s reload` 或对应服务 reload。用 `curl -H 'Host: api.example' http://127.0.0.1/api/orders` 通过 Nginx 验证，并检查代理及应用日志里的 URI、Host、状态和时延。必要时从负载均衡路径再验证。失败则恢复旧配置、重新语法检查并 reload，确认业务指标恢复。单一 200 可能来自错误路由、缓存或兜底页面，不足以证明请求语义正确。
