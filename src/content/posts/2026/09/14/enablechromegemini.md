---
title: "解决 Chrome「问问 Gemini」点击无反应：从内部诊断页定位地区限制与进程缓存问题"
description: "从 chrome://glic/internals 入手，系统排查 Gemini in Chrome 的账号、语言、地区和启动参数问题，并提供 Windows 临时启动与安全回滚方法。"
published: 2026-09-14T15:34:50+08:00
updated: 2026-09-17T00:00:00+08:00
category: "技术"
subcategory: "AI 与自动化"
tags: ["Gemini", "Chrome", "Windows", "故障排查"]
draft: false
---

不少人升级 Chrome 后发现顶部出现了「问问 Gemini」按钮，但点下去没有任何反应。遇到这种情况，先别急着去改 Chrome 配置文件（`Local State`），或者反复折腾代理规则。

最省时间的做法是先打开 Chrome 内置的诊断页面，看看到底是**账号没资格**、**语言不对**，还是**被客户端的地区缓存拦住了**。只有找准了卡在哪一步，后面的处理才不会做无用功。

这篇文章整理了一套在 Windows 环境下实测有效的排查与修复方案，主要解决下面这几个常见痛点：

- 顶部有「问问 Gemini」按钮，但点击后毫无反应；
- 代理出口明明在受支持地区，Chrome 却依然判定为 `cn`（中国大陆）；
- 在桌面快捷方式里加了启动参数，但打开浏览器后参数根本没生效；
- 手动改了 `Local State` 配置文件，刚打开能用，重启浏览器又被打回原形。

> **提前说明**：Gemini in Chrome 目前仍在分批推送。下文提到的启动参数属于 Chromium 原生的测试与诊断机制，**只能解决客户端本地的地区误判**，无法绕过 Google 账号本身的灰度资格或企业管理员限制。

## 一、先用内置诊断页揪出真正的拦截项

与其靠猜，不如直接看 Chrome 自己的判断结果。在出问题的 Chrome 窗口地址栏输入并打开：

```text
chrome://glic/internals
```

*(注：地址中间是斜杠 `/`，不是连字符 `chrome://glic-internals`。)*

这个页面会列出当前浏览器启用 Gemini 的所有前置条件检查结果。拿到页面后，重点核对下面这几项：

| 检查项 | 含义与排查说明 |
| :--- | :--- |
| `Enabled by Chrome Flags` | 相关功能开关是否已在 `chrome://flags` 中启用 |
| `Regular profile` | 当前是否为正常的个人配置（无痕模式、访客模式或受限配置无法使用） |
| `Pref or flag based rollout applies` | 当前浏览器版本是否已命中官方灰度推送（Rollout） |
| `Account exists and has the Gemini in Chrome capability` | 当前登录的 Google 账号本身是否具备使用资格 |
| `Account exists and is fully signed-in` | 账号是否已在 Chrome 中完整登录并开启同步（不能处于登录过期状态） |
| `Server side allows this feature` | Google 服务器端是否允许当前账号启用该功能 |
| `Passed locale filter` | 浏览器界面语言是否符合要求（目前通常要求为 `en-US`） |
| `Passed country filter` | 当前识别到的国家/地区是否在支持列表中 |
| `Permanent Country Code` | Chrome 本地持久化保存的长期国家代码 |
| `Session Country Code` | 本次启动会话识别到的国家代码 |

### 典型案例：为什么账号有资格，按钮还是点不动？

在很多国内用户的排查结果里，账号资格、登录状态、语言设置和服务器许可全都是 `true`，唯独卡在最后三项地区指标上：

```text
Passed country filter: false
Permanent Country Code: cn
Session Country Code: cn
```

这就是按钮点不动的直接原因：**Chromium 在代码层面把账号权限、语言和地区过滤拆成了独立的校验关卡**。哪怕你的 Google 账号已经拿到资格、界面也显示出了按钮，只要本地的 `Passed country filter` 为 `false`，点击面板时就会被客户端直接拦截。

### 为什么挂了美国代理，Chrome 还是显示 `cn`？

