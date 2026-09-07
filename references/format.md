# 第一层：格式与 maintainer 常见意见

只查格式时读本文件即可。先读取当前仓库 hooks；其未覆盖的字段顺序、表达、注释等仍逐包检查。

## 字段布局

普通 Go 库的顺序：SPDX、基础宏、测试宏、Name/Version/Release/Summary/License/URL/可选 VCS、Source、BuildArch、BuildSystem、Patch、BuildOption、BuildRequires、Provides/Requires 等、description、必要阶段、files、changelog。

下面仅演示排版，不要因此新增 patch 或测试参数：

```spec
BuildArch:           noarch
BuildSystem:         golangmodules

# Explain the patch purpose or give its upstream PR URL.
Patch2000:           2000-example.patch

BuildOption(check):  -short

BuildRequires:       go
BuildRequires:       go-rpm-macros

Provides:            go(%{go_import_path}) = %{version}

Requires:            go(example.org/dependency)
```

| 检查点 | 常见 review 意见与要求 |
|---|---|
| BuildArch | 最后一个 Source 之后、BuildSystem 之前，不能放依赖块里 |
| Patch | BuildSystem 后、BuildOption 前；没有 BuildOption 就在 BuildRequires 前，不能直接紧跟 Source 放到 BuildSystem 上方 |
| BuildOption | 按阶段排列，与 patch、依赖块用空行分开 |
| 对齐 | 整份 SPEC 的声明值共享一个起始列，按最长声明前缀确定；不用 Tab，具体算法见下节 |
| Source | HTTP(S) 来源前紧贴有效 `#!RemoteAsset:  sha256:...`；每条分别检查；Git/CreateArchive 按专门格式 |
| 宏位置 | 基本宏和 commit 在前，测试宏紧随其后；注释贴近作用对象；不用一次性 go_source_subdir 宏 |
| 版本引用 | 检查 Source/prep 内写死的重复版本；归一化版本不等于上游 tag 时，先判断再用宏 |
| files | 本批惯例为 doc、license、安装路径；检查 README/LICENSE 文件是否实际存在 |
| Release/changelog | 使用 `%autorelease` 和 `%autochangelog` |
| 空白 | 段落间空行，清理尾随空白，文件末尾换行，避免无关排版变动 |
| SPDX | 保留既有版权署名；普通小改不新增用户 FileContributor |

## 整份文件对齐

这是本用户要求的审查约定，不把它表述成官方文档规定的固定列宽。每份 SPEC 单独选择一个声明值起始列，空行、注释、条件分支及子包边界不重置列宽。普通字段、宏和 BuildOption 不能各自使用不同的值列。

纳入普通标签、Source/Patch、BuildOption、依赖字段、`#!RemoteAsset:` 和声明型 `%define`/`%global`。普通标签的前缀截至冒号；宏的前缀截至宏名称（含参数声明，如有），宏指令和名称之间保留一个空格。宏体、字段值的长度不参与计算；不能用匹配任意冒号的正则把 URL、注释或 shell 命令误当声明。

列号从 1 开始，使用空格，按原文件文本计算显示宽度，不展开 RPM 宏：

```text
P_i = 第 i 个声明的前缀
W   = 所有 P_i 的最大显示宽度
C   = W + 3
S_i = C - width(P_i) - 1
输出 = P_i + S_i 个空格 + 原值
```

最长前缀后至少两个空格，所有声明值均从 C 列开始。已有文件若所有声明已经统一在更宽且合规的列，可保留该列。新增长字段导致原列容不下时，整份文件一起右移；不能仅给长字段留一个空格作为例外。以下完整声明片段由宏前缀决定列宽：

```spec
%define _name           example
%define go_import_path  github.com/example/example

Name:                   go-github-example-example
Version:                1.0.0
Release:                %autorelease
BuildArch:              noarch
BuildSystem:            golangmodules

BuildOption(check):     -vet=off

BuildRequires:          go
BuildRequires:          go-rpm-macros

Provides:               go(%{go_import_path}) = %{version}
```

语法边界：description 正文按自然段排版；构建阶段的 shell 命令按控制结构缩进；files、patchlist 条目保留各自格式；多行宏只对齐声明首行，宏体列表保留层级缩进；SPDX 和普通说明注释不参与。不能改动 shell here-document、续行、字符串或宏体中的空白来凑声明列，也不能改名称、值、Patch 编号或依赖顺序来实现对齐。

验收只针对本次目标文件：报告目标列、偏离行号和实际列；确认目标文件集合非空；比较修改前后的声明前缀和值，确保仅声明分隔空白改变，其他内容保持不变。再运行相关格式 hook 并检查最终 diff。当前仓库 format-spacing 对 BuildRequires/BuildOption 检查的是冒号后至少两个空格，不是只能两个；执行时仍核对目标 revision 的 hook。纯对齐不下载源码、不触发 OBS，缓存检查结果时记录对齐规则版本；本节规则版本为 2。

AI 常错点：hook 通过不代表整体对齐；不能只修 review 指向的一行；新增长字段后必须重新计算全文件列宽；不能把未匹配到任何文件的检查当作通过。

## Patch 编号与格式

- 文件前四位表示类型：0001–0999 同版本 upstream；1000–1999 CVE/跨版本 backport；2000–2999 下游或未被 upstream 接受的修改。PR 已提交不等于已接受；合并后不只因状态变化重命名旧补丁。
- 本用户曾要求 `Patch1: 0001-...`、`Patch2000: 2000-...`。按当前任务约定核对索引、文件名、应用顺序，尤其第二个 patch 和重命名后的旧引用。RPM 允许 `Patch0: 0001-...`，官方示例也可能如此，不得虚称语法错误。
- 超过三个 patch 使用 `%patchlist`，放 description 上方，核对隐式顺序与逐项注释。
- 每项上方写目的或直接链接：`# https://github.com/owner/repo/pull/123`。不用额外 `Upstream:` 前缀，在SPDX署名一致的时候，不给普通注释加署名。
- 本地补丁用 git format-patch 生成；检查邮件头、真实作者、subject、说明、路径和 diff，不靠手工补 From 头冒充生成过程。
- 编号/文件名改变但 diff 不变时，仅查引用、顺序和内容一致性。重新生成改变了 hunk 时再升级语义检查。
- 保留上游真实作者，不伪造来源链接。说明补丁作用、API 影响，不能把 before/after 标反。

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
