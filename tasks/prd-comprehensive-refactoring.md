# PRD: Pad 全面重构

## Introduction

`pad` 是一个 Rust TUI 应用，用于监控和管理 tmux 中运行的 AI 编程助手（Claude、Codex、Kimi）。项目从 Python 原型演化为 Rust TUI，但积累了大量技术债：1,399 行的 `main.rs` 单体文件、Code*/AI*/Agent* 命名混乱、主题系统形同虚设、零测试覆盖，以及已废弃但未清理的 Python 代码。

本次重构目标：**一次性系统性解决所有问题**，废弃 Python 版本，全面转向 Rust TUI。UI/交互体验是最高优先级，架构改进为之服务。二进制名 `pad` 保持不变。

## Goals

- 将 main.rs 从 1,399 行拆分为多个聚焦模块，每个 < 300 行
- 统一内部命名：Code* → Agent*，消除命名混乱
- 实现真正的主题系统，支持 4+ 个可区分的配色方案
- 面板列表自适应终端宽度，窄屏不破版
- 添加空状态提示、帮助界面、预览滚动等交互改进
- 面板搜索升级为 fuzzy matching（nucleo-matcher 已在依赖中）
- 废弃并移除 Python 代码，更新安装和文档
- 添加 25+ 单元测试覆盖核心逻辑

## User Stories

---

### Phase 1: 架构拆分 + 命名统一（基础）

### US-001: 拆分 App 结构体到 app/ 模块

**Description:** As a developer, I want the App state and methods organized into focused sub-modules so that changes to one concern don't require reading 1,400 lines.

**Acceptance Criteria:**
- [ ] `main.rs` < 100 行，只含 `main()` 和终端 setup/teardown
- [ ] `app/mod.rs` 定义 App struct，字段按职责分组
- [ ] `app/state.rs` 含 Mode enum、Settings struct
- [ ] `app/navigation.rs` 含 next/previous/jump_to/filtered_panels/selected_panel
- [ ] `app/actions.rs` 含 refresh/delete/toggle_tree/toggle_settings 等动作
- [ ] `app/async_ops.rs` 含 trigger_async_scan/check_scan_result/trigger_async_preview 等异步操作
- [ ] `cargo build` 零新警告

### US-002: 提取 PTY 和会话管理

**Description:** As a developer, I want PTY attach logic isolated so I can reason about terminal I/O without App state noise.

**Acceptance Criteria:**
- [ ] `pty.rs` 含 attach_to_pane_pty()、find_detach_key()、find_f12_key()、Winsize
- [ ] `session.rs` 含 create_new_session_fuzzy()、create_session_in_path()
- [ ] `#[cfg(unix)]` 注解正确保留
- [ ] 手动测试：Enter 进入 pane，F12 返回正常

### US-003: 提取事件处理到 event.rs

**Description:** As a developer, I want key handling separated from state management so I can add new modes without touching the event loop.

**Acceptance Criteria:**
- [ ] `event.rs` 导出 `pub async fn run_event_loop(terminal, app)`
- [ ] 每个 Mode 的按键处理是独立私有函数（handle_normal_keys、handle_search_keys 等）
- [ ] 终端保存/恢复（PTY attach、fuzzy picker 前后）封装到 helper 函数

### US-004: 拆分 ui.rs 为子模块

**Description:** As a developer, I want rendering logic organized by component so I can modify one widget without reading all 660 lines.

**Acceptance Criteria:**
- [ ] `ui/mod.rs` 只含顶层 draw() 分发
- [ ] `ui/panel_list.rs` 含 draw_panel_list()
- [ ] `ui/preview.rs` 含 draw_preview()、draw_file_preview()、format_line()
- [ ] `ui/status_bar.rs` 含 draw_status_bar()
- [ ] `ui/modals.rs` 含 settings/theme/delete/agent launcher 弹窗
- [ ] `ui/layout.rs` 含 centered_rect() 和布局计算
- [ ] 渲染输出无变化

### US-005: 重命名 Code* → Agent*

**Description:** As a developer, I want naming to match the tool's purpose (monitoring AI agents, not "code") and align with the PRD.

**Acceptance Criteria:**
- [ ] `CodePanel` → `AgentPanel`
- [ ] `CodeType` → `AgentType`
- [ ] `code_type` 字段 → `agent_type`
- [ ] 所有引用更新：scanner、model、ui、tree、app
- [ ] 面板列表标题改为 `" Agent Panels "`
- [ ] `Cargo.toml` description 更新
- [ ] `cargo clippy` 通过

---

### Phase 2: UI/UX 改进（最高用户影响）

### US-006: 自适应列宽

**Description:** As a user, I want the panel list to look good on any terminal width so I don't see truncated or empty columns.

