# OA 04 答案与评分

先完成 [试卷](mock-oa-04.md)。评分仅供自测。

## A：24 分

每题 2 分：**A D A B B A C B B A A C**。

1. 重试会增加负载；写操作尤其需要幂等与结果确认。
2. 抓包结论受位置与过滤器约束。
3. TLS 要核对 SNI/SAN。
4. 优雅终止需处理接流量、在途请求与总时间预算。
5. ICMP 不能代替业务探测。
6. toleration 只允许、不吸引。
7. 同文件系统 rename 提供原子名称替换，不等于断电持久性。
8. localhost 属于当前网络命名空间，不自动指向宿主机。
9. 回环监听不接受外部目的地址。
10. 先解决原始错误，再解除速率限制；reset-failed 不是修复原因。
11. 控制变更范围、保留证据和回滚条件，有助于因果判断。
12. 可变 tag 不保证内容一致。

## B：30 分

每题：假设 2，命令与预期证据 3，替代解释 1，缓解 2，验证 2。

命令以 trading 命名空间为例，Pod、Deployment、节点与 PDB 名称需替换。修改通过声明配置/既有发布流程落地；查询命令与变更命令用途不同，不能把变更当成诊断的第一步。

### B01：定时任务静默失败

**先解释完整因果链。** cron 的环境通常比交互 shell 精简，自定义 PATH 与 TOKEN 不会自动继承。找不到 python3 时，该命令通常返回 127；但 gzip 可能把空输入压缩成功，默认 Bash 管道状态取最后一个命令，因此整条管道仍可返回 0。若 cd 失败，脚本还可能在错误目录执行并覆盖错误位置的 out.gz。

**第一步：查运行身份、环境假设与错误传播。**

```bash
crontab -l                         # 当前用户的任务；不一定是出问题的任务
sudo crontab -u report -l          # 假设实际任务用户为 report
sudo cat /etc/crontab              # 若任务来自系统 crontab
sudo ls -l /etc/cron.d             # 再读取实际定义该任务的文件
ls -ld /srv/report
ls -l /srv/report/build.py
```

**1. 先确认是哪一个用户在运行。** `crontab -l` 只列出当前用户的个人任务；以管理员身份执行时，看到的可能是管理员自己的任务。若实际任务属于 `report`，就应查 `sudo crontab -u report -l`。个人 crontab 的命令前只有五个时间字段，运行身份由该 crontab 的所属用户决定；`/etc/crontab` 与 `/etc/cron.d/` 中的任务通常还在时间字段后显式指定用户。例如：

```cron
# report 用户的个人 crontab
0 2 * * * /bin/bash /srv/report/report.sh

# /etc/crontab 或 /etc/cron.d/report：多一个 report 用户字段
0 2 * * * report /bin/bash /srv/report/report.sh
```

不能把两种格式原样互换。确认身份后，还要检查该用户能否逐级穿过 `/srv/report` 的父目录、读取脚本与凭据，以及在输出目录创建和替换文件。目录的 `x` 权限表示可以穿过目录；只看脚本自身的读写权限不够。必要时用 `id report`、`namei -l /srv/report/build.py` 查看用户组与路径各级权限，再以实际任务身份验证访问。

**2. 区分“cron 解析命令行的 shell”和“执行脚本的解释器”。** cron 通常用 `/bin/sh` 解析任务命令行，具体还要看 crontab 的 `SHELL` 设置；脚本最终用哪个解释器，则取决于调用方式：

| 任务命令 | 脚本由谁解释 | 注意事项 |
| --- | --- | --- |
| `/srv/report/report.sh` | 直接执行时按脚本的 `#!/bin/bash` 选择 Bash | 脚本需有执行权限，路径各级目录需可穿过 |
| `/bin/bash /srv/report/report.sh` | 显式指定的 Bash | 脚本需可读，不要求自身有执行权限 |
| `/bin/sh /srv/report/report.sh` | 显式指定的 sh，shebang 不参与选择 | 不能依赖 Bash 的 `[[ ... ]]`、`source` 或 `pipefail` 等特性；不同 sh 支持情况不同 |

因此，脚本首行写了 Bash，并不保证 `sh report.sh` 会用 Bash。对于下面使用 Bash 语法的修复脚本，任务命令也应明确使用 Bash，或直接执行带正确 shebang 的脚本。

