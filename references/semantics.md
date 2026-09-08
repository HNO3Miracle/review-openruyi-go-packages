# 第二层：打包语义

只展开受影响项。格式任务发现疑点可直接记为待查；不默认启动功能测试。

## 版本与边界

- 本用户的打包策略：能打完整大包时必须打大包，即使引入很多直接、间接或测试依赖。打包及包边界调整任务须审查完整 VCS 仓库的嵌套 go.mod 与实际安装路径，不得只打 Prometheus 当前用到的 module。
- 依赖多、OBS 暂缺依赖或为了让 CI 快过，都不是拆小包、少装源码或缩减测试的理由；补齐依赖并安排相关 producer 的提交和构建。
- 判断能否合并依据实际版本/API 兼容性、安装路径、重复文件归属及 consumer，而非方便程度。确有技术阻碍时给出具体证据和待解决项，不能自行将不完整打包当完成。纯格式任务仍不自动扩展为全仓拆合。
- 大包 Version 表示源码快照，嵌套 module 可有独立上游版本；能力版本遵循当前 Go 规范。
- 新增前查实际 provider、源码布局及相关 PR，不能只搜 RPM 包名：无 v5 后缀的包也可能已提供 v5。
- 稳定 release 优先；格式任务不顺便升级。降级先查原因与 consumer API，不自动把所有 v1 consumer 改为 v2。
- 新快照日期用打包日期，不是 commit 日期；已有 release 的快照保留版本前缀。只改格式不刷新已有快照日期。
- 核对归档与安装位置，防止 ansi/ansi、v2/v2；仓库包含不等于 RPM 已安装。

## 依赖

- 区分 VCS repo、module、package import、RPM 名称和 go(...) capability。
- 精确查变化 capability 的 producer/consumer，缓存依赖索引；不每包重扫全部分支。
- 源码确实提供但缺能力声明时按当前用户策略修 producer；没有文件不能凭空 Provides。自动生成是否上线应查当前宏/RPM，不沿用旧记忆。
- RPM 不会用 root capability 自动满足子路径 Require；改 root Require 前证明所需源码实际安装，并确认任务允许修改。
- 不删除所需间接或测试依赖换通过；测试工具归 BuildRequires，不机械复制全部测试依赖到 Requires。
- producer 变化需联动相关 consumer。只有拆分支/依赖审查任务才计算相关 DAG；约 30 包是目标，不为数字破坏依赖关系。

## 阶段与测试

- 读当前宏实现或可靠展开日志，优先默认阶段和最小 hook，不凭其它发行版经验否定声明式行为。
- 追加用 -a，环境初始化等前置才用 -p；无动作不留空 hook，真正要进入子目录就直接用目录。
- 跳包用 go_test_exclude 或 go_test_exclude_glob，不用 BuildOption(check) -e/-skip；核对实际宏语义与匹配结果。exclude 根路径未必排除了子包。
- 多个排除尽量统一表达，但不因此扩大范围。include 必须说明理由，不用 test_modules 只测 Prometheus 当前所需模块。
- 保持完整包内测试覆盖，不代表每次纯格式修改都重跑测试。
- 缺依赖/fixture 先补；外部测试归档需核对 SourceN 和解压位置，空 tests/ 可能导致 mv 嵌套。
- vet-off 只能处理实际 vet 问题；不能吞错误、删除断言，或把精确断言改成非空来换通过。快照期望变化需要行为依据。
- noarch 不代表可以无条件删 shell 依赖；证明是源码附带脚本误生成后，使用当前 RPM 机制精确过滤，保留真实运行依赖。

## 源码补丁

内容改变后按精确 Source 验证应用和 API 影响。用户要求上游化/升级或修复确有必要时，再查最新 release、commit、已有 PR，同一结果缓存复用。上游发布须有授权，且读对应模板与贡献指南。

按需规范：

- https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/Versioning/
- https://openruyi.cn/zh-Hans/docs/guide/packaging-guidelines/BuildSystems/golang/
