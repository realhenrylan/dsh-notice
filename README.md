# dsh-notice

**中文** | [English](README.en.md)

![Windows 10/11](https://img.shields.io/badge/platform-Windows%2010%2F11-blue)
![DeepSeek Harness](https://img.shields.io/badge/DeepSeek%20Harness-0.2.0--rc.2%2B-green)
![MIT License](https://img.shields.io/badge/license-MIT-lightgrey)

一个用于 Windows 的 DeepSeek Harness 桌面通知插件。当 agent 等待授权、提出问题或结束任务时，它会提醒你；点击通知可返回对应对话。

## 功能

- **需要操作时提醒**：授权请求会显示工具名和原因；提问会显示问题及选项摘要。
- **任务状态提醒**：回合完成、出错或中止时发出通知；被阻塞且没有收到其他等待提醒时也会补发。
- **两种通知方式**：通常使用 Windows 原生通知；检测到前台全屏时使用置顶卡片，原生通知投递失败时也会尝试用卡片补位。
- **点击回到现场**：唤起 DSH 并切换到对应对话，尽可能滚动到最新消息。
- **减少打扰**：正在查看对应对话时默认不提醒；过滤短回合和重复事件，默认不通知子代理。

## 环境要求

- Windows 10 / 11（x64）
- DeepSeek Harness 桌面版，`0.2.0-rc.2` 或兼容版本
- PowerShell 5.1 和 .NET Framework 4

插件随附通知所需的程序，正常安装无需自行构建。其他操作系统不支持 Windows 通知功能。

## 安装

### 通过插件管理器

在 DSH 的 **设置 → 插件** 中使用以下 spec 安装，然后重启 DeepSeek Harness：

```text
github:realhenrylan/dsh-notice
```

### 手动安装

1. 将仓库克隆到固定目录：

   ```powershell
   git clone https://github.com/realhenrylan/dsh-notice.git D:\tools\dsh-notice
   ```

2. 编辑 `%USERPROFILE%\.dsh\profiles\<profile>\package.json`（通常 `<profile>` 为 `desktop`），在现有配置中添加以下条目，保留已有的依赖和 bundle：

   ```json
   {
     "dependencies": {
       "dsh-desktop-notify": "link:D:/tools/dsh-notice"
     },
     "dsh": {
       "profile": {
         "bundles": ["dsh-desktop-notify"]
       }
     }
   }
   ```

   `link:` 后使用你实际的克隆路径，路径分隔符使用正斜杠。仓库名是 `dsh-notice`，插件包名仍为 `dsh-desktop-notify`。

3. 在对应 profile 目录使用 DSH 自带的 Node.js 和 pnpm 执行安装。它们位于 `%USERPROFILE%\.dsh\dsh-runtimes\` 下：

   ```text
   <自带 node 的路径> <自带 pnpm 的路径>/bin/pnpm.mjs install
   ```

4. 重启 DeepSeek Harness。默认启用调试日志，可在 `%USERPROFILE%\.dsh\dsh-desktop-notify.log` 中查找 `plugin loaded` 确认插件已加载。

也可以把上述安装步骤交给 DSH 中的 agent 执行。实际通知和点击跳转还需要在桌面环境中验证。

## 配置

默认配置见 [cordis.patch.yml](cordis.patch.yml)。在自己的 profile 配置中添加覆盖项；以下示例关闭完成提醒、将最短回合时间设为 10 秒，并允许卡片在非全屏时显示：

```yaml
- id: desktop-notify
  name: dsh-desktop-notify
  config:
    classes:
      completion: false
    minTurnMs: 10000
    alertOnlyWhenFullscreen: false
```

| 配置项 | 默认值 | 作用 |
| --- | --- | --- |
| `enabled` | `true` | 通知总开关 |
| `classes` | 各类均为 `true` | 分别控制 `approval`、`question`、`completion`、`failure` 和 `aborted` |
| `channels.toast` / `channels.alert` | 均为 `true` | 开关原生通知和置顶卡片 |
| `alertOnlyWhenFullscreen` | `true` | 双通道开启时，卡片通常只在前台全屏时显示 |
| `alertSeconds` | 按类别设置 | 卡片停留时间；`0` 表示等待点击 |
| `gateOnFocus` | `true` | 正在查看对应对话时抑制提醒 |
| `minTurnMs` | `5000` | 过滤短于该时长的完成提醒，单位毫秒 |
| `dedupeMs` / `waitingDedupeMs` | `8000` / `15000` | 普通事件和等待事件的去重窗口，单位毫秒 |
| `recheckMs` | `1800` | 授权或提问被焦点门控抑制后，短时间内切走则补发；`0` 关闭 |
| `notifySubagents` | `false` | 是否通知子代理会话 |
| `clickToFocus` / `sound` / `debug` | 均为 `true` | 点击跳转、提示音和调试日志 |

更多配置及说明见 [cordis.patch.yml](cordis.patch.yml)。

## 排查与限制

- **没有通知**：先确认插件已加载、DSH 已重启，再查看 `%USERPROFILE%\.dsh\dsh-desktop-notify.log`。日志会记录焦点抑制、去重和投递等判断。
- **点击后没有切换对话**：查看 `%USERPROFILE%\.dsh\dsh-desktop-notify.activate.log` 和主日志中的 `activation claimed by client`。滚动到最新消息是尽力而为，DSH 界面变化可能影响它。
- **全屏场景**：Windows 可能压制原生横幅；插件会尝试以置顶卡片补位。某些应用的窗口可能被误判为全屏。
- **版本兼容**：插件依赖 DSH 的事件和服务接口；DSH 更新后可能需要适配。
- 修改插件的 JavaScript 文件后需重启宿主，才能重新加载代码和路由。

## 卸载

从 profile 的 `dependencies` 和 `dsh.profile.bundles` 中移除 `dsh-desktop-notify`，在 profile 目录重新执行 pnpm install，然后重启 DSH。插件运行时可能在当前用户注册表中创建通知身份和 COM 激活器：

```text
HKCU\Software\Classes\AppUserModelId\DeepSeekHarness.Notify
HKCU\Software\Classes\CLSID\{D7A1F0B2-3C4D-4E5F-9A0B-1C2D3E4F5A6B}
```

如需清理这些注册项，请先确认它们未被其他程序使用。

## 许可与致谢

本项目采用 [MIT 许可证](LICENSE)，保留原项目的版权声明。

本项目基于 [VoodooB0Ys/dsh-desktop-notify](https://github.com/VoodooB0Ys/dsh-desktop-notify)。感谢原作者及原项目致谢的社区贡献者：[dsh-task-ask-notify](https://github.com/deadbushxw/dsh-task-ask-notify)、[dsh-donevoice](https://github.com/zywnb-2/dsh-donevoice) 和 [dsh-task-reminder](https://github.com/hawkongz/dsh-task-reminder)。
