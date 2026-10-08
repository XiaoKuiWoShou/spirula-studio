# Spirula Studio 开发指南

> 本文用于约束 AI 在本仓库中的开发行为。AI 在修改任何代码之前，必须先阅读项目开发规则和 Git 维护规则，再开始分析与实现。

## 1. 开发前必须先读

按以下顺序阅读：

1. `AGENTS.md`
   - 官方项目架构、构建、测试、CUDA/Vulkan、codegen、注释等规则。
2. `spirula_maintenance_guide.md`
   - 本仓库的 Git 分支、upstream、private/main、legacy 和功能移植规则。
3. 与本次任务直接相关的 `docs/`、子目录 `README.md` 或源码。

如果规则发生冲突：

- 代码架构、构建、测试、注释风格：以 `AGENTS.md` 为准。
- Git 分支和私人仓库维护：以 `spirula_maintenance_guide.md` 为准。
- 用户当前明确提出的要求优先于本指南中的一般工作流程。

## 2. 开发前检查 Git 状态

开始修改前必须先检查：

```bash
git status
git branch --show-current
git remote -v
```

分支约定：

```text
master
└── 只跟踪 upstream/master，不做私人开发

private/main
└── 稳定的私人长期版本

feature/*
└── 新功能

fix/*
└── Bug 修复

experiment/*
└── 实验、调试、性能测试

legacy/ccj
└── 旧仓库，只作为历史和功能来源
```

### 禁止直接在 `master` 开发

如果当前位于 `master`，需要开发时，应先切到 `private/main`，再根据任务创建分支：

```bash
git switch private/main
git switch -c feature/my-feature
```

或：

```bash
git switch private/main
git switch -c fix/my-fix
```

除非用户明确要求，否则 AI 不应自行：

- `git reset --hard`
- `git rebase`
- `git push --force`
- `git push --force-with-lease`
- 删除 branch/tag
- 向 `upstream` push
- 大规模改写 Git 历史

## 3. 跟踪官方最新版

更新官方基线：

```bash
git switch master
git fetch upstream
git merge --ff-only upstream/master
git push origin master
```

然后将官方更新带入私人版本：

```bash
git switch private/main
git merge master
```

解决冲突、编译和测试完成后：

```bash
git push origin private/main
```

长期关系：

```text
upstream/master
      ↓
    master
      ↓
 private/main
      ↓
 feature / fix / experiment
```

## 4. 从 legacy 移植旧功能

`legacy` 默认视为只读历史来源。

查看旧提交：

```bash
git log --oneline legacy/ccj
```

### 移植完整 commit

```bash
git cherry-pick <commit>
```

### 一个功能包含多个依赖 commit

按原历史顺序逐个 cherry-pick，并在每组功能后编译测试：

```bash
git cherry-pick <commit1>
git cherry-pick <commit2>
git cherry-pick <commit3>
```

### 只需要 commit 的一部分

```bash
git cherry-pick -n <commit>
```

检查并删除不需要的修改后重新提交：

```bash
git status
git add <需要的文件>
git commit -m "..."
```

### 只需要旧仓库中的单个文件

```bash
git restore --source=legacy/ccj -- path/to/file
```

然后正常提交。

### 移植前必须检查依赖

```bash
git show --stat <commit>
git show <commit>
```

不要因为只需要一个功能，就默认把整个 `legacy/ccj` merge 到当前仓库。

## 5. AI 修改代码时的基本原则

### 先理解，再修改

AI 不应根据文件名或局部搜索结果直接修改代码。

应先：

1. 找到真实调用链。
2. 阅读相关类型、配置和 backend 接口。
3. 阅读相关 subsystem 文档。
4. 判断是否存在现有实现可以复用。
5. 确认修改范围后再动代码。

### 保持最小修改

优先解决当前问题所需的最小改动。

避免顺手进行：

- 无关重构
- 大范围格式化
- 文件重命名
- API 改名
- 清理与当前任务无关的代码
- 把多个独立功能混在一个 commit 中

这样可以降低与 upstream 后续更新产生冲突的概率。

## 6. 遵守 Spirula 官方架构

必须遵守 `AGENTS.md` 的项目方向，尤其注意：

- 正式功能使用原生 C++。
- 不重新引入 PyTorch、nerfstudio、gsplat 等依赖。
- 不增加运行时 Python 依赖或 Python binding 层。
- 不为已有功能建立第二套并行实现。
- 修改训练 backend 时保持 CUDA/Vulkan 一致性。
- Vulkan-only subsystem 按 `AGENTS.md` 的例外规则处理。
- 不手动修改标记为 generated 的生成区域。
- 涉及 codegen 时使用官方生成流程。
- `TrainConfig.h` 等 single-source-of-truth 文件必须按官方规则扩展。
- `SS_ENABLE_PATENTED` 默认保持关闭。

