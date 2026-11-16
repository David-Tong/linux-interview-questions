# Linux / Kubernetes SA 面试追问题库

以下 30 道均为依据岗位方向原创的面试练习，不是已确认的 Optiver 面试题。括号内是自查要点，建议先遮住作答。

回答场景题时按“影响范围 → 时间线与最近变更 → 假设 → 证据 → 小范围缓解 → 恢复验证 → 防止复发”展开。不要只背命令。

## Linux：10 题

**L01**：Explain what happens from power-on until a systemd service is ready. 若机器进 emergency mode，你从哪里定位？

要点：固件、引导器、内核/initramfs、根文件系统、PID 1、依赖；控制台、内核命令行、journal、fstab、存储。追问：网络服务启动顺序不代表网络已经可用，如何设计 readiness？

**L02**：load 高但 CPU idle 高，和 CPU 真正饱和如何区分？

要点：runnable 与 D 状态、`vmstat`、`pidstat`、`iostat -xz`、PSI、栈；短时与趋势。追问：为什么 iowait 不是单一磁盘故障的证明？

**L03**：服务 p99 从 2ms 变成 20ms，但平均值变化很小，你先看什么？

要点：直方图、排队、调度延迟、CPU quota、GC、I/O、网络重传、NUMA 与最近变更。追问：如何控制变量验证优化，不让压测掩盖尾延迟？

**L04**：`free` 显示 used 很高，怎样证明机器存在内存压力？

要点：MemAvailable、swap in/out、major fault、PSI、OOM、工作集趋势。追问：容器内看到的内存和宿主机为何可能不同？

**L05**：文件已经删除，空间为什么没恢复？硬链接、软链接和打开 fd 有什么区别？

要点：inode、目录项、link count、fd 生命周期、`lsof +L1`。追问：怎样让服务安全重新打开日志？

**L06**：应用报告 too many open files，如何区分进程限制与系统总量？

要点：`/proc/<pid>/limits`、fd 数量、systemd LimitNOFILE、file-nr、连接泄漏。追问：为什么提高上限只可能暂缓问题？

**L07**：解释 fork、exec、线程、上下文切换及 zombie。

要点：地址空间、写时复制、共享资源、调度、wait。追问：为什么 SIGKILL 也无法让某些 D 状态任务立即消失？

**L08**：systemd 服务启动失败，但手动运行成功，怎么查？

要点：`systemctl status/cat`、`journalctl -u`、用户、工作目录、环境、权限、沙箱、依赖。追问：如何避免无限重启形成日志风暴？

**L09**：裸机升级内核前，你准备哪些检查与回滚路径？

要点：驱动/固件、NIC、磁盘、引导项、带外管理、canary、冗余容量、业务迁移、旧内核可启动。追问：无法 SSH 时如何区分网络故障与启动失败？

**L10**：CPU pinning、NUMA、IRQ affinity、huge pages 何时有用？

要点：先有测量与负载模型；局部内存、争用、TLB、隔离与容量取舍。追问：为什么低延迟优化可能降低总体吞吐或可维护性？

## Kubernetes：10 题

**K01**：从提交 Deployment 到 Pod 开始接流量，经过哪些组件？

要点：API、持久化、控制器、调度器、kubelet、CRI、CNI、探针、Service 后端。追问：调度成功为什么不代表镜像能拉取或网络可用？

**K02**：Pending 的 Pod 如何分类诊断？

要点：events、requests、taints/tolerations、affinity、PVC、配额、调度门控。追问：节点总空闲资源足够为什么仍调度失败？

**K03**：CrashLoopBackOff、OOMKilled、Evicted 分别说明什么？

要点：重试退避、终止原因、节点驱逐、previous logs、events 与历史监控。追问：137 是否必然 OOM？

**K04**：readiness、liveness、startup probe 如何搭配？

要点：接流量、重启、慢启动；不要将外部依赖暂时故障放大为重启风暴。追问：readiness 失败时已有连接是否必定马上断开？

