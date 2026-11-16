# 模拟 OA 04：Linux / Kubernetes SA

原创练习，非 Optiver 真题。90 分钟，100 分；假设 Linux、Bash、Python 3 与支持 EndpointSlice 的 Kubernetes。全部为合成案例，无需真实环境。A/B/C/D 建议用时 15/25/30/20 分钟。做完再看 [答案与评分](answers-04.md)。

## A. 单选：每题 2 分，共 24 分

**A01**：服务变慢后客户端立即无限重试，可能造成什么？

A. 放大负载，应使用重试预算、退避与抖动，并考虑幂等性；B. 必然无损恢复；C. 自动降低负载；D. 所有请求只执行一次。

**A02**：客户端只抓到反复 SYN，服务端未抓到 SYN。哪项最严谨？

A. 已证明应用代码死锁；B. 已证明证书过期；C. 必须重启数据库；D. 检查抓包接口、过滤条件及中间路径，尚不能唯一归因。

**A03**：TLS 验证提示证书主机名不匹配，应优先核对？

A. 请求主机名、SNI、证书 SAN 及代理所选证书；B. nice；C. swap；D. inode。

**A04**：应用收到 SIGTERM 后立即退出，但仍有请求在处理，合理改进是？

A. 忽略所有信号；B. 停止接收新请求，按预算完成或终止在途请求，再退出；C. 永不退出；D. 只改 containerPort。

**A05**：单次 ping 丢包能否证明业务 TCP 同比例丢包？

A. 一定证明 MTU 过大；B. 不能，协议处理、限速和路径等可能不同；C. 一定证明服务不可用；D. 能。

**A06**：toleration 的含义是？

A. 允许容忍匹配 taint，但不保证调度到该节点；B. 等于 nodeSelector；C. 保证独占节点；D. 自动增加资源。

**A07**：准备替换配置文件，哪种方式通常更有利于避免读者看到半写入内容？

A. 先删除父目录；B. 写一半后 sleep；C. 在同一文件系统写好临时文件，再 rename 替换；D. 原路径逐字覆盖。

**A08**：容器内 `localhost:9000` 默认指向哪里？

A. 集群 DNS；B. 该容器所在网络命名空间的回环接口；C. Service VIP；D. 任意集群节点。

**A09**：服务只监听 `127.0.0.1:8080`，客户端位于另一主机。最直接的问题？

A. 必须用 UDP；B. 回环监听不能直接接受发往主机外部地址的连接；C. 端口号太大；D. 缺少 AAAA。

**A10**：systemd 服务反复启动失败并触发启动速率限制。合理顺序是？

A. 查失败日志、修复原因，再按需 reset-failed 并启动；B. 改成 root 即可；C. 无限循环 restart；D. 删除 unit。

**A11**：事故中计划同时修改线程数、连接池和超时，主要诊断风险是？

A. 难以判断哪个变化生效或引入副作用，宜控制变量并设回滚标准；B. 必然触发 OOM；C. 修改一定失败；D. 日志一定清空。

**A12**：镜像改成同一 tag 的新内容后，单凭 tag 可以保证各节点运行同一内容吗？

A. 可以；B. readiness 会校验；C. 不可以，宜用不可变版本或 digest 并验证实际 imageID；D. Service 会校验。

## B. 故障诊断：每题 10 分，共 30 分

每题写首要假设、至少三个命令及预期证据、替代解释、受控缓解和恢复验证。

### B01：定时任务静默失败

脚本在交互式 Bash 中成功，在 cron 中失败。脚本使用 `#!/bin/bash`，包含 `cd /srv/report` 和 `python3 build.py | gzip > out.gz`；未检查 cd 结果，未设置 pipefail，依赖交互 shell 中导出的 TOKEN 和自定义 PATH。错误日志中有 `python3: command not found`，任务仍被记录为成功。如何解释、修改并验证？不要把密钥打印到日志。

### B02：有空闲资源却 Pending

Pod request 1 CPU、1Gi，三个节点均有足够的未分配资源。Pod 设置 `nodeSelector: {pool: lowlatency}`。只有 node-1 带该标签，且有 `dedicated=trading:NoSchedule` taint；Pod 无 toleration。如何建立证据链？为什么不能直接删除 taint？给出符合隔离意图的两种部署决策。

### B03：新版本连接数暴涨

新客户端版本上线后，每秒请求数不变，TCP 新建连接率增为原来的 20 倍，客户端大量 TIME_WAIT，偶发 `Cannot assign requested address`。服务端处理时间基本不变。如何验证连接复用与本地临时端口压力？还需排除哪些地址绑定或 NAT 问题？给出回滚及恢复标准。

## C. 编码：连续失败告警区间，26 分

Python 3 标准库程序，从 stdin 读取每行 `timestamp service result`。

- timestamp 为 1～12 位 ASCII 非负十进制整数；service 匹配 `[a-z][a-z0-9-]{0,31}`；result 只能是 `OK` 或 `FAIL`。非法行、空行忽略，最多 200,000 行。
- 每个 service 按 timestamp 数值升序处理；同 service、同 timestamp 只保留输入中最后出现的有效记录。
- 连续至少三次 FAIL 构成一个告警段；只看采样顺序，不限制时间间隔。一次 OK 结束当前连续失败段。
- 对每个达到阈值的最大连续失败段，输出 `service start_ts trigger_ts end_ts fail_count`：start_ts 为第一条 FAIL 时间，trigger_ts 为第三条 FAIL 时间，end_ts 为结束该段的 OK 时间；若输入结束仍未恢复，end_ts 输出 `-`。
- 不足三次的失败段不输出；按 service 字典序，再按 start_ts 数值升序输出。空结果不输出。
- 提交代码、复杂度和测试：乱序、重复时间覆盖、恰好三次、两次后恢复、多个失败段、未恢复、非法输入。

输入：

```text
30 quotes FAIL
10 quotes FAIL
20 quotes FAIL
40 quotes OK
50 quotes FAIL
60 quotes FAIL
70 quotes FAIL
80 gateway FAIL
90 gateway OK
20 quotes OK
20 quotes FAIL
```

输出：

```text
quotes 10 30 40 3
quotes 50 70 - 3
```

## D. Bash 修复：20 分

备份任务把 `/srv/app/config.json` 复制到**已存在**的 `/backup`，文件名由脚本第一个参数指定，只允许 ASCII 字母、数字、点、下划线、短横线组成的 1～64 字符 basename，且不得为 `.` 或 `..`。输入路径、目标目录均可信且不会被并发替换；目标可能已存在，失败时必须保留旧备份。GNU/Linux 上可用 Bash、mktemp、cp、mv。

```bash
#!/bin/bash
name=$1
cp /srv/app/config.json /backup/$name
echo backup-ok
```

写出完整修复脚本：检查恰好一个参数，校验名称，正确引用变量，在 `/backup` 内建临时文件，复制成功后原子替换目标；失败返回非零且不输出成功，退出时清理临时文件。说明原子可见性与断电持久性的区别，并列四个验证场景。
