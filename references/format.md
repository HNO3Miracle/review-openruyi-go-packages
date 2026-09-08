# 第一层：格式与 maintainer 常见意见

只查格式时读本文件即可。先读取当前仓库 hooks；其未覆盖的字段顺序、表达、注释等仍逐包检查。

## 字段布局

普通 Go 库的顺序：SPDX、基础宏、测试宏、Name/Version/Release/Summary/License/URL/可选 VCS、Source、BuildArch、BuildSystem、Patch、BuildOption、BuildRequires、Provides/Requires 等、description、必要阶段、files、changelog。

下面仅演示排版，不要因此新增 patch 或测试参数：

```spec
BuildArch:      noarch
BuildSystem:    golangmodules

# Explain the patch purpose or give its upstream PR URL.
Patch2000:      2000-example.patch

BuildOption(check):  -short

BuildRequires:  go
BuildRequires:  go-rpm-macros

Provides:       go(%{go_import_path}) = %{version}

Requires:       go(example.org/dependency)
```

| 检查点 | 常见 review 意见与要求 |
|---|---|
| BuildArch | 最后一个 Source 之后、BuildSystem 之前，不能放依赖块里 |
| Patch | BuildSystem 后、BuildOption 前；没有 BuildOption 就在 BuildRequires 前，不能直接紧跟 Source 放到 BuildSystem 上方 |
| BuildOption | 按阶段排列，与 patch、依赖块用空行分开 |
| 对齐 | 遵循仓库和 go2spec 的固定模板，不按单个文件的最长前缀整体右移；不用 Tab，具体算法见下节 |
| Source | HTTP(S) 来源前紧贴有效 `#!RemoteAsset:  sha256:...`；每条分别检查；Git/CreateArchive 按专门格式 |
| 宏位置 | 基本宏和 commit 在前，测试宏紧随其后；注释贴近作用对象；不用一次性 go_source_subdir 宏 |
| 版本引用 | 检查 Source/prep 内写死的重复版本；归一化版本不等于上游 tag 时，先判断再用宏 |
| files | 本批惯例为 doc、license、安装路径；检查 README/LICENSE 文件是否实际存在 |
| Release/changelog | 使用 `%autorelease` 和 `%autochangelog` |
| 空白 | 段落间空行，清理尾随空白，文件末尾换行，避免无关排版变动 |
| SPDX | 保留既有版权署名；普通小改不新增用户 FileContributor |

## 声明字段对齐

按仓库现有 Go SPEC 与 go2spec 的固定模板排版，不根据单个文件的最长声明前缀整体右移。普通 RPM 字段以第 17 列为首选值列；`%define` 和 `%global` 的宏值以第 25 列为首选值列。若前缀已经占到或超过目标列，则在前缀和值之间保留至少两个空格。语义相关的相邻自定义宏组可以按组内最长前缀统一对齐，但不能因此移动基础宏、普通字段或其它宏组。

普通字段的前缀截至冒号，宏的前缀截至宏名称（含参数声明，如有）；宏指令和名称之间只保留一个空格。只调整声明和值之间的空格，不把 URL、注释、description、shell 命令、宏体或 `%files` 条目当成声明。

列号从 1 开始，使用空格，不展开 RPM 宏：

```text
普通字段空格数 = max(2, 16 - width(字段前缀))
连续宏组目标列 = max(25, 组内最长宏前缀宽度 + 3)
宏定义空格数   = 连续宏组目标列 - width("%define 名称" 或 "%global 名称") - 1
```

因此普通短字段对齐到第 17 列；`BuildOption(...)` 等长字段通常在冒号后保留两个空格。宏定义独立使用第 25 列，并不要求与普通字段共享值列：

```spec
%define _name           example
%define go_import_path  github.com/example/example

Name:           go-github-example-example
Version:        1.0.0
Release:        %autorelease
BuildArch:      noarch
BuildSystem:    golangmodules

BuildOption(check):  -vet=off

BuildRequires:  go
BuildRequires:  go-rpm-macros

Provides:       go(%{go_import_path}) = %{version}
```

`#!RemoteAsset:` 由专用 hook 解析，必须保持字面格式 `#!RemoteAsset:  sha256:<64 位小写十六进制>`。这两个空格也让 `sha256` 从第 17 列开始。go2spec 生成的裸 `#!RemoteAsset` 只是占位符，提交前必须使用 `remoteassetify.py` 补齐；仓库中未修改的历史裸标记不能作为新文件或修改文件的示例。

语法边界：description 正文按自然段排版；构建阶段的 shell 命令按控制结构缩进；files、patchlist 条目保留各自格式；多行宏只对齐声明首行，宏体列表保留层级缩进；SPDX 和普通说明注释不参与。不能改动 shell here-document、续行、字符串或宏体中的空白来凑声明列，也不能改名称、值、Patch 编号或依赖顺序来实现对齐。

验收只针对本次目标文件：报告偏离固定模板的行号和实际列；确认目标文件集合非空；比较修改前后的声明前缀和值，确保仅声明分隔空白改变，其他内容保持不变。单独确认每个 RemoteAsset 指令满足固定语法，再运行相关格式 hook 并检查最终 diff。当前仓库 `format-spacing` 对 BuildRequires/BuildOption 只检查冒号后至少两个空格，但审阅仍应对齐 go2spec 模板。纯对齐不下载源码、不触发 OBS，缓存检查结果时记录对齐规则版本；本节规则版本为 4。