**cron 任务的环境变量从哪里来？** crontab 是任务配置，真正启动任务的是 cron 服务。任务环境主要来自 cron 的默认设置、系统/PAM 配置（取决于实现）、crontab 中的变量赋值，以及命令或脚本显式设置的变量，不能假设与登录终端一致。

| 变量 | 常见默认来源与用途 |
| --- | --- |
| `SHELL` | 通常为 `/bin/sh`，用于解析任务命令行；可在 crontab 中设置 |
| `HOME` | 通常来自任务用户的账户信息，可在 crontab 中设置；不等于脚本所在目录 |
| `LOGNAME` | 通常来自任务用户的登录名；常见实现不允许在 crontab 中覆盖它来改变身份 |
| `PATH` | cron 提供的程序搜索路径，具体默认值因系统与配置而异，应显式设置或使用绝对路径 |
| `LANG`、`LC_ALL`、`TZ` | 是否存在及其值取决于系统配置和显式设置，不能默认与终端一致 |
| `TOKEN` 等业务变量 | 需要通过受控配置显式提供，不能指望继承终端中的 `export` |

终端执行 `export TOKEN=...` 或修改 `PATH`，只会影响当前 shell 及其随后启动的子进程。独立运行的 cron 服务通常不是该终端的子进程，所以不会收到这些修改。普通 cron 任务通常也不会自动读取 `.bashrc`、`.bash_profile`；仅设置 `SHELL=/bin/bash` 不会使任务变成登录式或交互式 Bash。

非敏感变量可以直接写在个人 crontab 中，放在需要使用它们的任务之前：

```cron
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
LANG=C

0 2 * * * /bin/bash /srv/report/report.sh
```

这些设置会传给后续任务。`LANG=C` 是明确的示例；若程序需要 UTF-8，应选择目标系统实际支持的 locale，并注意 `LC_ALL` 会覆盖 `LANG`。crontab 的环境赋值不是 Bash 语句，不要写 `export PATH=...`，也不要依赖 `PATH=$HOME/bin:$PATH` 自动展开；需要自定义目录时写完整路径，例如 `PATH=/home/report/bin:/usr/bin:/bin`。`SHELL` 只决定命令行解析方式，不会改变前面表格中显式 `sh script.sh` 的解释器。

TOKEN 应由脚本从权限受控、内容可信的配置文件加载，例如下面修复脚本中的 `source /etc/report/report.env`，然后通过 `export TOKEN` 传给 Python。`source` 会执行文件中的 shell 代码，因此这个文件必须受信任，不能把任意外部输入当成配置加载。配置文件应只包含所需配置，不应为了方便直接加载整个个人 shell 配置。

要观察任务实际收到的非敏感变量，应把检查放进 cron 执行的测试脚本，而不是在终端查看：

```bash
printf 'user=%s\n' "$(id -un)"
printf 'HOME=%s\nPATH=%s\n' "${HOME:-<unset>}" "${PATH:-<unset>}"
printf 'LANG=%s\nLC_ALL=%s\nTZ=%s\n' \
  "${LANG:-<unset>}" "${LC_ALL:-<unset>}" "${TZ:-<unset>}"
```

环境中的 `SHELL` 值不一定代表正在解释脚本的进程；例如任务环境中 `SHELL=/bin/sh`，仍可以通过 `/bin/bash report.sh` 运行 Bash 脚本。TOKEN 只检查是否配置，见下文，不能一起打印到日志。

**3. 在隔离的测试目录重现精简环境。** 先准备 `/srv/report-test/test-report.sh`、测试输入、测试凭据及单独的输出路径，确认测试脚本不会写入 `/srv/report/out.gz` 或调用生产写接口。以下路径和用户名均为示例，需按已准备好的测试环境替换：

```bash
sudo -u report -- /usr/bin/env -i \
  PATH=/usr/bin:/bin HOME=/srv/report-test \
  /bin/bash --noprofile --norc -c \
  'cd /srv/report-test && exec /bin/bash ./test-report.sh'
```

`sudo -u report` 切换运行身份；`env -i` 清除继承的环境变量，再仅设置列出的变量；`--noprofile --norc` 避免读取常见的 Bash 启动文件；显式 `cd` 固定工作目录。测试凭据应由测试脚本从受控文件读取，不要把 TOKEN 值写在命令行里。`env -i` 不会把进程变成 cron，也不会隔离文件系统或网络；它主要用于验证“缺少交互环境是否触发问题”。

