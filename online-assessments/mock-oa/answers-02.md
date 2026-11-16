# OA 02 答案与评分

先完成 [试卷](mock-oa-02.md)。评分仅供自测。

## A：24 分

每题 2 分：**A A A C C D B D A B B A**。

1. 环境变量不随 ConfigMap 动态刷新。
2. init 错误看对应容器。
3. TIME_WAIT 本身正常。
4. 分实例证据可避免聚合指标掩盖局部故障。
5. 路由用最长前缀。
6. PDB 约束自愿 eviction。
7. Pod 容器共享网络；文件系统、环境变量和 PID 共享不能一概而论。
8. startup probe 成功后才启动其他探针。
9. SIGKILL 不允许应用做信号清理，也不承诺立即结束内核等待。
10. `--resolve` 保持 URL 主机名。
11. 调度同时受资源、标签和 taint 等约束。
12. si/so 趋势结合压力与延迟判断；首行统计口径需留意。

## B：30 分

每题：假设与因果链 2，命令及预期证据 3，替代解释 1，受控缓解 2，恢复验证 2。等价方案可得分。

命令示例面向 Linux；`<pid>`、`<unit>` 等需替换。写答案时应把“命令 → 预期结果 → 下一步决策”连起来，而不是堆工具名。

### B01：平均 CPU 低，行情处理仍堆积

**首要判断。** 单线程工作阶段可能已打满一核。16 核中一核满载约占总算力的 6.25%，所以全机 8% 与关键线程受 CPU 限制并不矛盾。队列持续增长意味着到达速率长期超过有效处理速率；请求条数未变也不代表每条消息的计算成本没变。

**第一步：证明瓶颈在哪个线程，而不是看全机平均。**

```bash
top -H -p <pid>
pidstat -t -u -w -p <pid> 1
taskset -pc <tid>
cat /proc/<pid>/task/<tid>/status
cat /proc/<pid>/cgroup
```

记录热点 TID、user/system CPU、上下文切换与队列增长时间。对关键工作线程检查 Cpus_allowed_list；每线程 affinity 可以不同，仅检查主线程会漏掉限制。按实际路径查看 cgroup v2 的 `cpu.max`、`cpu.stat`、`cpuset.cpus.effective`，并注意父级限制：如果 throttled 计数和时间持续增长，CPU 配额可能影响吞吐。工具中“100%”的归一化方式也要确认。

**第二步：区分计算、锁与调度限制。**

```bash
sudo perf record -F 99 -g -p <pid> -- sleep 15
sudo perf report
```

上面是短时采样示例，先确认权限和开销；Java/Python 等可能还需要对应运行时 profiler 才能正确还原栈。与旧版本、相同输入数据集对比每条消息 CPU 成本和解析函数占比。

| 假设 | 典型证据 | 不能直接得出的结论 |
| --- | --- | --- |
| 新解析逻辑计算更重 | 热点在解析/转换，关键线程持续占核，每条消息成本上升 | 仅有单核满不证明具体哪个函数有错 |
| 阻塞锁竞争 | 多次栈在锁等待，持锁线程执行过慢，等待时间增长 | CPU 低不代表无瓶颈 |
| 自旋锁竞争 | CPU 高但热点在自旋/锁实现 | 高 CPU 不一定是在做有效业务计算 |
| affinity/cpuset/配额限制 | 可运行线程被限制在少数核，或 throttling 增长 | 加节点物理核不一定自动解除容器限制 |

补看消息大小、字段复杂度、热点分区、CPU 频率与缓存行为。若每条输入更大，RPS 相同也能造成积压；存储平均延迟正常只能降低存储方向的优先级，不能完全排除局部 I/O 等待。

**检查结果如何形成诊断结论。** 这套检查的目的，是把“全机 CPU 不高，但队列堆积”拆成以下四层判断。题面首先支持局部单线程瓶颈的假设；容器配额、锁竞争和调度限制需要进一步取证，也可能同时存在。

**1. 算力真相：全机平均值掩盖了关键线程过载。**

在统一按单核满载为 100% 的口径下，16 核机器上一个核心满载仅占整机的 `1 / 16 = 6.25%`；全机平均 8% 约相当于 1.28 个核心满载，完全容得下一个关键工作线程长期占满单核。如果该线程负责串行解析或分发，它就可能决定整个处理链的吞吐上限。

