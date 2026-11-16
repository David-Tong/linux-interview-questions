# 模拟 OA 05：Linux / Kubernetes SA

原创练习，非 Optiver 真题。90 分钟，100 分；假设 Linux、Bash、Python 3 与支持 EndpointSlice 的 Kubernetes。全部为合成案例，无需真实环境。A/B/C/D 建议用时 15/25/30/20 分钟。做完再看 [答案与评分](answers-05.md)。

## A. 单选：每题 2 分，共 24 分

**A01**：已打开的普通文件被 rename 到同一文件系统的新路径后，原 fd 通常如何？

A. 自动指向新建的同名文件；B. 仍引用原文件对象；C. 立即失效；D. 变成 socket。

**A02**：进程报告 `Too many open files`，同机其他进程可正常打开文件。最直接的检查是？

A. 增大 MTU；B. 清理 page cache；C. 重启交换机；D. 核对该进程 fd 数与 RLIMIT_NOFILE。

**A03**：用 `kubectl top` 的单次读数判断毫秒级 CPU 峰值，主要局限是什么？

A. 它会自动修复负载；B. 它只能显示磁盘；C. 它显示的是所有历史最大值；D. 采样与聚合可能掩盖短暂峰值，应结合更细粒度指标。

**A04**：Deployment 的 Pod 模板标签改变后不再匹配其 selector，会怎样？

A. API 校验拒绝该不匹配配置；B. 自动创建 Service；C. 静默创建所有 Pod；D. 自动修改 selector。

**A05**：Bash 未开启 pipefail，执行 `false | true` 的退出状态是？

A. 137；B. 随机；C. 1；D. 0。

**A06**：Service 已有 ready EndpointSlice 地址，但客户端超时。合理判断？

A. 一定是 scheduler；B. 证明全路径正常；C. 仍需检查端口、进程监听、策略及数据平面；D. 必须删除集群。

**A07**：排查间歇性磁盘延迟，哪种观察更有用？

A. 只看某次 uptime；B. 只看磁盘容量；C. 连续采样 I/O 延迟、队列、吞吐并关联应用延迟；D. 只看文件名。

**A08**：分析函数耗时，哪个时间来源更适合计算持续时间？

A. 文件 mtime；B. DNS TTL；C. 单调时钟；D. 格式化日期字符串。

**A09**：连接目标返回 TCP RST，通常意味着？

A. 某端点或中间设备主动拒绝/复位，需定位来源；B. 必然发生丢包；C. 三次握手成功；D. DNS 超时。

**A10**：普通文件 mode 为 0640，进程既非 owner，也不属于其 group；无 ACL、无特权。它能否读取？

A. 总可以；B. 不能；C. 取决于扩展名；D. root 所在主机上均可。

**A11**：`ps` 中进程处于 D 状态，发 SIGKILL 后仍显示。合理解释是？

A. 可能仍在不可中断内核等待中，需查等待原因；B. 信号命令必然失效；C. 它是 zombie；D. 它已正常退出。

**A12**：执行脚本提示 permission denied，文件已有执行权限且路径权限正常。哪项值得继续核对？

A. DNS 是否有 AAAA；B. 网卡速率；C. 挂载点是否带 noexec；D. load 是否低于 1。

## B. 故障诊断：每题 10 分，共 30 分

每题写首要假设、至少三个命令或具体观测及预期证据、替代解释、受控缓解与恢复验证。

### B01：数据库恢复后，服务仍雪崩

数据库曾慢了 30 秒，应用有三层调用，每层最多尝试三次且无退避。数据库恢复后，应用请求量仍是正常值的数倍，连接池等待上升。部分请求是扣款操作，调用超时后客户端无法确认结果。如何安全止损？说明重试放大、超时预算、幂等和结果核对的关系。

### B02：连接越来越少

API 进程运行六小时后开始报 `accept: Too many open files`。进程 soft/hard nofile 均为 4096，`/proc/<pid>/fd` 数量接近 4096，系统总文件句柄仍有余量。重启后短暂恢复。如何区分连接泄漏、正常并发增长和其他 fd 泄漏？只调大限制是否算修复？

### B03：节点维护卡住

Deployment replicas=2，PDB minAvailable=2，两个副本均 Ready。drain 使用 eviction API 时提示违反 PDB；第三个副本因资源不足无法调度。业务要求维护期间至少两个 Ready 副本。如何满足维护前提？什么时候应暂停维护？为什么强制绕开 PDB 或单纯把 minAvailable 改成 1 不符合本题目标？

## C. 编码：合并维护窗口，26 分

用 Python 3 标准库实现 stdin → stdout 程序。输入每行三列：`host start end`，空白分隔。

- host 匹配 ASCII `[a-z][a-z0-9-]{0,31}`；start/end 是 1～12 位 ASCII 十进制非负整数，且 start < end。
- 非法行、空行整行忽略。最多 200,000 行。
- 时间单位为秒，窗口为 `[start, end)`；每台 host 分别合并重叠或首尾相接的窗口。重复窗口也合并。
- 按 host 字典序，再按 start 数值升序输出 `host start end`。空结果不输出。
- 提交代码、时间与空间复杂度。测试嵌套、相接、重复、乱序、多 host、非法输入及 Unicode 数字。

输入：

```text
node-b 8 10
node-a 5 9
node-a 1 5
node-a 2 3
node-b 11 12
node-a 9 9
node-b 8 10
```

输出：

```text
node-a 1 9
node-b 8 10
node-b 11 12
```

## D. Nginx 配置排错：20 分

以下是配置节选。Nginx 与应用运行在同一主机、同一网络命名空间；应用只接受 HTTP，监听 `127.0.0.1:8080`，提供 `/ready` 和 `/api/orders`。客户端发送 `GET /api/orders`，期望后端收到的 URI 仍为 `/api/orders`。应用另需正确的原始 Host；TLS 由前置负载均衡终止，Nginx 对内仅提供 HTTP。

```nginx
upstream app_backend {
    server 127.0.0.1:8081;
}
server {
    listen 80;
    server_name api.example;
    location /api/ {
        proxy_pass https://app_backend/;
        proxy_set_header Host wrong.example;
    }
}
```

指出端口、协议、URI 与 Host 四处问题，写出修复节选。列出配置语法检查、后端直连、通过 Nginx 验证、受控 reload 与回滚步骤。成功标准必须包括后端实际收到的 URI 和 Host；只得到一个 200 是否足够？