先用精简环境观察故障，再逐项补齐实际任务所需条件，并比较结果：

- **身份与权限：** 用户、附加组、目录访问权限和凭据文件读取权限是否一致。用 root 测试成功，不能证明任务用户也能运行。
- **工作目录与 HOME：** `build.py`、配置、输入和输出使用了哪些相对路径；cron 不会因为脚本位于 `/srv/report` 就自动切到该目录。
- **解释器与虚拟环境：** 交互 shell 的 `command -v python3` 只能说明当前 shell 找到了什么。若项目依赖 venv，应核对并直接使用例如 `/srv/report/.venv/bin/python`，且确认该解释器安装了需要的包；不必依赖交互式 `activate`。
- **locale、时区与运行时依赖：** 对比实际需要的 `LANG`/`LC_ALL`、时区、外部程序、动态库、网络/DNS、证书和代理配置。不要为了“环境完整”就把整个交互环境复制进任务。

**4. 只记录配置状态，不泄露内容。** 可以在实际执行 Python 的脚本中、加载配置之后加入以下 Bash 检查，先用测试凭据验证：

```bash
if [[ -n ${TOKEN:-} ]]; then
    printf 'TOKEN is configured\n' >&2
else
    printf 'TOKEN is not configured\n' >&2
    exit 1
fi
export TOKEN
```

`${TOKEN:-}` 在变量未设置时也能安全检查，空字符串按未配置处理；`export TOKEN` 让随后启动的 Python 子进程能够继承它。这个检查只证明变量非空，不能证明凭据有效、未过期或有足够权限，后者要用测试环境中的实际请求验证。不要使用 `echo "$TOKEN"`、完整的 `env`/`printenv` 输出或 `set -x` 排查凭据，否则日志可能记录密钥。若精简环境测试通过，还应通过实际 cron 调度执行同一测试任务，核对退出状态、错误日志和产物。

**第二步：写出能保留旧产物的脚本。** 以下假设目录可信、GNU mv 可用、凭据文件由运维维护为仅任务用户可读的 Bash 配置文件，内容受信任；Python 路径需替换成已核对的解释器或 venv。

```bash
#!/bin/bash
set -euo pipefail
export PATH=/usr/bin:/bin
umask 077
cd /srv/report || exit 1
source /etc/report/report.env
if [[ -z ${TOKEN:-} ]]; then
    printf 'TOKEN is not configured\n' >&2
    exit 1
fi
export TOKEN
tmp=''
trap 'if [[ -n $tmp ]]; then rm -f -- "$tmp"; fi' EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
tmp=$(mktemp /srv/report/.out.XXXXXXXXXX)
if /usr/bin/python3 build.py | /usr/bin/gzip > "$tmp"; then
    mv -fT -- "$tmp" /srv/report/out.gz
    tmp=''
    printf 'report-ok\n'
else
    printf 'report generation failed\n' >&2
    exit 1
fi
```

pipefail 让 Python 失败或 gzip 失败都令管道失败；显式 if 处理管道结果，不依赖对 `set -e` 的模糊理解。同目录临时文件避免失败产物覆盖旧文件；mv 只在成功后发布。脚本只输出非敏感状态，cron 需把 stderr 交给可监控的日志通道。

**验证与替代解释。** 在测试环境依次注入 cd 失败、解释器不可用、TOKEN 缺失、Python 产生部分数据后失败、gzip/写盘失败以及正常成功。失败均应退出非零、保留旧 out.gz 并清理临时文件；成功后还要 `gzip -t` 和检查解压内容/记录数。压缩包完整不等于业务数据正确。若 PATH 修复后仍失败，再查权限、venv 依赖、网络、时区及任务重叠；重复调度还可能需要单实例锁，但它不是当前错误的首要原因。

### B02：有空闲资源却 Pending

**首要判断。** 调度需要同时满足所有硬约束，不是“集群总剩余资源够就行”。nodeSelector 先把候选节点缩小到 node-1，该节点又带 Pod 不容忍的 NoSchedule taint，结果没有合格节点。

**第一步：收集调度器看到的配置与事件。**

```bash
kubectl -n trading describe pod <pod>
kubectl -n trading get pod <pod> -o yaml
kubectl get nodes -L pool
kubectl describe node node-1
```