AI 常错点：hook 通过不代表符合仓库排版；不能只修 review 指向的一行；不能因出现较长的 BuildOption 或宏名就把整份文件全部右移；也不能让同一组自定义版本/目录宏的值散落在不同列；不能给 RemoteAsset 增减空格；不能把未匹配到任何文件的检查当作通过。

## Patch 编号与格式

- 文件前四位表示类型：0001–0999 同版本 upstream；1000–1999 CVE/跨版本 backport；2000–2999 下游或未被 upstream 接受的修改。PR 已提交不等于已接受；合并后不只因状态变化重命名旧补丁。
- 本用户要求 Patch 标签编号对应文件名前四位去除前导零后的数值：`Patch1: 0001-...`、`Patch2: 0002-...`、`Patch2000: 2000-...`。逐项机械校验这一对应关系、引用存在性和应用顺序。RPM 语法本身允许 `Patch0: 0001-...`，但它不符合本仓任务采用的编号约定，说明时不能虚称为 RPM 语法错误。
- 超过三个 patch 使用 `%patchlist`，放所有头部字段和依赖声明之后、`%description` 之前。列表内只放补丁条目和相关注释；不能把 Patch 标签的放置规则套用到列表段，更不能在列表后继续写 BuildRequires/Provides/Requires。核对隐式顺序与逐项注释。
- 每项上方写目的或直接链接：`# https://github.com/owner/repo/pull/123`。不用额外 `Upstream:` 前缀，在SPDX署名一致的时候，不给普通注释加署名。
- 本地补丁用 git format-patch 生成；检查邮件头、真实作者、subject、说明、路径和 diff，不靠手工补 From 头冒充生成过程。
- 编号/文件名改变但 diff 不变时，仅查引用、顺序和内容一致性。重新生成改变了 hunk 时再升级语义检查。
- 任何时候都不能删除原作者信息。无论 backport、其他发行版、邮件列表或其他来源，均保留原始 From 姓名/邮箱、作者日期、Co-authored-by、Signed-off-by、版权及来源记录；不能把原作者改成打包者，也不能只恢复姓名而丢失日期和签名。
- 原始提交说明应保留，本地适配另追加说明；只有实际参与适配时才按项目规则追加自己的记录，不能取代原记录。来源不明先查明，不能猜作者或伪造来源链接。
- 多个上游提交回移时逐提交拆成独立 patch，用 git format-patch 生成，跨版本使用 1000–1999。只回移部分文件或调整上下文时注明范围，避免误带 module 路径及依赖升级；保留每个原作者，并比较拆分前后应用后的完整源码树。
- 说明补丁作用、API 影响，不能把 before/after 标反。

## 描述与注释

- Summary 简短、无尾随句号、无重复文本，不塞完整功能清单。description 描述本包职责，精简复制来的 README/营销文案。
- 已有描述正确或用户要求保留时不自行重写。
- 自定义 check、环境变量、exclude/include、特殊 BuildOption 注释说明具体原因，不写“为了通过 CI”这样的空话。
- 测试专用外部工具按需归到 `# For tests` 下；示例里的 git/gnupg 不是必须添加的依赖。
- RPM 注释中的宏也可能展开；字面宏检查是否需 `%%` 转义，尤其 `%go_common`。不能把此类变动一概视作纯注释。

## AI 常错点与提交检查

- hook 或 OBS 通过不豁免人工格式检查。
- 不因 BuildSystem 名为 golangmodules 就认定使用 Go Modules 模式，不擅自补全手写 prep/build/install/check。
- `%check -p` 是前置、`-a` 是追加；改它们需语义检查。
- “按 go2spec 对齐”不等于重新生成所有包；格式审查不自动重命名、降版本或补猜测 Provides。
- 只跑变化文件的相关 hook，不使用全仓 `--all-files`。纯 review 防止修改型 hook 污染工作树；修复后检查 hook 是否擅自加署名或无关内容。
- 已授权整理提交时，一包一提交、Signed-off-by、无 fixup/squash。修改 metadata 用 `SPECS: 包名: 修改说明.`，不是 Add。改写历史先核对远端 head，再用明确 lease 推送。
- PR description 遵守目标模板，用户说不改就不改；格式检查不自动发布评论或 upstream PR。
- 回复 comment 产生任何公众能看得到的交流前，需经过用户同意。

## 历史保护与防止回归

1. 开始修改时记录实际 PR head、base、本地 HEAD、工作树状态及目标文件差异。远端跟踪引用也可能过期，应与 GitHub/远端核对。恢复任务时先检查现状，不能仅相信旧总结。
2. 本地与 PR diverged 时先比较提交和文件内容，识别已合入、重写及仅本地的修改；不得将较短历史直接强推覆盖 PR，也不能仅凭提交数决定采用哪条历史。需重建时从已确认的真实 head 或经逐项核验的基线开始。
3. 一包一提交、无 fixup 的整理不意味着重做旧修复。先记录本次允许变化的包和已接受的特殊处理；完成后对照修改前的 PR head，检查目标包最终内容、非目标路径差异，以及历史重排的提交对应关系。任何超出预期的恢复、删除、降级都需查明。
4. 推送前备份被重写的本地引用，使用针对已核对远端 SHA 的明确 force-with-lease。lease 失败先查远端新增内容，不能更新 lease 后盲推。
5. 推送后核对 GitHub head，并同步将继续使用的本地分支和 worktree。不能远端已修好，本地仍停在旧历史，导致下轮再次覆盖。保护未提交修改和仅本地提交，确认后再移动分支引用。
6. 可合并状态只证明 Git 没有合并冲突，不能证明旧修复未丢失或 CI 能过。对新旧内容和验证证据分别报告，不把本轮有限检查扩大为所有分支无回归。
