**简体中文** | [English](README.en.md)

# dsh-approval-gate

**DeepSeek Harness 自动审批门控 —— 最小人工介入，安全自动放行、危险转人工（fail-safe）。**

> ## ⚠️ 这是复刻（fork）版本
>
> 上游仓库：[moon09300731/dsh-approval-gate](https://github.com/moon09300731/dsh-approval-gate)
>
> 上游 `0.5.2` 在 **DSH `0.1.5-rc.2`** 上存在**门控整体失效**问题：审批监听器在第一步就抛错并被静默吞掉，
> 导致 Flash 判定、自动放行、学习计数全部不执行，插件形同未安装。
>
> 本仓库针对该问题只做了**一行修复**，其余逻辑与上游保持一致，便于将来同步。
>
> ```sh
> dsh plugin --profile web add github:VanemKrAu/dsh-approval-gate
> ```

Flash 模型预判每次沙箱越界：常规操作自动放行，硬风险操作（删除 / 凭据 / 远程 / 系统 / 批量）永远转人工确认；学习沉淀只针对你确认过的操作，并提供界面化人工审查入口。

## 🔧 本复刻版本的改动

### 改了什么

仅一处，位于 `src/index.mjs` 的 `approval/request` 监听器内：

```diff
- preset = permissionPresets.current(session.events)
+ preset = permissionPresets.current(session)
```

### 为什么改

DSH 的 `PermissionPresetService.current(session)` 接收的是 **Session 对象**：它内部经
`permissionState(session)` → `sessionProjections.stateOf(session, 'permissions')` →
`cellFor(registration, session)` 读取 `session.header` 与 `session.snapshotEvents()`。

而 `Session` 类**没有 `events` 属性**（只有 `snapshotEvents()` 方法），因此 `session.events` 恒为
`undefined`，调用必然抛错：

```
[dsh-approval-gate] permissionPresets.current failed
TypeError: Cannot read properties of undefined (reading 'header')
```

异常被监听器自身的 `try/catch` 捕获后执行 `return next()`，把审批请求原样交给下游 —— 门控从不接管。

### 修复前的表现

- 每一次沙箱越界都弹人工确认，**没有任何自动放行**
- 「审批」视图始终空白
- `~/.dsh/auto-approve/` 下只有 `allowlist.json`，`audit.log` / `events.jsonl` / `learning.json` **从不生成**
- 设置页「学习沉淀」永远为空，看不到「学习 n/N」进度

### 验证

环境：DSH `0.1.5-rc.2` + 本插件 `0.5.2`，profile `web`，会话预设钉在 `auto-approve`。

| 项目 | 修复前 | 修复后 |
| --- | --- | --- |
| `audit.log` | 文件不存在 | 每次判定写入一行 |
| Flash 判定 | 从未执行 | `ALLOW … (flash-safe)` 自动放行 |
| 人工确认 | 6 场 auto-approve 会话累计 97 次，全部转人工 | 安全操作静默通过，不再打扰 |
| 学习计数 | 恒为空 | `learning.json` 正常累计（`neutral` 类别） |
| 危险操作 | 转人工 | 仍然转人工（fail-safe 未受影响） |

### 与上游的关系

同一问题在上游已有报告与修复 PR，截至本复刻版本建立时**均未合并**：

- Issue [#3](https://github.com/moon09300731/dsh-approval-gate/issues/3)（原始报告）、[#5](https://github.com/moon09300731/dsh-approval-gate/issues/5)、[#14](https://github.com/moon09300731/dsh-approval-gate/issues/14)（重复报告）
- PR [#1](https://github.com/moon09300731/dsh-approval-gate/pull/1)、[#7](https://github.com/moon09300731/dsh-approval-gate/pull/7)、[#10](https://github.com/moon09300731/dsh-approval-gate/pull/10)、[#11](https://github.com/moon09300731/dsh-approval-gate/pull/11)

若上游后续合并了修复，本仓库可直接同步（改动只有一行，冲突面极小）。

## ✨ 特性

- ⚡ **Flash 风险预判**：每次沙箱越界由 Flash 模型判定（`SAFE` / `RISKY:<类别>`），可回补操作自动放行
- 🛡️ **硬风险永远人工**：删除、凭据、远程/生产、系统路径、批量不可回补五类操作直接转人工，不计数、不学习
- 🎯 **确认制学习**：同一「工具 | 模式 | 类别」被人工确认满 N 次（默认 3）后，**第 N+1 次起自动放行**；沉淀规则携带**操作指纹**，只放行你确认过的操作
- 🧠 **语义同类验证**：措辞变化但意图相同的操作，由 Flash 对照你的确认样本语义判断，不再依赖关键词
- 🔧 **配置热更新**：`allowlist.json` 修改即时生效，无需重启
- ✅ **人工审查 UI**：自动放行时输入框上方出现绿色提示；「审批」视图（轨迹右侧）展示当前会话完整放行时间线
- 📄 **文件改动对比与撤销**（v0.5.0+）：审批涉及的文件可点击查看 **unified diff**——变动行带上下 5 行上下文、多处修改按 hunk 分区并以「N unmodified lines」分隔条折叠、绿加红删灰上下文、双行号；一键「撤销此改动」投递指令让 AI 按快照恢复文件
- 🗂️ **会话级快照管理**（v0.5.0+）：快照按事件归属会话，审批视图按当前会话统计；清理支持「仅清本会话」与「清空全部」两档，避免误删其他会话未查看的 diff 记录

## 📸 界面速览

### ① 审批视图

![审批视图](docs/screenshots/approval-view.png)

「审批」标签页（轨迹右侧）按时间倒序展示当前会话的自动放行与人工审批记录：每条记录含工具名（`bash` / `pwsh` / `edit`）、判定标签（「自动放行 · Flash 判定安全」「人工通过」等）、时间与操作说明。顶部统计栏显示本会话的 **diff 快照占用**（`2.9 KB · 3 条`），并提供两个清理入口：**「仅清本会话」**（只删除当前会话的快照，不影响其他会话未查看的 diff）与 **「清空全部」**（二次确认后清空所有会话，防止误删）。

### ② 文件改动对比（diff）

![diff 对话框](docs/screenshots/diff-panel.png)

点击审批记录中的文件即可打开对比面板：以 **unified diff** 展示改动前后差异——新增行绿底（`+`）、删除行红底（`-`）、上下文行灰底；左侧显示**原/新双行号**；多处修改按 **hunk 分区**，块间以灰色「`6 unmodified lines`」分隔条折叠未变更区间。顶部统计 `+2 / -2 行变更 · 20 行未变`。底部 **「撤销此改动」** 一键向对话投递撤销指令，AI 将按审批前的快照恢复文件。

### ③ 设置 · 自动审批

![设置-自动审批](docs/screenshots/settings-auto-approve.png)

设置页「自动审批」分区提供完整配置：**初始化权限预设**（一键写入 `cordis.patch.yml` 的 `auto-approve` 预设）、**当前判定管道总览**（DENY → 白名单 → denyRules → Flash → 学习）、**危险词黑名单**（预置条目 + 自定义添加）、以及热更新说明（修改即时生效，无需重启）。

## 🚀 快速开始

安装本复刻版本：

```sh
dsh plugin --profile web add github:VanemKrAu/dsh-approval-gate
```

1. **配置权限预设**：在 `~/.dsh/profiles/web/cordis.patch.yml` 添加 `auto-approve` 预设（[详见指南](docs/GUIDE.md#%E5%AE%89%E8%A3%85%E5%90%8E%E5%BF%85%E9%A1%BB%E6%89%8B%E5%8A%A8%E9%85%8D%E7%BD%AE%E6%9D%83%E9%99%90%E9%A2%84%E8%AE%BE%E5%85%B3%E9%94%AE%E6%AD%A5%E9%AA%A4)）
2. **重启** DSH（CLI 为 `dsh web`；桌面版退出应用后重新打开）
3. **选择预设**：会话权限下拉选中「自动审批（Flash）」

## 📖 文档

- [完整指南（管道 / 配置 / 安全设计 / 审查 UI）](docs/GUIDE.md) · [English Guide](docs/GUIDE.en.md)

## 📄 License

MIT，与上游一致。原始版权归 [moon09300731](https://github.com/moon09300731) 所有；本复刻版本仅在其基础上追加上述一处修复。