describe 中应看到 FailedScheduling，原因包含标签不匹配与未容忍 taint。事件措辞和计数可能随版本变化，不应死记固定字符串。get pod 看准入之后的实际 requests、nodeSelector、affinity 和 tolerations，而不只看 Git 原始 YAML。describe node 看 taint、allocatable 和已分配 requests，不能拿 top 的瞬时空闲代替调度容量。

| 节点 | pool 是否匹配 | taint 是否容忍 | 结果 |
| --- | --- | --- | --- |
| node-1 | 是 | 否 | 被 taint 排除 |
| node-2 | 否 | 题目无须继续判断 | 被 selector 排除 |
| node-3 | 否 | 题目无须继续判断 | 被 selector 排除 |

**第二步：按部署意图选择修复。** 若该工作负载确实属于 trading 专用池，在 Pod 模板加入精确容忍：

```yaml
spec:
  template:
    spec:
      nodeSelector:
        pool: lowlatency
      tolerations:
        - key: dedicated
          operator: Equal
          value: trading
          effect: NoSchedule
```

保留 selector 是为了继续约束目标池；toleration 只是允许进入，不会吸引或保证调度到该节点。若业务不属于专用池，应改到获准的通用节点池，或增加符合业务要求且有容量的节点，而不是无条件给所有 Pod 加宽泛 Exists 容忍。

删除 node-1 的 taint 会改变整个节点的准入边界，使其他原本被阻挡的负载也可能进入，且不代表已经验证隔离、性能和容量要求。标签与 taint 都是配置手段，不是完整的安全隔离机制。

**替代原因与验证。** 如果修复后仍 Pending，重新看新事件，查资源 requests、必需 affinity、PVC 拓扑、端口及节点状态，不假设只有一个阻碍。`kubectl -n trading get pods -o wide` 应显示 Pod 到符合意图的节点，随后检查 Ready、重启数、应用请求和延迟。Running 只说明已经启动，不等于可服务；还要确认其他负载没有因误改节点边界被迁入。

### B03：新版本连接数暴涨

**排障主线：为什么连接变多 → 为什么新连接失败 → 如何恢复并验证。**

题目有两个需要解释的现象：请求量没变，新建连接率却增加 20 倍；同时出现大量 TIME_WAIT 和 `Cannot assign requested address`。优先怀疑新版连接复用下降，但“连接变多”不等于已经证明“端口耗尽”，这两件事需要分别验证。

以下命令默认在**发起请求的客户端所在网络命名空间**执行。`<server-ip>`、`<interface>` 要替换成实际服务器 IP 和网卡名。

**第一步：确认连接增长确实与新版有关。**

对齐发布时间和异常开始时间，在相近负载下比较新旧版本的请求量、新建连接率、错误率与延迟，并检查实例数、目标地址和重试次数是否变化。业务请求量不变，重试后的请求尝试数仍可能增加。

```bash
ss -s
ss -tan dst <server-ip>:443
ss -tan state time-wait
```

| 命令 | 做什么、重点看什么 |
| --- | --- |
| `ss -s` | 查看 socket 汇总，比较已建立连接和 TIME_WAIT 的数量变化 |
| `ss -tan dst <server-ip>:443` | 查看指定服务的 TCP 连接，确认异常是否集中在该目标；ESTAB 是已建立，SYN-SENT 是等待握手，TIME-WAIT 是关闭后保留的状态 |
| `ss -tan state time-wait` | 只看本地 TIME_WAIT，通过本地和对端的 IP、端口判断它们主要来自哪些连接 |

其中 `-t` 表示 TCP，`-a` 包含各状态，`-n` 用数字显示地址和端口；`dst` 筛选对端，`state` 筛选状态。`ss` 是当前状态的快照，不能直接给出每秒新建连接数，后者要结合应用指标或观测数据。

TIME_WAIT 通常出现在主动关闭的一端，不固定属于客户端或服务端。服务器执行 `ss` 只能看到自己的本地状态，不能看到远端客户端保留的 TIME_WAIT；业务服务器调用下游时，在这条连接中也是客户端。TIME_WAIT 多只能说明近期有较多连接关闭，不能单独证明故障。

**第二步：验证复用是否下降，找到新版的具体变化。**

