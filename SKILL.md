---
name: review-openruyi-go-packages
description: 逐包检查或修正 openRuyi Go SPEC 格式与 maintainer 常见审阅问题。适用于包格式审查、Patch 排版编号、PR review 意见处理；按需升级到依赖与布局审查、OBS 功能验证，复用已有证据，避免每次全量测试。
---

# openRuyi Go 包分层审查

默认从格式层开始。格式本身就是交付要求；构建成功不能豁免格式问题。中文汇报，spec 描述与注释使用简洁美式英语。

## 层级与完成边界

| 请求或变更 | 读取参考 | 完成边界 |
|---|---|---|
| 格式、空格、描述、注释、字段位置 | [格式检查](references/format.md) | 逐包格式检查及相关静态检查，不触发 OBS |
| Patch 编号、文件名、邮件格式 | 格式检查 | 核对引用、顺序和 diff；内容改变再升级 |
| 依赖、版本、安装布局、测试参数 | 格式检查 + [语义检查](references/semantics.md) | 判断实际影响，列出需要验证的包 |
| 构建失败、明确要求 OBS、行为变更验证 | 按需读 [功能验证](references/validation.md) | 验证受影响包和必要 consumer |

不要一次读取全部参考。只要求格式时，语义风险记录为待查，不能自动扩成升级、拆包或全仓修复。
“检查/review”默认只报告；“改/修复”才修改。提交、重写历史、推送、修改 OBS、发布 upstream PR 沿用用户实际授权，skill 本身不提供授权。

## 逐包流程

1. 确认工作树和范围。PR 使用实际 base/head SHA，不能把 upstream/main 当 head；保护用户已有改动。
2. 首次读取目标仓库 instructions、相关格式 hooks 和本次 PR 行内评论。后续只获取新增意见。
3. 第一次逐个完整读 spec；检查 patch 时读完整补丁。清单脚本和 formatter 可辅助，不能替代逐文件判断。
4. 按层级处理每个包；记录已查、问题及未决项。独立读取可并行，共享文件修改和历史重写顺序执行；等待 OBS 时推进其它独立包。
5. 格式修复集中完成后，对变化文件运行相关静态检查，并读最终 diff。没有新变化或失败理由不重复运行。
6. 汇报已审/剩余包和证据，分别写“格式通过”“语义已查”“OBS 通过”。格式任务不因未跑功能测试而无法完成。

## 复用工作

多包或跨轮任务复用已有工作记录；没有时在非提交工作记录中保存下表，不放进 SPECS。单包小改无需额外造记录文件。

| 包 | 内容标识 | 层级/结论 | 证据 | 待办 |
|---|---|---|---|---|
| 包名 | spec/patch blob 或 SHA256 | 格式通过，功能未测 | hook 结果/日志位置 | 无 |

- 附检查规则版本、文档 revision/日期、评论检查时间；规则更新只使对应检查失效。
- 按文件内容判定复查范围，rebase/cherry-pick 改 commit ID 不意味着全部重查。
- 有可靠记录且 Source/校验值未变，不重新下载；provider 索引按仓库 revision 缓存，后续增量更新。
- 功能结果额外绑定宏、工具链、依赖环境，按功能验证参考判断可复用性。
- 纯格式只复查格式，但含 RPM 宏的注释、续行、shell 引号、字段归属、段落顺序变化需确认是否改变行为。

## 依据

首次按任务读取相关最新规范或可靠缓存，同一版本不逐包重复下载。补充规范优先于主规范；旧 spec、go2spec 输出只是对照，不能视为必然正确。
文档、hook 和 maintainer 意见冲突时指出具体差异；不能把某 PR 的特例推广到所有包。

- https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/
- https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/languages/Golang/
- https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/Patch/
