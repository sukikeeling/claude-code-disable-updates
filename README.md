# Claude Code 桌面端（3p）禁用自动更新

> 逆向分析记录：Claude Code 桌面端（第三方部署版，3p）的自动更新机制与禁用方法  
> 适用版本：1.28929 ~ 1.30096+  
> 分析时间：2026-08-14

---

## 问题

Claude Code 桌面端会在后台自动下载并安装更新，每次更新后重启应用，打断工作流。用户找不到关闭入口。

网上常见的方案（`%APPDATA%\Claude\claude_desktop_config.json` 加 `{"update":{"mode":"none"}}`）**对这个版本无效**——那是 Claude 聊天 App（1p）的配置格式，3p 版完全不读。

## 分析方法

反编译 app.asar（Electron 应用包），追踪更新器 bootstrap 流程：

1. 定位 `autoUpdater` 模块
2. 追踪配置读取路径：`autoUpdate.disabled` ← 配置扁平化映射 ← 配置文件顶层键
3. 确认配置目录解析函数：3p 模式读 `%LOCALAPPDATA%\Claude-3p\` 而非 `%APPDATA%\Claude\`
4. 新旧两个版本（1.28929 / 1.30096）都验证过，机制一致

## 禁用方法

### 修改配置文件

**文件路径**：
```
%LOCALAPPDATA%\Claude-3p\claude_desktop_config.json
```

**添加顶层键**：
```json
{
  "disableAutoUpdates": true,

  "...原有配置保持不动..."
}
```

### 原理

| 配置键 | 映射路径 | 效果 |
|--------|----------|------|
| `disableAutoUpdates` | `autoUpdate.disabled` | 更新器 bootstrap 在设置 feed URL、检查更新、下载之前就检查此值，为 `true` 时直接 return，所有更新活动全停 |

更新器入口函数 `An()` 的伪代码逻辑：

```
function An() {
  if (autoUpdate.disabled) return;  // ← 早退，跳过全部
  setFeedURL();
  checkForUpdates();
  downloadUpdate();
  installUpdate();
}
```

手动检查更新 `In()` 和暂存更新安装 `Ln()` 同样检查此值，为 `true` 时早退。

### 验证

修改后重启应用，日志中应出现：
```
[updater] Auto-updates disabled by enterprise policy
```

不再出现：
```
[updater] Checking for updates
[updater] Downloading update
[updater] Installing update
```

日志位置：`%LOCALAPPDATA%\Claude-3p\logs\main.log`

## 技术细节

### 为什么不是 `%APPDATA%\Claude\`

代码中配置目录解析函数 `V7()`：

```
B7 = "claude_desktop_config.json"
z7 = "Claude-3p"
```

3p 模式下拼接路径为 `%LOCALAPPDATA%\Claude-3p\claude_desktop_config.json`。

### 为什么是 flat key 而不是嵌套对象

配置系统使用 `CQ()` 函数将 flat key 递归映射到嵌套 schema 路径：

```
disableAutoUpdates → autoUpdate.disabled
```

这是 enterprise policy 级别的配置读取方式，与普通用户配置（`preferences` 对象）走不同通道。

### 更新通道

- 更新 feed：`https://api.anthropic.com`（channel=production）
- 应用包类型：侧载 MSIX（`SignatureKind: Developer`）
- 无 `.appinstaller` 文件
- 无 Squirrel 注册表项
- 应用内更新器是唯一更新机制

### 配置不会被覆盖

配置文件中的 `deploymentMode`、`coworkUserFilesPath` 等 managed 键历经多次重启和版本更新仍然保留，说明应用写 `preferences` 时保留 managed 键，`disableAutoUpdates` 同理。

## 文件清单

```
%LOCALAPPDATA%\Claude-3p\
├── claude_desktop_config.json    # ← 配置文件（改这里）
├── claude-code\                  # Claude Code 引擎
│   └── 2.1.227\claude.exe
├── logs\
│   └── main.log                  # 更新器日志
└── ...
```

## 注意事项

1. 修改后**必须完全退出并重启** Claude 桌面端，当前进程的 bootstrap 已读取旧配置
2. 如果应用大版本更新改变了配置读取逻辑，此方法可能失效（但 1.28929 → 1.30096 已验证一致）
3. 此方法仅适用于 3p（第三方部署）版 Claude Code 桌面端，1p（官方直连）版配置路径和键名不同

---

> 声明：此分析仅用于技术研究目的，帮助用户管理本地软件行为。请遵守 Anthropic 的服务条款。

## 仓库声明

本仓库的署名与参考规范见 [NOTICE](NOTICE)：参考、借鉴或改造本仓库时，请标明来源 sukikeeling/claude-code-disable-updates。