很多人会纳闷：我已经开了全局代理，IPv4/IPv6 出口都在美国，DNS 和 WebRTC 也没有泄漏，网页版 `gemini.google.com` 都能正常对话，为什么 `chrome://glic/internals` 里还是显示 `cn`？

这是因为 Chrome 判断地区并不只看你当前的实时公网 IP。它底层的 **Variations 服务**会综合参考本地保存的“长期一致性国家代码（`Permanent Country Code`）”、上一次启动会话记录的国家，以及服务器下发的缓存状态。你临时切了网络出口，Chrome 不会立刻刷新内部的 Variations 国家标记，往往需要跨版本更新或通过特定参数强制覆盖才会生效。

## 二、动手改配置前，先做一遍常规排查

如果诊断页里不仅地区没过，其他项目也有 `false`，建议先按官方要求把基础条件补齐：

1. **检查基础门槛**：确保使用的是最新版 Chrome（Windows、macOS 或 Chromebook Plus），且处于普通窗口而非无痕模式。
2. **确认账号与权限**：确保已完整登录 Google 个人账号。如果是工作或学校账号（Google Workspace），需要单位管理员在后台开启权限。
3. **切换界面语言**：将 Chrome 的显示语言设为英语（美国 `en-US`），并完全重启浏览器。
4. **重置实验性开关**：如果之前乱改过 `chrome://flags`，建议先点页面右上角的「Reset all」恢复默认并重启。
5. **刷新面板**：如果所有条件都是 `true` 但面板空白，可以在 Gemini 侧边栏区域右键，选择「重新加载（Reload）」。

如果以上步骤做完后，诊断页里**只有 `Passed country filter` 依然是 `false`**，就可以用下面的启动参数或脚本来解决地区缓存问题了。

## 三、核心原理：为什么改配置文件和加快捷方式经常失效？

在介绍一键工具之前，先花两分钟讲清楚两个很多人踩过的坑——为什么网上的常规教程到了自己电脑上就不灵了。

### 1. 为什么给快捷方式加了启动参数，打开却没反应？

解决地区限制最直接的 Chromium 原生参数是下面这两个：

```text
--variations-override-country=us
--disable-features=GlicCountryFiltering
```

- `--variations-override-country=us`：强制把本次启动会话的 Variations 国家覆盖为美国（`us`），仅对当次会话生效。
- `--disable-features=GlicCountryFiltering`：在当前会话中关闭 Gemini in Chrome 的客户端国家过滤校验。

原理很简单，但很多人把这两个参数加进桌面 Chrome 快捷方式的「目标」栏后，双击打开却发现一点用都没有。

原因在于 **Chrome 的单主进程机制**：Windows 开机或后台运行时，往往已经有一个不带参数的 Chrome 主进程在后台驻留了（比如系统自动拉起的 `chrome.exe --no-startup-window --restart --restore-last-session`）。这时候你双击桌面上的带参数快捷方式，系统并不会启动一个新的主进程，而是直接让后台那个旧主进程帮你开了一个新窗口——**你写的启动参数直接被忽略了**。

要让启动参数真正生效，必须在启动前**彻底杀掉所有正在运行的 `chrome.exe` 进程**，再由带参数的命令拉起第一个主进程。

### 2. 为什么手动修改 `Local State` 重启后又被还原？

有些教程会让你用记事本打开用户目录下的 `Local State` 和 `Preferences` 文件，手动改这几个字段：

```text
variations_country = us
variations_permanent_consistency_country = [..., "us"]
intl.app_locale = en-US
is_glic_eligible = true
```

这样改有两个明显的隐患：
- **会被服务器状态覆盖**：Chrome 联网运行后，会根据当前账号和 Variations 服务重新计算资格，把你改好的本地字段覆盖回去。
- **容易损坏配置**：如果 Chrome 还没完全退出就直接修改 JSON 文件，很容易引发文件写入冲突，导致配置损坏甚至扩展数据丢失。

## 四、实操方案：使用脚本一键配置与独立启动