将队列增长、到达速率和完成速率对齐：若到达速率稳定、完成速率持续更低，说明有效处理能力不足。再比较新旧版本处理同一消息集合的 CPU 时间；若新版本每条消息成本上升且热点集中在新增解析逻辑，才能进一步把性能退化归因到该路径。单核满载本身不能证明具体函数有问题。

**2. 瓶颈性质：计算、内核开销和调度等待要分开解释。**

| 观测结果 | 支持的判断 | 下一步验证 |
| --- | --- | --- |
| 热点 TID 的 `%usr` 持续很高 | 大量 CPU 时间用于用户态执行，可能是解析计算，也可能是忙等或自旋 | 用采样栈区分业务函数与自旋路径，比较每条消息的 CPU 成本 |
| `%system` 显著偏高 | 内核执行开销较大，可能涉及系统调用、缺页或其他内核路径 | 用内核栈及相关计数定位；不能仅凭该指标认定是收发小包或锁竞争 |
| `nvcswch/s` 较高，线程经常可运行 | 可能存在时间片竞争或抢占，但单个切换指标不足以确认原因 | 结合运行队列、调度等待时间、同核任务和线程优先级判断 |
| 多次栈显示阻塞锁等待 | 线程可能在等待其他线程释放锁 | 找到持锁线程，检查持锁时长和等待时间，而不只看等待线程 CPU |

`/proc/<pid>/task/<tid>/status` 中的上下文切换计数是累计值，应在两个时间点取差值；`pidstat` 可直接观察区间速率。非自愿切换增加不能单独证明“被高优先级任务抢占”，也不能直接证明缓存失效是主要损失。

**3. 调度隔离：确认关键线程实际能在哪些 CPU 上运行。**

Linux 线程可以具有不同的 CPU affinity。逐个检查关键 TID 的 `taskset -pc <tid>` 和 `Cpus_allowed_list`，再结合 `cpuset.cpus.effective`，判断多个可运行工作线程是否被限制在同一 CPU 或少数 CPU 上。只查主线程可能漏掉工作线程的独立设置。

这些列表中的编号是逻辑 CPU 编号；若需要判断是否落在同一物理核心的 SMT 线程上，还需查看 CPU 拓扑。发现限制后先核对部署意图：绑核可能是有意的延迟优化，只有限制与设计不符、且与争抢证据相符时，才应调整。严格串行的单线程也不会仅因允许使用更多 CPU 就自动并行。

**4. 容器配额：确认宿主机空闲时，工作负载是否仍受限流。**

根据 `/proc/<pid>/cgroup`，结合实际 cgroup 挂载点定位工作负载所在目录；必要时从宿主机查看。对 cgroup v2，检查本级及相关父级的 `cpu.max`、`cpu.stat` 和 `cpuset.cpus.effective`：

- `cpu.max` 若为 `50000 100000`，表示每 100ms 周期最多使用 50ms 的 CPU 时间，长期平均约为 0.5 个核心的算力预算；`max` 表示本级不设该配额上限，仍需检查父级限制。
- `cpuset.cpus.effective` 表示实际允许使用的 CPU 集合，与 CPU 时间配额是不同约束。允许使用多个 CPU，不代表具有相同数量核心的持续算力预算。
- 在队列增长的同一采样区间，若相关 cgroup 的 `nr_throttled` 和 `throttled_usec` 增加，说明该层级记录了 CPU 配额限流。应看增量及其与延迟的时间关系，不能拿历史累计值直接解释当前事故。

因此，即使宿主机仍有大量空闲 CPU，容器也可能因为自身或父级配额耗尽而被暂停执行。不过，“发生过限流”还不等于“限流是积压的主要原因”；需结合采样窗口，并在受控调整配额后观察处理率、队列和 p99 是否改善。

**可用于作答的结论。** “全机平均 CPU 8% 不能排除关键串行线程已达到单核吞吐上限。优先对比新旧版本同一输入的解析成本，同时排查锁等待、线程 affinity、cpuset 和 CPU 配额；依据热点栈、调度等待及限流增量确定主因。题目没有给出内存压力证据，直接提高 Pod memory limit 无法解释或解决已观察到的单线程计算瓶颈。”

**缓解与验证。** 新版本关联强时先回滚，或者限制输入速率并给积压设上限。只有业务允许按 key/分区拆分且保证顺序、状态和去重正确时才并行化；一个严格串行阶段不会因随意加线程就变快。确认 affinity/配额设置与设计不符时修正限制。提高 memory limit 不会直接增加单线程计算能力，题中也没有内存压力证据。

