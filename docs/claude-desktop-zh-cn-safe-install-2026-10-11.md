# Claude Desktop 简体中文补丁项目记录

日期：2026-10-11（Asia/Shanghai）  
目标：在 Windows Store/MSIX 版 Claude Desktop `2.31226.1.0` 上启用简体中文，并保留第三方 API / CC Switch 使用能力。

## 1. 最终结果

- 已安装并验证简体中文界面，用户已在 Claude Desktop 设置中看到“常规、隐私、用量、技能、连接器、插件”等中文页面。
- 使用上游项目 `javaht/claude-desktop-zh-cn` `1.4.11`，固定提交 `daee457c47594a10a93153456f59328e0eb6fee6`，MIT License。
- 使用 Windows `safe` 模式并显式传入 `-SkipAsarPatch`：不改 `app.asar`，不改 `Claude.exe` 内嵌完整性哈希，不启用 `official` 模式。
- Claude 仍通过 AppX 身份正常重启；`CoworkVMService` 当前为 `Running`，启动类型为 `Automatic`。
- Claude 3P 配置与语言配置仍在本机，已确认 `locale=zh-CN`；本记录不复制任何 API Key、Token、Cookie 或完整用户配置。

## 2. 上游与安装来源

上游：<https://github.com/javaht/claude-desktop-zh-cn>  
发布版本：`1.4.11`，发布日期 `2026-10-05`  
固定提交：`daee457c47594a10a93153456f59328e0eb6fee6`  
许可证：MIT

本次 Git clone 因 GitHub `443` 连接超时没有反复重试，改为下载 GitHub codeload 源码 ZIP，再在本机审计后执行。没有执行 README 中的远程 `irm ... | iex` 安装方式。

源码副本目录（本机）：

```text
work/javaht-src/claude-desktop-zh-cn-main
```

安装命令（已执行）：

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass `
  -File .\scripts\install_windows.ps1 `
  -Action install `
  -Language zh-CN `
  -PatchMode safe `
  -SkipAsarPatch
```

## 3. 实际改动范围

safe 模式写入了中文语言资源、动态资源和需要的前端静态替换，并更新 Claude 的语言白名单。安装日志显示约处理 `3,333` 个前端 JS 文件，其中约 `465` 个文件发生硬编码中文替换，约 `4,206` 条替换。

没有执行以下高风险路径：

- `official` 模式：不修改 `app.asar`，不修改 `Claude.exe` 的 Authenticode/内嵌 asar 完整性数据。
- Frida 运行时汉化：不注入进程，不下载便携 Python/Frida，不启动 watcher。
- 计划任务、RunKey、后台常驻自愈。
- CC Switch 的 API Key、Claude 3P 会话或聊天数据迁移。

安装备份目录由脚本创建在 Claude 资源目录下：

```text
app/resources/.zh-cn-backups/20261011-105042
```

## 4. 验收证据

| 检查项 | 结果 |
|---|---|
| Claude 版本 | `2.31226.1.0` |
| 上游补丁 | `1.4.11` / 固定提交 `daee457c...` |
| `app.asar` 安装前 SHA-256 | `EFA9E2310CF943B333B153C093BABB8B5A92F80A23CB68764DBA9B29A0801D4A` |
| `app.asar` 安装后 SHA-256 | 与安装前相同：`EFA9E2310CF943B333B153C093BABB8B5A92F80A23CB68764DBA9B29A0801D4A` |
| Claude 启动 | 通过 AppX 激活正常启动 |
| 中文界面 | 已通过实际 UI 验证 |
| Claude 3P `locale` | `zh-CN` |
| CoworkVMService | `Running` / `Automatic` |
| Frida 计划任务 | `ClaudeDesktopZhCn-FridaZh` 不存在 |
| Frida RunKey | `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\ClaudeDesktopZhCnFridaZh` 不存在 |
| `%LOCALAPPDATA%\claude-zh` | 不存在 |
| API Key / Token | 未读取、未复制、未输出 |

## 5. 安全审计结论

### 已检查到的正常行为

1. Windows 安装脚本的明确外网请求只有 GitHub Releases 更新检查：

   ```text
   https://api.github.com/repos/javaht/claude-desktop-zh-cn/releases/latest
   ```

   请求用于比较版本，不把 Claude 配置、API Key 或聊天内容作为参数发送。本次运行中该请求遇到匿名 API 限流，脚本按设计忽略并继续安装。

2. 实验性 Frida 代码包含下载 Python/Frida 依赖的逻辑，也包含计划任务和 RunKey 的注册逻辑，但这些代码不属于本次执行路径；本次已主动运行停用清理，并现场核对任务、RunKey 和目录均不存在。

3. 源码副本没有发现 `.exe`、`.dll`、`.sys`、`.msi` 等二进制载荷；未发现常见 API Key、私钥、Bearer Token 或明显混淆/编码载荷。

4. 本次运行期间 Claude 进程没有观察到额外的第三方回传连接。Claude 自身访问 Anthropic/你配置的 API 服务属于应用正常功能，不能被汉化补丁审计替代。

### 风险边界

- 这不是数学意义上的“绝对没有后门”证明。结论是：截至固定提交、已审计源码和本次 safe 执行路径，没有发现将 API Key、Claude 配置或聊天内容主动上传到第三方的明显代码。
- 安装需要管理员权限，因为目标是 `C:\Program Files\WindowsApps`；这本身是高权限补丁风险。
- 上游未来更新可能改变代码或安装行为；复制到本仓库的源码固定在上述提交，不会自动跟随更新。
- Claude/CC Switch 连接的中转服务仍能看到发送给它的请求，这是 API 路由的固有信任边界，与中文补丁无关。

## 6. 回滚方法

优先使用同一份固定源码：

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass `
  -File .\scripts\install_windows.ps1 `
  -Action uninstall `
  -SkipAsarPatch
```

也可以运行 `install-windows.bat`，选择 `[4] 恢复原样 / 卸载补丁`。脚本会停止 Claude、恢复最近备份的前端资源、删除语言资源、移除语言注册，并清理可能存在的 Frida 常驻项。回滚后应再次检查 Claude 是否能从开始菜单正常启动。

## 7. 已复制到个人仓库的内容

目标仓库：<https://github.com/ryeeeerrr/wc-ai-memory>  
选择理由：该仓库是本地优先的 AI 记忆库，适合保存可审阅、可 Git 追踪的项目过程与安全证据。

将新增以下独立目录，不覆盖原有文件：

```text
docs/claude-desktop-zh-cn-safe-install-2026-10-11.md
vendor/claude-desktop-zh-cn-1.4.11/
```

`vendor` 是固定上游源码副本，已排除本机 `install-windows.log`；不会放入 API Key、Cookie、Claude 配置或本机绝对路径日志。另有 `UPSTREAM-SNAPSHOT.md` 记录提交、版本、许可证和复制边界。

## 8. 后续维护建议

- Claude 自动更新后，先记录新版本与 `app.asar` 哈希，再决定是否重新运行 safe 模式。
- 更新上游前先审阅 diff，尤其是网络、下载、提权、计划任务、RunKey 和动态执行相关代码。
- 不要在公共仓库提交 `config.json`、`.env`、CC Switch 导出文件或安装日志；如需保存证据，只保存脱敏后的版本、哈希、路径类别和结果。
- 若要验证“无回传”，应在隔离环境中结合 Windows 防火墙/抓包做动态测试；静态审计不能替代完整供应链审计。