**Acceptance Criteria:**
- [ ] 列宽根据 `area.width` 动态计算
- [ ] 终端 < 80 列时隐藏 Git 列
- [ ] 终端 < 60 列时同时隐藏 Directory 列
- [ ] `shortened_path()` 接收实际可用宽度而非硬编码 20

### US-007: 真正的主题系统

**Description:** As a user, I want themes to actually change the application's colors so I can customize the look to my preference.

**Acceptance Criteria:**
- [ ] 新建 `theme.rs` 含 Theme struct（bg, fg, accent, highlight_bg, border, status_bar 等颜色字段）
- [ ] 至少实现 4 个视觉可区分的主题：Default, Dracula, Nord, Catppuccin
- [ ] 所有 ui/ 渲染函数使用活跃主题颜色，不再硬编码 Color::DarkGray 等
- [ ] 选择主题后立即全屏刷新
- [ ] 主题设置持久化到 `~/.config/pad/config.toml`

### US-008: 空状态处理

**Description:** As a new user, I want helpful guidance when no AI agents are detected so I know what to do next.

**Acceptance Criteria:**
- [ ] panels 为空时，面板列表区域显示居中提示（如何启动 agent、按键帮助）
- [ ] 预览区域显示欢迎/帮助信息而非 "Select a panel to preview"
- [ ] 空状态使用活跃主题颜色

### US-009: 改进状态栏

**Description:** As a user, I want the status bar to show me useful context so I know what's happening and what I can do.

**Acceptance Criteria:**
- [ ] 左侧：模式指示器（Normal/Search/Tree/Settings），每个模式不同颜色
- [ ] 中间：面板计数 + 选中索引（如 `2/5`）
- [ ] 右侧：扫描状态（进行中显示 spinner）+ 上次刷新时间
- [ ] 搜索模式下内联显示查询：`/ search_query_here`

### US-010: 改进 Tree 布局

**Description:** As a user, I want the tree mode to not squeeze the panel list too much.

**Acceptance Criteria:**
- [ ] tree 可见时：30%/30%/40% 分割（默认）
- [ ] 终端 > 160 列时：25%/35%/40%
- [ ] 终端 < 100 列时：tree 自动隐藏并在状态栏提示
- [ ] `Tab` 键在 tree、panel list、preview 间切换焦点（视觉指示当前焦点）

---

### Phase 3: 交互改进

### US-011: Fuzzy 搜索面板列表

**Description:** As a user, I want fuzzy search for panels so I can find agents quickly even with partial/typo input.

**Acceptance Criteria:**
- [ ] 面板搜索使用 nucleo-matcher（已在依赖中）
- [ ] 匹配范围：session name、window name、working dir、git branch、agent type
- [ ] 结果按匹配分数排序
- [ ] 搜索时高亮匹配字符
- [ ] 逐键增量更新

### US-012: 帮助界面

**Description:** As a new user, I want to discover all keybindings without reading source code.

**Acceptance Criteria:**
- [ ] `?` 键打开帮助叠加层（新 `Mode::Help`）
- [ ] 按模式分组展示所有快捷键
- [ ] 内容超出终端高度时可滚动
- [ ] `Esc` 或 `?` 关闭帮助

### US-013: 可配置 Agent Launcher

**Description:** As a user, I want to add my own AI agents to the launcher so I'm not limited to the hardcoded 5.

**Acceptance Criteria:**
- [ ] agent 列表从 `~/.config/pad/config.toml` 的 `[[agents]]` 段加载
- [ ] 首次运行生成默认配置（当前 5 个 agent）
- [ ] 支持自定义条目：`{ name = "copilot", cmd = "gh copilot" }`
- [ ] 配置缺失时 fallback 到硬编码默认值

### US-014: 预览滚动

**Description:** As a user, I want to scroll the preview pane to see more output history.

**Acceptance Criteria:**
- [ ] Normal 模式下 PageUp/PageDown 滚动预览
- [ ] Home/End 跳转到顶部/底部
- [ ] 切换面板时重置滚动位置
- [ ] 预览标题显示滚动位置（如 `[line 50/200]`）

---

### Phase 4: Python 清理 + 测试

### US-015: 移除 Python 代码

**Description:** As a maintainer, I want to remove the dead Python codebase to reduce confusion and maintenance burden.

**Acceptance Criteria:**
- [ ] 移除 `src/` 目录、`pyproject.toml`、`.python-version`、`tests/`、`rust/` 目录
- [ ] `install.sh` 改写为安装 Rust 二进制
- [ ] `README.md` 改写为 Rust-only 工具文档
- [ ] `PRD.md` 更新：移除 Python 引用，数据模型使用 AgentPanel/AgentType

### US-016: 核心逻辑单元测试

**Description:** As a developer, I want unit tests for core logic so refactoring doesn't introduce silent regressions.