用相同消息集合和速率重测：稳定处理率应超过到达速率、队列逐步下降、端到端 p99 恢复，且结果、顺序和丢失/重复检查通过。仅把排队消息丢弃造成队列变短不能算性能修复。

### B02：容量够，但无法创建文件

**首要判断。** 创建普通文件通常既需要可用 inode，也需要目录及数据块资源。题中块使用率仅 20%、inode 已满，加上小文件生产和清理失败，首先怀疑小文件积压。即使文件很小，也通常各占一个 inode，释放大文件的数据块未必解决此问题。

**第一步：确认文件系统和占用来源。**

```bash
findmnt -T /data
df -hT /data
df -i /data
sudo du --inodes -x --max-depth=2 /data
```

前三项排除看错挂载点，第四项为 GNU du，用 inode 计数而非字节数定位密集目录。预期临时目录占用异常高，并与两天的生产规模大致一致。百万级目录扫描可能耗时并增加元数据 I/O，宜逐层缩小范围，不一开始遍历整个系统。硬链接共享 inode，因此目录项数量也不一定等于 inode 数。

**第二步：确认为何没清理，以及哪些数据能删除。**

```bash
systemctl status <cleanup-unit>
journalctl -u <cleanup-unit> --since '-2 days'
systemctl list-timers --all
```

若用 cron，则查实际运行用户的 crontab、调度日志和脚本 stderr。区分“没被触发”“启动后报错”“规则写错”“扫描速度赶不上生成速度”。检查路径、权限、工作目录、时间规则以及清理是否遇到 ENOSPC 后无法写锁文件/日志。

先确定业务认可的保留时间来自哪里：文件名、数据库记录、创建记录还是最后修改时间。mtime 是内容修改时间，不等于创建时间；`find -mmin` 也有取整和边界语义。可以先对**已确认的临时子目录**进行只读抽样：

```bash
sudo find /data/<confirmed-temp-dir> -xdev -type f -mmin +70 -print
```

这个例子只适用于业务已确认“mtime 可用于保留判定”的情况；70 分钟是留余量的示例，不能机械替代题目的最近一小时要求。核对候选清单和生产者行为后，再按固定截止时间、正确处理特殊文件名的方式分批删除。

**第三步：止损与恢复。** 临时降低文件生成速率，先清理确定超期且不再使用的一批文件，为业务和清理程序腾出 inode；随后修复调度或清理逻辑并追赶积压。打开但已删除的文件在最后引用释放前可能仍占 inode，要结合 lsof 检查。若占用来源不是临时目录，则排查其他生产者、挂载对象或文件系统特定限制。

`rm -rf /data/*` 会删除保留期内数据和其他业务内容，不能满足题意；它也不能解决持续生成、错误轮转或仍打开引用的问题。恢复标准是 inode 有持续余量、业务可创建文件、最近一小时数据完整、清理下一周期按预期完成。长期监控 inode 使用率与增长率，设置保留上限，必要时改用批量文件或其他存储组织方式。

### B03：仅一台机器访问失败

**首要假设。** B 的解析或地址缓存过旧，使它连接错误目标；但旧地址为何超时仍是另一个问题。应先证明“新地址在 B 上可用”，再定位旧地址来源，避免一上来重启所有 DNS 服务。

**第一步：保持域名语义，固定连接 IP。**

```bash
curl --noproxy '*' --connect-timeout 3 --max-time 10 -v \
  --resolve api.example:443:<new-ip> https://api.example/ready
```

这里 `/ready` 应换成实际安全探测路径。URL 仍是 api.example，所以 Host、TLS SNI 和证书主机名校验仍保留。若 B 这样访问成功，而普通域名访问连接旧 IP 超时，说明新 IP 的基本连通性和 TLS 路径可用，解析/缓存方向更强。若固定新 IP 也失败，继续查 B 的路由、防火墙和证书信任，不应仅修 DNS。直接请求 `https://<new-ip>/` 再加 Host 不等价于正确设置 SNI，也不要用 `-k` 掩盖验证失败。

**第二步：按解析层次比较 A 与 B。**

```bash
getent ahosts api.example
cat /etc/nsswitch.conf
cat /etc/hosts
cat /etc/resolv.conf
resolvectl status
resolvectl query api.example
dig @<recursive-resolver-ip> api.example A
dig @<recursive-resolver-ip> api.example AAAA
```

