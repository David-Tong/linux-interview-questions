# 网上检索与适用范围

检索日期：2026-11-12。优先使用雇主与项目官方资料。以下区分岗位事实、其他岗位的面经和原创练习；没有可靠证据的内容不当作 OA 要求。

## 1. 最接近的官方职位

[Optiver：Linux Systems Engineer – Bare Metal & Kubernetes，Sydney，Experienced](https://www.optiver.com/join-us/jobs/technology/sydney/linux-systems-engineer-bare-metal-and-kubernetes/)

岗位强调裸机 Linux、机房内 Kubernetes、Cilium、Rancher、ArgoCD、Terraform，以及网络、性能排障和生产值班。这是本题库选题的主要依据，但尚不能确认就是你申请的职位。该页面没有列出 OA 的具体格式。

## 2. 官方招聘流程资料

[Optiver Europe Recruitment FAQ](https://www.optiver.com/join-us/stories/optiver-europe-recruitment-faq/)

欧洲 FAQ 说明开始测评后的重测限制、设备与连接问题处理方式。它没有披露 Linux/K8s SA 题目或考试时长，也不能据此确定其他地区的全部规则。

[公开 HackerRank 页面：Optiver Software Engineer 48h – AMS](https://www.hackerrank.com/test/4j8f39pt16p)

页面描述一个 SWE 测评的选择题与编程组合。它属于别的岗位且可能是历史配置，不能把它的 48 小时、8 题或语言限制套到 SA。本练习仅采用通用的“知识题 + 动手处理问题”结构。无需启动该测试。

## 3. 公开候选人经验：参考信号较弱

[Reddit：Trading Operations Analyst Interview，2023](https://www.reddit.com/r/csMajors/comments/16zxfio/)

有匿名候选人提到 Linux、日志解析和交易系统诊断；岗位是 Trading Operations Analyst，无法独立核实，不能作为 SA OA 的题型证明。本套题的日志练习与此方向相关，但题目完全原创。

[Glassdoor：Software Engineering Intern 面经](https://www.glassdoor.co.uk/Interview/Optiver-Software-Engineering-Intern-Interview-Questions-EI_IE243355.0,7_KO8,35.htm)

候选人报告过编程和认知测试组合。实习 SWE 与有经验 SA 的流程不同，因此未将数列、概率、反应游戏列为本套题必考内容，也未采用网传通过线。

## 4. 技术答案的官方核对入口

- [Kubernetes Service 排错](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)：用于核对 Service、selector、端口与后端排查。
- [Kubernetes 容器资源管理](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)：用于核对 request、limit、CPU 限流和内存限制。
- [Cilium Troubleshooting](https://docs.cilium.io/en/stable/operations/troubleshooting/)：用于核对 agent 状态、日志和网络诊断方向；命令需匹配实际版本。

## 5. 有待你的邀请说明确认

用户提供的邮件标题 `Your HackerRank Optiver Linux Systems Engineer Invitation` 已确认岗位名称和 HackerRank 平台。这是用户提供的信息，不是网上公开来源；它不能确认具体城市，也不能证明与上述 Sydney 职位完全相同。

仍待确认：模块名称、每模块时长、能否暂停或回看、允许语言、是否有 Linux shell、是否需要集群、查资料和计算器规则、监考与设备要求、提交截止时间。

当前结论：适合先准备 Linux / 网络 / Kubernetes 排障和脚本能力；没有证据支持声称这就是你将收到的 OA 格式。若邀请明确列出认知或数学模块，再增加对应训练。