比较同一时间窗口的“成功请求数 / 新建 TCP 连接数”：例如同样每秒成功 1000 次，旧版新建 10 条连接，新版新建 200 条，比值从 100 降到 5，支持“同样业务需要更多新连接”的判断。结合连接池命中率、活跃/空闲连接数继续核对：

- 是否每次请求都创建并销毁 HTTP client，导致连接池无法持续复用？
- 响应体是否按所用库的要求消费和关闭，连接能否归还连接池？
- keep-alive、最大连接数、空闲超时或协议是否变化？服务端是否要求关闭连接？

指标不足时，可以抓包确认握手行为：

```bash
sudo tcpdump -ni <interface> -c 200 \
  'host <server-ip> and tcp port 443 and (tcp[tcpflags] & tcp-syn != 0)'
```

`sudo` 提供抓包权限；`-n` 不解析名称，`-i` 指定网卡，`-c 200` 在捕获 200 个匹配包后停止；反斜杠表示续行。引号内筛选与目标 IP、TCP 443 端口相关且 SYN 位被设置的报文。

**判断：** 不同连接的握手增加，支持新建连接增多；同一连接反复重传 SYN，则需检查网络或服务端响应。过滤结果包含 SYN、SYN-ACK 和重传，不能把包数当成成功建立的连接数。流量不足时命令会继续等待，可按 `Ctrl+C` 停止。

**第三步：针对报错，查清新连接在哪里失败。**

`Cannot assign requested address` 要先从客户端本地查起。结合日志，必要时短时跟踪单实例的系统调用：`bind` 是绑定本地 IP/端口，`connect` 是向目标发起连接；记录具体失败位置和错误码。

**先排除源 IP 配置错误。** 通常应用只指定目标服务器，由操作系统选择本地源 IP；也有应用在代码或配置中指定必须使用哪个本地 IP。若新版误用了其他机器的源 IP，绑定就可能失败。这是待排除的备选原因，不是题目已知事实。

```bash
ip addr show
ip route get <server-ip>
```

`ip addr show` 列出本机实际配置的 IP，用来与应用的源地址配置对照。`ip route get` 查询访问目标通常使用的源 IP（src）、网卡（dev）和网关（via）；它不建立连接，也不证明目标可达。若应用显式绑定地址或使用特殊路由策略，还要匹配实际条件核对。

**源 IP 正确，再验证是否存在本地端口压力。** 客户端连接也需要源端口，例如 `10.0.0.5:42001 → 服务器:443`。频繁连接同一个目标，已有连接和 TIME_WAIT 可能限制新连接可用的端口。

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

这条命令读取内核自动分配临时端口的范围，例如 `32768 60999`，不是剩余端口数。结合 `ss` 中同一源 IP 到该目标 IP/端口的连接占用，以及实际端口分配失败证据判断。不同目标可能复用同一源端口，不能用全机连接总数或 TIME_WAIT 数量直接认定端口耗尽。

原因是 TCP 连接由四项共同区分：**源 IP、源端口、目标 IP、目标端口**，称为“四元组”。例如：

```text
连接 A：10.0.0.5:40000 → 服务器甲:443
连接 B：10.0.0.5:40000 → 服务器乙:443
```

两条连接使用相同的本地源端口 `40000`，但目标 IP 不同，系统仍能区分它们。具体能否这样使用，还受程序绑定方式等条件影响，所以说“可能复用”。

因此，临时端口范围即使只有约 28000 个，全机有 30000 条连接也不一定耗尽：连接可能分散到多个目标，部分使用相同的源端口。**大量连接分散访问不同服务器，与集中访问同一服务器的同一端口，对源端口的压力可能不同。** TIME_WAIT 也要按源 IP、目标 IP 和目标端口分析，再结合实际分配失败判断，不能只数全机总量。

**确认 Pod 到目标的实际网络路径，再检查沿途的 SNAT。** Pod 流量不一定经过 NAT，要区分访问目标和集群网络配置：

| 访问目标 | 常见路径与转换方式 |
| --- | --- |
| 同集群的 Pod IP | 通常直接路由；跨节点也不代表一定经过 NAT |
| Service ClusterIP | 常见实现通过 DNAT 将目标地址改成后端 Pod 地址，具体取决于数据平面实现 |
| 集群外部地址 | 常见做法是通过节点或出口网关做 SNAT，将 Pod 源地址改为对外地址；也可能直接路由 Pod 地址 |