为了省去每次手动杀进程、备份配置、敲命令行的麻烦，可以使用开源的 **[Gemini in Chrome Toolkit](https://github.com/yyzmia1/gemini-in-chrome-toolkit)**。它提供了相互独立的 PowerShell 和 Python 脚本，帮你自动完成：**关闭残留进程 -> 备份原配置 -> 写入美区语言与地区标记 -> 生成带自动清进程功能的桌面启动器**。

> **重要提醒**：无论是首次运行启用脚本，还是日常双击生成的专用快捷方式，脚本都会**先强制关闭当前所有 Chrome 窗口和后台进程**。操作前请务必保存好网页里没写完的表单、在线文档，并等待正在下载的文件完成！

### 方式 A：使用 PowerShell 脚本（Windows 推荐）

确保系统已安装 PowerShell 7.4 或更高版本。下载仓库后，在项目根目录打开终端执行：

```powershell
# 启动交互式菜单（可按数字选择：1.启用并启动 / 2.恢复最新备份 / 3.查看状态 / 0.退出）
pwsh -NoProfile -File .\scripts\gemini_chrome.ps1
```

如果习惯直接用命令行，也可以加上 `-Action` 参数一键执行：

```powershell
# 备份配置、写入美区参数并启动 Chrome
pwsh -NoProfile -File .\scripts\gemini_chrome.ps1 -Action Enable

# 查看当前配置与状态
pwsh -NoProfile -File .\scripts\gemini_chrome.ps1 -Action Status

# 从最新备份恢复原始配置
pwsh -NoProfile -File .\scripts\gemini_chrome.ps1 -Action Restore
```

### 方式 B：使用 Python 脚本

如果电脑里装了 Python 3.10 或更高版本，也可以直接跑 Python 脚本，效果完全一致：

```bash
# 启动交互式菜单
python .\scripts\gemini_chrome.py

# 或者直接指定操作：
python .\scripts\gemini_chrome.py enable
python .\scripts\gemini_chrome.py status
python .\scripts\gemini_chrome.py restore
```

### 脚本在后台具体做了什么？

当你执行 `Enable`（启用）时，脚本会严格按以下安全顺序执行，不会乱动你的注册表或企业管理策略：

1. 自动定位本机 Chrome 安装路径与 `User Data` 用户数据目录。
2. 提示确认后，彻底关闭所有 `chrome.exe` 进程，避免文件读写冲突。
3. 将原始的 `Local State` 和所有常规 Profile 的 `Preferences` 完整备份到 `%USERPROFILE%\Documents\GeminiInChromeToolkit\backups\<时间戳>`。
4. 通过临时文件安全替换的方式，将 Variations 国家设为 `us`、界面语言设为 `en-US`、接受语言设为 `en-US,en`，并将已存在的 `is_glic_eligible` 字段设为 `true`（不会伪造虚假字段）。
5. 在 `%LOCALAPPDATA%\GeminiInChromeToolkit\launch_gemini_chrome.cmd` 生成一个稳定启动器，并在桌面创建名为 **`Chrome - Gemini US`** 的快捷方式。

### 日常怎么用最省心？

首次运行完 `Enable` 后，你桌面上就会多出一个 **`Chrome - Gemini US`** 快捷方式。以后每次想用 Gemini 侧边栏时：

1. 保存好当前浏览器里的未提交内容。
2. 直接双击桌面的 **`Chrome - Gemini US`**。
3. 启动器会自动帮你清理后台残留的 Chrome 进程，并挂上 `--variations-override-country=us` 和 `--disable-features=GlicCountryFiltering` 参数重新拉起浏览器。

如果以后不小心删掉了桌面快捷方式，或者不想再用这个快捷方式了，可以用下面的命令单独管理快捷方式（不会影响你的配置备份）：

```powershell
# 单独重新生成桌面快捷方式（PowerShell / Python 二选一）
pwsh -NoProfile -File .\scripts\gemini_chrome.ps1 -Action Shortcut
python .\scripts\gemini_chrome.py shortcut

# 删除桌面快捷方式与启动器（PowerShell / Python 二选一）
pwsh -NoProfile -File .\scripts\gemini_chrome.ps1 -Action RemoveShortcut
python .\scripts\gemini_chrome.py remove-shortcut
```

## 五、如何验证参数已经真正生效？

通过 `Chrome - Gemini US` 启动浏览器后，花十秒钟做个两步验证：

### 第一步：看进程命令行

在地址栏打开：

```text
chrome://version
```

找到页面里的**「命令行（Command Line）」**一栏，确认里面明确包含下面这两段参数：

```text
--variations-override-country=us --disable-features=GlicCountryFiltering
```

如果没看到这两个参数，说明刚才启动时后台还有没杀干净的 Chrome 进程把窗口接管了，彻底关掉浏览器再双击一次 `Chrome - Gemini US` 即可。

### 第二步：复查内部诊断页

再次打开：

```text
chrome://glic/internals
```

正常情况下，此时的 `Session Country Code` 会变成 `us`，`Passed country filter` 也会变成 `true`。这时候再去点浏览器右上角的「问问 Gemini」，侧边栏就能正常加载出来了。

**如果所有检查项都已经变成 `true`，但点开面板还是空白怎么办？**
可以按下面顺序排查网络与插件干扰：
1. 新开一个标签页，直接访问 `https://gemini.google.com/`，确认当前代理节点能正常打开网页版 Gemini；
2. 在面板空白处点击右键，选择「重新加载」；
3. 暂时关闭广告拦截器或脚本类扩展插件后重试；
4. 如果用的是公司或学校邮箱，切换到个人 Gmail 账号测试，排除组织策略拦截。

## 六、随时恢复原状（无损回滚）

如果你后续不再需要这套配置，或者想把浏览器彻底还原到折腾前的状态，非常简单：

- **停用启动参数**：直接关闭 Chrome，以后用原本的普通 Chrome 快捷方式打开就行。启动参数只对当次会话生效，不会永久驻留。
- **还原配置文件**：运行下面的 `Restore` 命令，脚本会自动找到时间戳最新的有效备份，把 `Local State` 和 `Preferences` 还原回去。

```powershell
# PowerShell 恢复命令
pwsh -NoProfile -File .\scripts\gemini_chrome.ps1 -Action Restore

# Python 恢复命令
python .\scripts\gemini_chrome.py restore
```

恢复操作只覆盖最初备份的那几个状态配置文件，**不会删除你在此期间新建的书签、浏览历史、安装的扩展或新添加的 Profile**。

## 附：常见症状速查表

最后放一张速查表，遇到问题时直接对号入座：

| 遇到的现象 | 根本原因与首选排查动作 |
| :--- | :--- |
| 浏览器顶部完全没有 Gemini 按钮 | 账号未命中灰度、浏览器版本过旧，或界面语言不是 `en-US` |
| 按钮存在，但点击后毫无反应 | 打开 `chrome://glic/internals`，90% 是 `Passed country filter` 为 `false` |
| 挂了美国全局代理，诊断页依然显示 `cn` | Variations 服务缓存了旧的国家代码，需配合启动参数覆盖会话国家 |
| 快捷方式加了启动参数，但打开不生效 | 后台有残留的 `chrome.exe` 主进程接管了新窗口，必须先结束全部进程再启动 |
| 手动改了 `is_glic_eligible` 重启又失效 | 联网后被 Google 服务器下发状态重新计算覆盖，不要强改本地 JSON，改用启动参数 |
| 诊断页全绿（均为 `true`），但侧边栏空白 | 检查 `gemini.google.com` 连通性、右键重新加载面板、排查扩展冲突或切换个人账号 |

## 参考资料

- [Google Chrome 帮助：使用 Chrome 中的 Gemini](https://support.google.com/chrome/answer/16283624?hl=zh-Hans)
- [Google Chrome 帮助：Chrome 中的 Gemini 支持范围](https://support.google.com/chrome/answer/17140089?hl=zh-Hans)
- [Chromium：Variations 国家覆盖参数](https://chromium.googlesource.com/chromium/src/+/main/components/variations/variations_switches.cc)
- [Chromium：Gemini in Chrome 启用条件实现](https://chromium.googlesource.com/chromium/src/+/main/chrome/browser/glic/public/glic_enabling.cc)
- [Chromium：创建 `chrome://glic/internals` 的变更记录](https://chromium.googlesource.com/chromium/src/+/854c23b412aaccb4172d27dc727b5d2891fd02be)
- [GitHub：Gemini in Chrome Toolkit](https://github.com/yyzmia1/gemini-in-chrome-toolkit)
