# 第三层：功能验证

## 最小有效范围

| 变更 | 验证 |
|---|---|
| 空格、描述、无宏的注释 | 相关静态检查，无需 OBS |
| Patch 编号/重命名 | 引用、内容一致性、顺序；应用顺序改变再验证应用 |
| 改为 patchlist 或移动声明段 | 查询 RPM 的补丁列表及 BuildRequires/Provides/Requires，确认没有声明被当成补丁条目；不能只看 pre-commit |
| Source/tag/checksum | 校验变化资源、解压/补丁应用、相关包 OBS |
| Requires/Provides | 精确求解和文件证据，相关包及必要 consumer OBS |
| 宏、构建阶段、源码 patch | 展开行为、该包 OBS 完整既定测试 |
| 拆合包、major API、宏更新 | 布局/冲突、受影响 consumer；宏更新选代表性包后按风险扩大 |

用户明确要求全量测试时按其范围执行。节省的是无关包与重复构建，不是擅自缩窄包内测试。

## 证据不能混用

- 格式 hook 通过不能证明 RPM 识别了依赖。涉及声明段边界时使用目标 RPM 的元数据查询，核对能力名称和版本；`rpmspec -P` 的文本中仍出现声明，不代表声明已进入包头。
- 本地 RPM 不认识 openRuyi BuildSystem 时不能宣称原 SPEC 解析通过。临时移除该字段、补占位宏的副本仅可用于隔离语法问题，必须注明验证限制，真实构建仍以 OBS 为准。
- 本地 Go Modules 测试可能使用 go.mod 锁定的旧依赖，而 OBS 使用仓库依赖。临时 modfile 的测试应报告实际版本，不把人为选择的版本称作仓库当前版本；完整确认需构建日志支撑。
- 测试进程退出且退出码为零才算通过，部分包输出 ok 不代表全量完成。超时、取消或仍在运行都不能报成功。
- 拆分 patch 时比较同一原始源码应用旧/新补丁后的完整树；仅邮件信息变化且源码树一致可复用源码测试，但 SPEC 段落变化仍需单独验证 RPM 元数据。
- OBS service 返回 ok 仅代表接受请求；需确认新 revision/srcmd5 和实际内容，再查看对应架构构建结果，不能把旧 succeeded 或 GitHub MERGEABLE 当作本次构建通过。

## 复用条件

记录源文件/patch 校验值、有效 spec、宏/Go 版本、OBS revision/srcmd5、项目 meta、依赖环境、架构和日志时间。
不能用随意去空格的哈希证明 spec 语义相同；字段和 shell 内容可能依赖空白。
仅 rebase/cherry-pick 且内容、环境一致可引用旧结果，但明确它是历史验证，不是当前 head 新跑的 CI。
仓库/工具链变更按受影响依赖使结果失效；环境无法确认时写“历史通过，当前未验证”。

## OBS 流程

1. 复用合适项目和 checkout，先核对 meta。模拟 GitHub CI 时仅依赖 openRuyi；用户授权的串联项目测试须标为集成验证，不能声称独立 CI 可过。
2. 要求“OBS 通过再推 PR”时直接上传待测 spec/patch，或用已授权测试分支，不能先推未验证修复到 PR。
3. 更新变化包，手动 trigger service，确认取得待测内容；不能把旧 succeeded 当新内容成功。
4. 日志保存一次，先读失败摘要及上下文，按需展开，不重复输出百万行测试日志。
5. 按本批用户要求 amd64 通过即可；riscv64 分别报告。未测不称大概率成功，不用本地 pbuild 替代 OBS。

## 失败与等待

- broken：查服务/元数据原因，不能默认都是未 trigger。瞬态错误可重试一次，反复同错改查根因。
- failed：查实际阶段和断言，不止末尾 Bad exit status。
- unresolvable：核对精确 capability、provider、合并/发布状态、项目配置，不直接断言都因 DAG 未合并。
- blocked/scheduled/building：有限等待并推进其它工作，不高频轮询或重复 trigger。

没有新变化或证据不重建；外部阻塞保留日志和下一步，不无限重试。格式通过和功能通过分别报告；纯格式任务写“功能测试未运行，本次无需”即可完成。
