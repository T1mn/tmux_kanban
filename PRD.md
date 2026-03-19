# Tmux AI Kanban - 产品需求文档 (PRD)

> 版本: 0.1.0  
> 日期: 2026-03-19  
> 状态: 需求确认阶段

---

## 1. 产品概述

### 1.1 背景
用户在 tmux 中同时运行多个 AI 编程助手（Claude、Codex、Kimi），随着 session 增多，难以快速定位和切换不同的 AI 对话窗口，也无法直观了解每个 panel 的工作状态、所在项目和对话内容。

### 1.2 目标
构建一个轻量级 CLI 工具，帮助用户：
- 快速筛选出所有运行 AI 工具的 tmux panel
- 一目了然地查看每个 panel 的上下文信息
- 方便地在不同 AI 对话间切换

### 1.3 核心场景
1. **多项目并行开发** - 在不同目录同时询问不同 AI 问题
2. **长时间任务监控** - 查看哪些 AI 还在运行、是否产生新输出
3. **快速定位** - 从众多 tmux 窗口中快速找到特定 AI 对话
4. **上下文切换** - 了解每个 AI panel 的 git 分支和工作目录

---

## 2. 功能需求

### 2.1 核心功能 (MVP)

| 优先级 | 功能 | 说明 | 状态 |
|-------|------|------|------|
| P0 | AI Panel 检测 | 自动识别运行 claude/codex/kimi 的 pane | ✅ |
| P0 | 基础信息展示 | session/window/pane、工作目录 | ✅ |
| P0 | Git 信息 | 分支、commit、变更文件数 | ✅ |
| P0 | 列表视图 | 表格形式展示所有 AI panels | ✅ |
| P1 | 内容捕获 | 捕获 pane 历史输出（最近 N 行） | ✅ |
| P1 | 对话解析 | 识别用户输入与 AI 回复的轮次 | ✅ |
| P1 | 活跃状态 | 检测 AI 是否正在生成内容 | ✅ |
| P2 | 实时监控 | watch 模式，定时刷新 | ✅ |
| P2 | 内容详情 | 查看指定 pane 的完整输出 | ✅ |

### 2.2 扩展功能 (Future)

| 优先级 | 功能 | 说明 |
|-------|------|------|
| P3 | TUI 交互界面 | 使用 Textual 构建可交互看板 |
| P3 | 搜索过滤 | 按目录、关键词搜索对话内容 |
| P3 | 会话持久化 | 保存对话历史到文件 |
| P3 | 跨 session 跳转 | 一键跳转到指定 pane |
| P3 | Web 界面 | 浏览器访问的看板 |

---

## 3. 数据模型

### 3.1 AIPanel (AI Panel 实体)
```python
{
  # Tmux 标识
  session: str,           # e.g., "idea"
  window: str,           # e.g., "blog"
  pane: str,             # e.g., "1"
  pane_id: str,          # e.g., "%155"
  
  # AI 信息
  ai_type: "claude" | "codex" | "kimi",
  ai_version: str?,      # 版本号（如有）
  
  # 上下文
  working_dir: str,      # 工作目录
  git_branch: str?,      # git 分支
  git_commit: str?,      # commit hash (short)
  git_status: "clean" | "dirty" | "no_git",
  changed_files: int,    # 变更文件数
  
  # 内容
  last_content: str,     # 最近内容摘要
  history_lines: int,    # 可回溯历史行数
  conversation_turns: [  # 对话轮次
    {role: "user" | "assistant", content: str}
  ],
  
  # 状态
  is_active: bool,       # 是否活跃（正在生成）
  process_count: int,    # 子进程数
}
```

### 3.2 数据来源

| 数据 | 来源命令 | 说明 |
|------|---------|------|
| Pane 基础信息 | `tmux list-panes -a -F` | session/window/pane_id/pid/path |
| 进程识别 | `pstree -p [pid]` | 识别 claude/codex/kimi 进程 |
| Pane 内容 | `tmux capture-pane -p -t [pane]` | 捕获屏幕输出 |
| Git 信息 | `git branch --show-current` 等 | 在 working_dir 执行 |
| 历史行数 | `tmux list-panes -F #{history_size}` | 可回溯内容量 |

---

## 4. 技术方案

### 4.1 技术栈
- **语言**: Python 3.9+
- **CLI 框架**: Typer
- **表格展示**: Rich
- **可选 TUI**: Textual
- **打包**: hatchling

