---
name: go-commit
description: >
  生成语义化 Git Commit Message，并默认自动暂存当前会话改动后提交。
  当用户提到commit, 提交, git commit, 生成提交信息或直接调用 /go-commit.
  Also trigger when user say "帮我写 commit message", "提交一下", "commit 怎么写"
  or any request to create, generate, or review a git commit message.
---

# go-commit

## 核心原则

1. **描述优先使用中文**，保留英文技术术语和代码标识符不翻译
2. **Conventional Commits 格式**：`type(scope): subject`，body 可选
3. **无参自动提交**：默认路径是添加当前会话改动 → 生成一行 commit message → 执行 `git commit`
4. **只提交当前会话改动**：优先使用 Agent 当前会话已修改文件列表；避免把用户既有改动或无关 staged 变更混入提交
5. **显式预览机制**：仅在用户带 `--preview` 或主动要求确认时，展示生成结果并等待确认

## 用法

```
/go-commit [options]
```

### 参数

| 参数 | 说明 |
|------|------|
| `--type=<type>` | 强制指定 commit type，覆盖 AI 推断 |
| `--scope=<scope>` | 强制指定 scope，覆盖 AI 推断 |
| `--verbose` / `-v` | 生成带 body 的详细 commit message |
| `--amend` | 改写上一次提交 message，结合上次 message 和当前 staged diff 重新生成 |
| `--breaking` | 在 type 后加 `!` 标记破坏性变更，并隐式启用 body 生成说明迁移影响 |
| `--preview` | 只预览生成结果，等待用户确认后再提交 |
| `--yes` | 兼容参数；默认已直接提交，带 `--preview` 时可跳过确认 |
| `--push` | 提交后自动推送到远程；与 `--amend` 配合时使用 `--force-with-lease` |

### 示例

```bash
/go-commit
/go-commit -v
/go-commit --type=fix --scope=frame
/go-commit --preview
/go-commit --amend
/go-commit --amend --push
/go-commit --yes --push
/go-commit --breaking
/go-commit --verbose --push
```

## 执行流程

### 阶段 1: 环境检查

1. 查找 git 仓库根目录（从当前目录向上追溯）。
2. 检查 `git status`：
   - 若存在未解决的 merge 冲突 → 报错退出，提示先解决冲突。
   - 若不在 git 仓库中 → 报错退出。
3. 确定本次提交目标文件：
   - 优先使用 Agent 当前会话记录的已编辑文件列表，包括新增、修改、删除、重命名文件。
   - 只选择本会话改动的文件；不要默认纳入会话开始前已存在的 unstaged/staged 改动。
   - 若无法可靠识别当前会话文件，但暂存区已有内容，则以当前 staged 内容作为提交目标。
   - 若无法可靠识别当前会话文件，且只有 unstaged 改动，则展示候选文件并请求用户确认，不要静默执行整仓 `git add -A`。
4. 若不是纯 message amend，则暂存目标文件：
   - 对当前会话目标文件执行 `git add -A -- <paths...>`，让新增、修改、删除都进入暂存区。
   - 后续 diff 和 commit 都限定在这些目标文件上：`git diff --staged -- <paths...>`、`git commit -- <paths...>`。
   - 若目标文件为空且 staged 为空 → 提示没有可提交变更并退出。
5. 若带 `--amend`：
   - 读取上一次 commit message（`git log -1 --format=%B`）。
   - 若仓库没有任何提交历史 → 降级为普通提交模式。
6. 读取目标 staged diff：
   - 普通模式：读取 `git diff --staged -- <paths...>`。
   - 无目标文件但已有 staged 内容时：读取 `git diff --staged`。
   - 若 diff 为空且不是纯 message amend → 提示没有可提交变更并退出。

### 阶段 2: 上下文收集

1. **读取目标 staged diff**：获取当前会话目标文件的完整变更内容，作为生成核心依据。
2. **提取常用 scope**：执行 `git log --oneline -50`，解析所有 `type(scope):` 模式中的 scope，去重后作为候选 scope 列表。
3. **分析变更特征**：
   - 统计变更文件数和行数
   - 识别涉及的主要目录/模块
   - 识别变更类型（新增/修改/删除/重命名）

### 阶段 3: Message 生成

由 PROMPT.md 中定义的生成规则驱动，核心逻辑：

1. **推断 type**：根据 diff 内容判断（可被 `--type` 覆盖）。
2. **推断 scope**：根据文件路径和 git 历史判断（可被 `--scope` 覆盖）。
3. **生成 subject**：中文一句话总结，保留英文术语。
4. **决定是否生成 body**：默认不生成 body（详见 PROMPT.md 按需模式规则）。仅在以下情况生成：
   - 用户带 `--verbose` / `-v`
   - 用户带 `--breaking`（隐式启用）
   - 预览阶段用户要求补充细节
5. **处理 `--breaking`**：强制在 type 后加 `!`，并生成 body 说明破坏性变更。

### 阶段 4: 默认提交或预览确认

默认不进入交互确认：生成 commit message 后直接执行提交，**默认为一行 subject**。若用户带了 `--verbose` 或 `--breaking`，则使用含 body 的完整版本提交。

默认成功输出：

```
已提交：
feat(frame): 新增 config gen 生成 sample 配置

变更概览：
- 文件数：3 个
- 新增：+45 行，删除：-5 行
```

若用户带 `--preview`，展示生成的 commit message（含 diff 摘要）并等待确认。

**`--preview` 预览（一行 subject）：**

