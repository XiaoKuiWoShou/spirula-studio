# Spirula Studio 私有分支维护指南

## 当前分支结构

```text
upstream
└── harry7557558/spirula-studio
    └── master          # 官方最新代码

origin
└── XiaoKuiWoShou/spirula-studio
    ├── master          # 官方代码镜像
    └── private/main    # 自己长期使用的版本

legacy
└── spirula-studio_light
    └── ccj             # 旧开发仓库，作为功能来源
```

基本原则：

- `master`：只跟踪官方，不在这里开发自己的功能。
- `private/main`：自己长期使用和维护的版本。
- `legacy/ccj`：旧代码仓库，只作为历史和功能来源。
- 新实验尽量单独开 branch，不直接在 `master` 上修改。

---

## 1. 跟踪官方最新版本

先切换到 `master`：

```bash
git switch master
```

获取官方最新代码：

```bash
git fetch upstream
```

更新本地 `master`：

```bash
git merge --ff-only upstream/master
```

同步到自己的 GitHub：

```bash
git push origin master
```

然后把官方更新合入自己的版本：

```bash
git switch private/main
git merge master
```

测试正常后：

```bash
git push origin private/main
```

日常更新流程可以记成：

```text
upstream/master
      ↓
master
      ↓
private/main
```

---

## 2. 从 legacy 移植一个完整功能

先查看旧仓库的提交：

```bash
git log --oneline legacy/ccj
```

找到需要的 commit，例如：

```text
d1fafa18 Add custom training presets
```

切到自己的版本：

```bash
git switch private/main
```

移植这个 commit：

```bash
git cherry-pick d1fafa18
```

测试正常后：

```bash
git push origin private/main
```

---

## 3. 一个功能由多个 commit 组成

如果一个功能有多个依赖 commit，按原来的顺序逐个移植：

```bash
git cherry-pick <commit1>
git cherry-pick <commit2>
git cherry-pick <commit3>
```

每移植一组功能后，建议先编译和测试，再继续移植下一组。

---

## 4. 只移植某个 commit 的部分修改

先把修改放进工作区，但不立即提交：

```bash
git cherry-pick -n <commit>
```

查看变化：

```bash
git status
```

删除不需要的修改，保留需要的部分，然后：

```bash
git add .
git commit -m "描述这次保留的功能"
```

---

## 5. 只从 legacy 取一个文件

例如只需要旧仓库中的某个 preset：

```bash
git restore --source=legacy/ccj -- presets/example.json
```

然后：

```bash
git add presets/example.json
git commit -m "Add selected preset"
```

---

## 6. 开发新功能

不要直接在 `master` 上开发。

从 `private/main` 创建新分支：

```bash
git switch private/main
git switch -c feature/my-feature
```

开发并提交：

```bash
git add .
git commit -m "Add my feature"
```

测试完成后合回：

```bash
git switch private/main
git merge --ff-only feature/my-feature
```

然后：

```bash
git push origin private/main
```

---

## 7. 实验性修改

实验代码建议使用：

```bash
git switch private/main
git switch -c experiment/test-name
```

可以随便调试、加日志、做临时修改。

实验成功后，把真正有价值的 commit 移回 `private/main`：

```bash
git switch private/main
git cherry-pick <commit>
```

---

## 8. 遇到 merge 冲突

合并官方更新：

```bash
git merge master
```

如果出现冲突，手动修改冲突文件，然后：

```bash
git add <file>
git commit
```

如果冲突太复杂，想取消这次合并：

```bash
git merge --abort
```

---

## 9. 日常最常用命令

查看当前状态：

```bash
git status
```

查看当前分支：

```bash
git branch --show-current
```

查看所有分支：

```bash
git branch -a
```

查看历史：

```bash
git log --oneline --graph --decorate -20
```

查看 remote：

```bash
git remote -v
```

---

## 推荐长期规则

1. `master` 永远保持官方干净版本。
2. 自己真正使用的是 `private/main`。
3. `legacy` 不继续开发，只用于查历史和移植旧功能。
4. 一个新功能尽量一个 branch。
5. 从 legacy 移植时优先使用 `cherry-pick`。
6. 每次合入新功能或官方更新后先编译、测试，再 push。
7. 不确定时，不要随便使用 `reset --hard`、`rebase` 或强制 push。