**K05**：ClusterIP 不通，Pod IP 通，如何定位？

要点：DNS、selector、EndpointSlice、端口、就绪状态、Service 数据平面、CNI/eBPF 或 kube-proxy 模式。追问：为什么不能假设所有集群都有 kube-proxy？

**K06**：解释 Cilium、eBPF、底层网络与 Kubernetes NetworkPolicy 的边界。

要点：节点 agent、数据平面、身份/策略、路由或封装、观测；区分标准 NetworkPolicy 和 Cilium 扩展策略。追问：策略允许是否证明 underlay 一定通？

**K07**：默认拒绝 egress 后，怎样只允许 DNS 和指定业务目的地？

要点：源 selector、namespace/pod selector、实际 DNS 路径、UDP/TCP 53、目的端口；标准 NetworkPolicy 不提供通用 FQDN 规则。追问：NodeLocal DNS 对规则有什么影响？

**K08**：维护节点时，cordon、drain 和 PDB 分别做什么？

要点：新调度、驱逐、可用性预算、DaemonSet、本地数据、容量；PDB 约束特定自愿中断，不保证抵御节点故障。追问：drain 被 PDB 阻塞时如何判断应该扩容还是改变维护方案？

**K09**：三节点 etcd 集群坏一台，你如何安排维护？

要点：多数派、先恢复健康、备份及恢复演练、磁盘延迟、网络与证书。追问：备份成功为什么不等于恢复能力已验证？

**K10**：容器已经 non-root，为什么仍需 RBAC、capabilities、seccomp 和网络隔离？

要点：控制平面权限、内核攻击面、宿主机挂载、service account、最小权限、凭据生命周期。追问：如何验证 RBAC 权限而不直接授予 cluster-admin？

## 网络、自动化与生产判断：10 题

**P01**：Explain a TCP connection and how you would investigate intermittent timeouts.

要点：握手、重传、队列、conntrack、路径、双端抓包；区分连接超时与应用响应超时。

**P02**：DNS 解析成功但 HTTPS 请求失败，你如何逐层排查？

要点：IP/路由、TCP、TLS/SNI/证书/时间、HTTP；解析成功不证明返回 IP 正确。

**P03**：为什么 MTU 不一致可能只影响大响应或部分跨节点流量？

要点：封装开销、PMTUD、ICMP 过滤、TCP MSS、不同路径；用包长和抓包验证。

**P04**：ArgoCD self-heal 环境里，紧急热修复怎样避免反复被覆盖？

要点：优先 Git 修复；必要时按事故流程协调同步行为、限制范围、保留审计、及时回写 Git 并恢复管理。

**P05**：Terraform plan 显示要替换生产资源，你怎么判断？

要点：replacement 原因、provider 版本、state/drift、依赖、备份与恢复、审批和维护窗口；不直接 apply 试错。

**P06**：怎样让基础设施 GitHub Actions 在部分失败后可以安全重跑？

要点：幂等、锁与并发、分阶段验证、状态与恢复、最小权限、固定依赖版本、敏感信息管理。

**P07**：Rancher 管理界面不可用，但业务正常，你如何判断影响范围？

要点：区分管理平台与下游集群；检查直接 API 访问、控制面和数据面；按预置应急凭据流程操作。

**P08**：你接到交易时段延迟告警，前五分钟做什么？

要点：确认影响与可信指标、确定负责人和时间线、检查变更、缩小范围、选择低风险缓解、持续沟通；保留诊断证据。

**P09**：Describe an incident you owned and what changed afterwards.

要点：用自己的经历讲清影响、证据、个人判断、协作、恢复和可衡量改进，避免只讲团队做了什么。

**P10**：设计一个能发现 Linux/K8s 平台性能退化的监控方案。

要点：业务成功率与延迟 SLI、资源饱和与压力、网络、控制平面、关联维度、可行动告警、runbook；避免只监控 CPU 平均值。