resolvectl 只在使用 systemd-resolved 的主机适用。重点比较 NSS 的 hosts 查询顺序、本地覆盖项、实际递归服务器、搜索域/VPN 分流，以及 A/AAAA 记录。dig 直接询问 DNS，并不按系统 NSS 顺序读取 `/etc/hosts`，因此 dig 正常、getent 仍旧时，优先查本地来源。

| 结果 | 下一步 |
| --- | --- |
| hosts 明确有旧地址，NSS 优先 files | 修正受管 hosts 配置并验证，不必清所有 DNS 缓存 |
| 递归服务器返回新地址，本地解析返回旧地址 | 查本地 resolver 缓存、VPN/多接口解析及实际查询路径 |
| 某递归服务器仍返回旧地址 | 核对权威记录与剩余 TTL，判断正常缓存、记录未更新还是查询了不同视图 |
| 系统解析已新，但应用仍连旧地址 | 查运行时 DNS 缓存、连接池、静态配置和进程启动参数 |

若应用有独立网络命名空间或运行在容器里，应在相同环境检查，宿主机 getent 不代表应用结果。连接池还可能持续复用旧 IP 上已建立的连接，这与 DNS 缓存不同。

**缓解与验证。** 修正已确认的来源，只刷新实际涉及的缓存，必要时滚动重建应用连接池。若 DNS TTL 尚未到期，不能把正常缓存一概称为故障；发布迁移宜让旧后端保留到缓存和连接排空完成。验证 A/B 的解析、应用真实连接的 remote IP、证书校验、错误率及延迟；等待跨过相关缓存周期再复查，确保不是一次强制 curl 成功而应用仍错误。

## C：26 分

校验 5，事件建模 6，同刻聚合与半开区间 7，首次峰值及排序 4，复杂度与测试 4。

```python
import re
import sys
from collections import defaultdict

name_re = re.compile(r"[a-z][a-z0-9-]{0,31}")
num_re = re.compile(r"[0-9]{1,12}")
events = defaultdict(lambda: defaultdict(int))
for line in sys.stdin:
    f = line.split()
    if len(f) != 3:
        continue
    service, a, b = f
    if not (name_re.fullmatch(service) and num_re.fullmatch(a)
            and num_re.fullmatch(b)):
        continue
    start, end = int(a), int(b)
    if start >= end:
        continue
    events[service][start] += 1
    events[service][end] -= 1

for service in sorted(events):
    active = peak = 0
    earliest = None
    for t, delta in sorted(events[service].items()):
        active += delta
        if active > peak:
            peak, earliest = active, t
    print(service, peak, earliest)
```

同刻增减合计后再比较峰值，避免把相接区间算作重叠；严格 `>` 保留最早时刻。n 条有效记录，时间 O(n log n)，空间 O(n)。测试：`[0,5)`、`[5,10)` 峰值 1、最早 0；两条重复 `[0,5)` 峰值 2、最早 0；嵌套、并列峰值、多服务、乱序、空和非法输入。

## D：20 分

发布策略 5，startup 5，readiness/liveness 4，验证与回滚 4，限制 2。

```yaml
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    spec:
      containers:
        - name: quotes
          image: registry.example/quotes:v2
          startupProbe:
            httpGet: {path: /startup, port: 8080}
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 24
          livenessProbe:
            httpGet: {path: /live, port: 8080}
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 3
          readinessProbe:
            httpGet: {path: /ready, port: 8080}
            periodSeconds: 5
            timeoutSeconds: 1
            failureThreshold: 1
```

startup 名义探测预算约 120 秒，给 75 秒启动留余量；并非承诺精确到秒的调度计时。maxUnavailable=0 使控制器等待新副本可用后再缩旧副本；maxSurge=1 需可承载额外 Pod，终止中的 Pod 还可能暂时占用额外资源。可按稳定性要求加 minReadySeconds。

修改 Git 中的受管配置，先审查 diff 和配置校验，再经 GitOps 同步；`kubectl rollout status deployment/quotes -n trading --timeout=300s`，配合 `get pods -w`、`describe pod`、EndpointSlice 及业务合成请求，观察 ready 副本、重启数、错误率与延迟。测试冷启动和依赖短暂故障的语义。失败则停止推进，在 Git 中回退本次配置/镜像提交并同步，确认旧版本和业务指标恢复；self-heal 场景只执行临时 rollout undo 可能被覆盖。

该策略约束控制器的计划缩容，不保证节点故障、共享依赖失败、应用误报 Ready、连接排空不足或流量层传播延迟时仍有两个真正能服务的副本。readiness 不是所有请求成功的证明。