### 4.2 架构
```
tmux-ai-kanban/
├── src/tmux_ai_kanban/
│   ├── main.py           # CLI 入口 (Typer)
│   ├── models.py         # 数据模型 (dataclass)
│   ├── detector.py       # AI panel 检测逻辑
│   ├── tmux_client.py    # tmux 命令封装
│   ├── git_info.py       # git 信息获取
│   └── ui/
│       └── table.py      # Rich 表格渲染
├── pyproject.toml
└── README.md
```

### 4.3 AI 进程识别规则

| AI 工具 | 进程名匹配 | 备注 |
|---------|-----------|------|
| Claude | `claude` | 主进程名 |
| Codex | `codex` | npm 包名 |
| Kimi | `Kimi`, `Kimi Code` | 含空格 |

### 4.4 内容捕获策略
1. 捕获最近 100 行历史 (`capture-pane -S -100`)
2. 解析对话轮次（识别 `$`/`❯` 用户输入 和 `●`/`•` AI 回复）
3. 提取最后 2-3 轮对话作为摘要

---

## 5. UI/UX 设计

### 5.1 命令行接口

```bash
# 列表视图
ai-kanban list
ai-kanban list --filter claude
ai-kanban list --watch

# 详情视图
ai-kanban show %155 --lines 50
ai-kanban conversations

# 统计与跳转
ai-kanban summary
ai-kanban jump idea blog 1
```

### 5.2 表格列设计

| 列 | 宽度 | 说明 |
|---|------|------|
| AI | 4 | Emoji 标识 🟣🔵🟢 |
| Status | 6 | ● 活跃 / ○ 空闲 |
| Session | 自适应 | Session 名 |
| Window | 自适应 | Window 名 |
| Working Dir | 25 | 工作目录（折叠） |
| Git | 20 | 分支@commit +变更 |
| Latest Activity | 45 | 最近对话摘要 |

### 5.3 颜色规范

| 元素 | 颜色 | 说明 |
|------|------|------|
| AI Emoji | 紫/蓝/绿 | Claude紫、Codex蓝、Kimi绿 |
| Active Status | green | 正在生成内容 |
| Git Clean | green | 无变更 |
| Git Dirty | yellow/red | 有变更 |
| Path | cyan | 目录路径 |
| Content | dim | 灰色摘要 |

---

## 6. 里程碑计划

### Phase 1: MVP (已完成)
- [x] 项目结构与依赖配置
- [x] AI Panel 检测器
- [x] Git 信息获取
- [x] Rich 表格展示
- [x] 基础 CLI 命令 (list/summary/jump)

### Phase 2: 内容增强 (建议)
- [ ] 优化对话解析准确率
- [ ] 支持更多 AI 工具（如 aider, cursor）
- [ ] 内容搜索功能

### Phase 3: TUI 界面 (可选)
- [ ] Textual 交互式看板
- [ ] 键盘导航 (j/k/Enter/q)
- [ ] 实时刷新与动画

### Phase 4: 高级功能 (未来)
- [ ] 会话历史持久化
- [ ] Web 界面
- [ ] 对话导出 (markdown)

---

## 7. 验收标准

### 7.1 功能验收
- [ ] 能正确识别运行中的 claude/codex/kimi panel
- [ ] 列表展示 session/window/pane/目录/git/内容
- [ ] 能捕获并展示最近对话内容
- [ ] watch 模式能定时刷新
- [ ] 过滤功能正常工作

### 7.2 性能标准
- 检测全部 pane 耗时 < 2s
- 内容捕获不影响目标 pane 性能
- 内存占用 < 50MB

### 7.3 兼容性
- Python 3.9+
- tmux 3.0+
- Linux/macOS

---

## 8. 附录

### 8.1 竞品参考
- [vim-tmux-navigator](https://github.com/christoomey/vim-tmux-navigator) - pane 导航
- [tmuxinator](https://github.com/tmuxinator/tmuxinator) - session 管理

### 8.2 风险与限制
1. **内容捕获限制** - 依赖 tmux 缓冲区，无法获取 AI 内部状态
2. **进程识别** - 基于进程名启发式匹配，可能误判
3. **性能** - 大量 pane 时扫描可能变慢

### 8.3 术语表
- **Pane**: tmux 中的分割窗口单元
- **Window**: tmux 中的标签页
- **Session**: tmux 会话，包含多个 window