```
即将生成的 commit message：

────────────────────────────────────────
feat(frame): 新增 config gen 生成 sample 配置
────────────────────────────────────────

变更概览：
- 文件数：3 个
- 新增：+45 行，删除：-5 行

确认提交？
- y / yes / 确认  → 执行 git commit
- 直接输入修改意见 → AI 重新生成（如：scope 改成 cli，加 body 补充细节）
- edit / 编辑     → 进入结构化字段编辑
- n / no / 取消   → 退出
```

**`--preview --verbose` 或 `--preview --breaking` 预览（含 body）：**

```
即将生成的 commit message：

────────────────────────────────────────
feat(frame): 新增 config gen 生成 sample 配置

- frame: 支持根据模块类型生成对应的 sample 配置
- config: 调整配置结构以适配新的生成逻辑
────────────────────────────────────────

变更概览：
- 文件数：3 个
- 新增：+45 行，删除：-5 行

确认提交？
- y / yes / 确认  → 执行 git commit
- 直接输入修改意见 → AI 重新生成
- edit / 编辑     → 进入结构化字段编辑
- n / no / 取消   → 退出
```

#### 用户响应处理

| 用户输入 | 动作 |
|---------|------|
| `y` / `yes` / `确认` / `1` | 执行 `git commit -m "..."` |
| `n` / `no` / `取消` / `q` / `quit` | 退出，不执行任何操作 |
| `edit` / `编辑` / `e` | 进入结构化字段编辑模式（兜底） |
| 自然语言 | 理解意图，修改后重新生成并再次预览 |

**自然语言反馈示例**：
- "scope 不对，改成 metrics" → 修改 scope 后重新生成
- "太长了，精简一下" → 缩短 subject 后重新生成
- "加 body，解释一下为什么改" → 补充 body 后重新生成
- "type 应该是 fix 不是 feat" → 修改 type 后重新生成
- "直接提交吧" → 同 `y`，执行 commit

#### 结构化字段编辑（兜底模式）

当自然语言反馈多次无法理解时，展示当前 message 的结构化形式：

```
当前 commit message 结构：

[type]    feat
[scope]   frame
[subject] 新增 config gen 生成 sample 配置
[body]    - frame: 支持根据模块类型生成对应的 sample 配置
          - config: 调整配置结构以适配新的生成逻辑

请指定要修改的字段和新值，或直接发送完整的 commit message。
示例：
- "改 scope 为 cli"
- "subject 改成：生成 sample 配置时支持模块类型识别"
- "删除 body"
```

### 阶段 5: 提交与推送

1. **执行 commit**：
   - 普通模式且有目标文件列表：`git commit -m "subject" -- <paths...>`；若有 body，使用多个 `-m` 后再追加 `-- <paths...>`。
   - 普通模式且仅使用已有 staged 内容：`git commit -m "subject"`；若有 body，使用多个 `-m`。
   - Amend 模式：`git commit --amend -m "new message"`；若同时有目标文件，先按阶段 1 暂存目标文件。
2. **若带 `--push`**：
   - 检查远程分支状态：`git fetch` 后比较本地与远程。
   - 若本地落后远程 → 拒绝 push，提示先 `git pull`。
   - 若本地领先或一致 → 执行 push。
   - 若是 amend + push → 使用 `git push --force-with-lease`。

## 错误处理

| 场景 | 处理方式 |
|------|---------|
| 不在 git 仓库中 | 报错："当前目录不是 git 仓库" |
| 存在未解决的 merge 冲突 | 报错："存在未解决的合并冲突，请先解决后再提交" |
| 当前会话目标文件为空且 staged 为空（非 amend） | 提示："没有检测到当前会话可提交变更" |
| 无法识别当前会话文件但存在 unstaged 改动 | 展示候选文件并请求确认，不静默执行整仓 `git add -A` |
| 暂存区存在非目标文件 | 默认不纳入提交；commit 命令限定 `-- <paths...>` |
| `--amend` 但无提交历史 | 降级为普通提交模式 |
| `--push` 时本地落后远程 | 拒绝 push，提示先 pull |
| `--amend` + `--push` 远程拒绝 | 提示使用 `--force-with-lease` 后的结果，若仍失败则建议手动处理 |

## Type 定义

| Type | 使用场景 |
|------|---------|
| `feat` | 新增功能、新增模块、新增接口 |
| `fix` | Bug 修复、异常处理、边界情况修复 |
| `refactor` | 重构（不改变外部行为）、代码结构调整、依赖迁移 |
| `chore` | 杂项：依赖升级、配置调整、内部文档更新、CI/构建脚本 |
| `docs` | 对外文档更新（README、API 文档、CHANGELOG 补充） |
| `style` | 纯代码格式调整（格式化、lint 修复、去未使用 import） |
| `test` | 测试相关：新增/修改测试用例、测试工具调整 |

## 多模块变更处理

- **Subject**：使用主要变更的 scope；若涉及多个同等重要的模块，scope 留空。
- **Body**：按模块/scope 分条目说明，每个条目一句话。仅在用户带 `--verbose` 或预览时要求时才生成。
- **拆分建议**：若检测到明显不相关的多模块变更（如同时改了 `server` 和 `frame` 且没有关联），在预览时附加警告："这些变更看起来属于不同模块，建议考虑拆分为多次提交。"

## 文件说明

| 文件 | 说明 |
|------|------|
| `SKILL.md` | 技能定义、触发条件、执行流程、交互设计 |
| `PROMPT.md` | Claude 生成 commit message 时的核心系统指令（type/scope/body 规则） |