如果修改 kernel/backend 层，应检查是否同时需要：

```text
CUDA implementation
Slang/Vulkan implementation
backend parity test
```

## 7. 注释规则

严格遵守 `AGENTS.md` 的 Comments 规则。

默认：

```text
能从代码本身看懂的内容，不写注释。
```

注释主要用于说明：

- 为什么必须这样做
- 不明显的约束/invariant
- 实测数字
- 被否决方案的原因
- 数值稳定性或平台限制

避免：

```cpp
// Loop over cameras
// Initialize buffer
// This ensures that...
// Note that...
// Previously we used...
```

复杂设计说明应优先放到对应 `docs/` 或 subsystem README。

## 8. 构建和测试

必须优先使用项目官方开发构建入口，不自行发明替代构建流程。

Windows 示例：

```bat
build_develop.bat -DSS_BACKEND=vulkan
```

其他 backend 和测试方式以 `AGENTS.md`、`docs/build.md`、`docs/testing.md` 为准。

修改后至少完成：

1. 编译受影响的目标。
2. 运行与修改相关的测试。
3. 涉及 backend 时考虑 CUDA/Vulkan parity。
4. 涉及真实工作流时进行针对性的功能验证。

如果由于当前环境无法测试某个平台或 backend：

**明确说明“未测试”，不要声称已经验证。**

## 9. 私人项目数据与路径

不要把以下内容提交进 Git：

- 本机绝对路径
- 私人 dataset 路径
- 私人采集项目名称
- 临时输出
- build 目录
- 大型训练数据
- checkpoint
- 私人凭证或 token

提交前检查：

```bash
git diff
git status
```

并按照 `AGENTS.md` 的要求运行私人路径检查工具（如适用）。

## 10. 地理坐标 / LiDAR / GPS 相关修改

本私人版本经常涉及 GPS、LiDAR、真实尺度和地理坐标，因此修改这些部分时额外检查：

- 不要无意破坏真实尺度。
- 区分 world/georeferenced coordinates 与 training-local coordinates。
- 保持 camera、point cloud、Gaussian 和 export transform 的一致性。
- 对大绝对坐标注意 float32 精度和数值稳定性。
- 如果训练内部采用 local origin/recentering，导出时必须确认地理定位能正确恢复。
- 不要仅根据“训练结果看起来正常”判断坐标变换正确，应验证相对位置、尺度和最终导出坐标。

## 11. Commit 原则

一个 commit 尽量表达一个明确目的。

推荐：

```text
Add LiDAR surface guidance
Fix LiDAR guidance scale
Add fisheye multiview loss
Add aerial training presets
```

避免：

```text
update
changes
fix stuff
my modifications
```

提交前：

```bash
git status
git diff
```

如果已经 staged：

```bash
git diff --cached
```

确保没有带入：

- debug print
- 临时测试代码
- 不相关格式化
- 本机路径
- 无关文件

## 12. AI 完成任务后的报告

每次开发完成，AI 应简要报告：

1. 修改了什么。
2. 为什么这样修改。
3. 修改了哪些主要文件。
4. 做了哪些编译/测试。
5. 哪些内容没有测试。
6. 是否存在已知风险或后续工作。
7. 当前 Git 状态。

除非用户明确要求：

- 不自动 push。
- 不自动 merge 到 `private/main`。
- 不自动删除开发分支。
- 不自动修改 `master`。

## 13. 推荐的一次完整开发流程

```bash
# 1. 确认状态
git status
git branch --show-current

# 2. 从私人稳定版创建修复分支
git switch private/main
git switch -c fix/example

# 3. 阅读
# AGENTS.md
# spirula_maintenance_guide.md
# 相关 docs / README / source

# 4. 修改代码

# 5. 编译和测试

# 6. 检查修改
git status
git diff

# 7. 提交
git add <files>
git commit -m "Fix example issue"
```

测试确认后，再由用户决定是否合入：

```bash
git switch private/main
git merge --ff-only fix/example
```

## 最核心的几条

```text
先读 AGENTS.md
        ↓
再读 spirula_maintenance_guide.md
        ↓
确认当前 Git 分支和状态
        ↓
理解相关源码和文档
        ↓
创建 feature/fix/experiment 分支
        ↓
做最小、符合项目架构的修改
        ↓
编译 + 测试
        ↓
检查 diff
        ↓
提交
        ↓
由用户决定是否 merge / push
```

目标不是“尽快写出代码”，而是：

**在持续跟踪 upstream 的前提下，让私人功能长期保持清晰、可测试、可移植、低冲突。**