**Acceptance Criteria:**
- [ ] model.rs：AgentType::from_processes()、shortened_path()、git_display()、status_icon()
- [ ] scanner.rs：tmux output 解析（提取为可测试函数）
- [ ] app/state.rs：Mode transition 验证
- [ ] app/navigation.rs：next/previous/jump_to 边界、filtered_panels 过滤
- [ ] theme.rs：所有主题可加载
- [ ] 至少 25 个测试，`cargo test` 全部通过

### US-017: Clippy + Fmt 清理

**Description:** As a developer, I want clean linting so CI can enforce code quality.

**Acceptance Criteria:**
- [ ] `cargo clippy -- -D warnings` 零警告
- [ ] `cargo fmt --check` 通过
- [ ] 审查所有 `#[allow(dead_code)]`，移除不需要的
- [ ] 删除未使用的 `content_hashes` 字段（如确认无用）

## Functional Requirements

- FR-1: App struct 拆分为 app/ 子模块，main.rs < 100 行
- FR-2: PTY 管理、会话创建各自独立模块
- FR-3: 事件处理独立模块，每个 Mode 有独立 handler
- FR-4: UI 拆分为 panel_list/preview/status_bar/modals/layout 子模块
- FR-5: 所有 Code* 内部类型重命名为 Agent*
- FR-6: 面板列表列宽自适应终端宽度
- FR-7: 主题系统使用真实配色方案，4+ 个主题
- FR-8: 空面板状态显示引导信息
- FR-9: 面板搜索使用 nucleo-matcher fuzzy matching
- FR-10: `?` 打开帮助界面
- FR-11: Agent launcher 列表可通过配置文件自定义
- FR-12: 预览窗格支持滚动
- FR-13: 移除所有 Python 代码和配置
- FR-14: 25+ 单元测试覆盖核心逻辑

## Non-Goals

- 不改变二进制名（保持 `pad`）
- 不添加 Web 界面
- 不添加会话持久化/对话导出功能
- 不重写 PTY 内部逻辑（Phase 1 只移动代码，不改行为）
- 不支持 Windows（tmux 是 Unix-only）
- 不添加鼠标交互支持
- 不做 Python → Rust 的迁移工具

## Technical Considerations

### 目标文件结构
```
rust-tui/src/
├── main.rs           (~80 lines)   - 入口：终端 setup、调用 event loop
├── app/
│   ├── mod.rs        (~120 lines)  - App struct 定义
│   ├── state.rs      (~80 lines)   - Mode enum、Settings struct
│   ├── navigation.rs (~60 lines)   - 导航方法
│   ├── actions.rs    (~200 lines)  - 动作方法
│   └── async_ops.rs  (~120 lines)  - 异步扫描/预览
├── model.rs          (~120 lines)  - AgentPanel、AgentType、GitInfo
├── scanner.rs        (~220 lines)  - tmux pane 扫描
├── pty.rs            (~200 lines)  - PTY attach + detach key 检测
├── session.rs        (~80 lines)   - 会话创建
├── theme.rs          (~150 lines)  - Theme struct + 预定义主题
├── config.rs         (~100 lines)  - 配置文件读写（主题 + agent 列表）
├── fuzzy.rs          (~340 lines)  - 目录 fuzzy picker
├── tree.rs           (~510 lines)  - 文件树 + agent launcher
├── ui/
│   ├── mod.rs        (~30 lines)   - draw() 分发
│   ├── panel_list.rs (~80 lines)
│   ├── preview.rs    (~120 lines)
│   ├── status_bar.rs (~40 lines)
│   ├── modals.rs     (~120 lines)
│   ├── layout.rs     (~40 lines)
│   └── help.rs       (~80 lines)   - 帮助界面渲染
└── event.rs          (~300 lines)  - 事件循环 + 模式 key handlers
```

### 新增依赖
- `toml` - 配置文件解析（~/.config/pad/config.toml）
- `dirs` - 已在依赖中，用于获取配置目录

### 关键约束
- PTY 代码是平台相关且脆弱的，Phase 1 只移动不重构
- Phase 1 的每个 US 完成后必须 `cargo build` + 手动测试
- 配置文件模块应统一处理主题持久化和 agent 列表（避免重复）

## Success Metrics

- main.rs < 100 行
- 最大单文件 < 350 行
- 4 个视觉可区分的主题
- 面板搜索响应 < 16ms（一帧内）
- 25+ 单元测试全部通过
- `cargo clippy -- -D warnings` 零警告
- Python 代码完全移除

## Open Questions

1. 配置文件是否需要版本号以支持未来格式迁移？
2. 是否需要支持从旧 Python 版配置迁移设置？
3. 主题是否需要支持用户自定义（~/.config/pad/themes/）还是只提供预定义主题？
4. 帮助界面是否需要支持 i18n（中英文）？