这里要排查的“端口不足”，主要是**出站 SNAT 的地址/端口分配压力**，不能仅凭访问 Service 时存在 DNAT 就认定有该问题。Pod 访问外部 API 的路径可能是：

```text
Pod IP:源端口 → 节点 SNAT → 节点 IP:转换后端口
             → 云 NAT 网关 → 公网 IP:转换后端口 → 外部 API:443
```

先根据实际 CNI、出站规则和网关配置确认经过哪些转换层；若存在节点 SNAT、出口网关或云 NAT，就与 Pod 本地检查一起核对这些层的端口分配失败、容量、连接跟踪和丢弃指标。Pod 内的 `ss` 只能看到本地连接，看不到后续转换层的端口分配。NAT 问题常表现为超时，不能仅凭本地的 `Cannot assign requested address` 就归因于 NAT。

**第四步：按原因修复，并验证整条证据链。**

若新版与故障高度相关且已影响业务，先受控回滚并限制重试。复用问题修复 client 生命周期、响应体处理或连接池配置；源地址错误修正绑定配置；NAT 问题按设备证据处理。控制变更范围，不首先调整 TIME_WAIT 内核参数；扩端口范围或增加源地址只作为确认容量不足后的受控缓解。

在相近请求率下，确认新建连接率下降、复用恢复、地址错误消失、端口/NAT 压力回落，同时请求成功率、p99 延迟及 TLS 开销正常。TIME_WAIT 可逐渐回落，不要求立即归零。

最终应能说明：**新版哪项变化导致连接复用下降；连接失败的直接原因是什么；回滚或修复后，连接行为和业务指标是否一起恢复。**

## C：26 分

校验 4，同时间覆盖与排序 6，连续段/阈值/触发时间 8，恢复与未恢复输出 4，测试与复杂度 4。

```python
import re
import sys
from collections import defaultdict

num_re = re.compile(r"[0-9]{1,12}")
name_re = re.compile(r"[a-z][a-z0-9-]{0,31}")
records = defaultdict(dict)
for line in sys.stdin:
    fields = line.split()
    if len(fields) != 3:
        continue
    timestamp, service, result = fields
    if not (num_re.fullmatch(timestamp) and name_re.fullmatch(service)
            and result in {"OK", "FAIL"}):
        continue
    records[service][int(timestamp)] = result

for service in sorted(records):
    count = 0
    start = trigger = None
    for timestamp, result in sorted(records[service].items()):
        if result == "FAIL":
            count += 1
            if count == 1:
                start = timestamp
            if count == 3:
                trigger = timestamp
        else:
            if count >= 3:
                print(service, start, trigger, timestamp, count)
            count = 0
            start = trigger = None
    if count >= 3:
        print(service, start, trigger, "-", count)
```

n 行输入，字段长度有界，时间 O(n log n)，空间 O(n)。同时间覆盖发生在排序与连续段计算之前；前导零时间按数值归为同一个键。第三次失败只设一次 trigger，第四次以后只增加 count。时间间隔不打断失败段，只有 OK 打断。非法行不得覆盖有效记录。

## D：20 分

参数校验 5，引用与错误传播 4，同目录临时文件及原子替换 5，清理 2，验证与持久性说明 4。

```bash
#!/bin/bash
set -euo pipefail
export LC_ALL=C
if (( $# != 1 )); then
    printf 'usage: backup NAME\n' >&2
    exit 2
fi
name=$1
if [[ ! $name =~ ^[a-zA-Z0-9._-]{1,64}$ || $name == . || $name == .. ]]; then
    printf 'invalid name\n' >&2
    exit 2
fi
tmp=''
trap 'if [[ -n $tmp ]]; then rm -f -- "$tmp"; fi' EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
tmp=$(mktemp /backup/.backup.XXXXXXXXXX)
cp -- /srv/app/config.json "$tmp"
mv -fT -- "$tmp" "/backup/$name"
tmp=''
printf 'backup-ok\n'
```

GNU `mv -T` 避免把目标目录当作复制目的目录；目标为目录时失败。同文件系统 rename 让读者看到旧或新文件，但断电持久性还需正确的文件及目录 fsync 等机制，本脚本不提供该承诺。验证：正常新建及覆盖；空参数/多参数/非法名称；源不可读或复制中失败时旧备份保留；目标为目录或替换失败时不报成功并清理临时文件。SIGKILL/断电无法执行 trap。
