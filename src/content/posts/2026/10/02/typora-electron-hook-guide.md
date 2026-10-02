---
title: "Typora 从零激活：一键脚本、手动改造与原理剖析"
published: 2026-10-02T21:50:00+08:00
description: "记录一套完整的 Typora 激活方案，涵盖 Windows 全自动部署脚本、手动改造分步执行与底层技术原理（Electron fuse、asar 解包、JS 钩子拦截与 fs 重定向绕过）。"
category: "技术"
subcategory: "开发与编程"
tags: ["Typora", "Electron", "逆向", "Windows", "自动化"]
draft: false
pinned: false
comment: true
lang: "zh_CN"
---

> 项目地址：**https://github.com/yyzmiao/typora-electron-hook** ｜ 使用文档见 [README.md](https://github.com/yyzmiao/typora-electron-hook#readme)
>
> 本文记录一套完整的 Typora 激活方案。文中的一键脚本已开源，手动流程与技术原理一并给出，
> 你可以只用工具，也可以完全自己动手。

---

## 写在前面

Typora 从 1.0 版本起转为付费授权。它的授权校验跑在 Electron 主进程里，代码打包在 `resources/app.asar` 中——这意味着存在一条不修改二进制、只通过注入 JS 钩子完成激活的路径。

我把这条路径做成了全自动工具，也在过程中踩了一堆坑。这篇文章分三部分：

1. **无脑一键**：三分钟完成，不需要任何技术背景
2. **手动改造**：不用脚本，一步步自己操作
3. **技术原理**：它到底动了什么，为什么有效

原始思路来自 [CSDN 上这篇教程](https://blog.csdn.net/m0_62312746/article/details/163153769)，本项目是在其基础上做工程化实现与问题修复，末尾有完整致谢。

---

## 一、无脑一键：三分钟搞定

### 1.1 准备工作

需要两样东西：

- **已安装的 Typora**（任意版本 1.x，安装位置不限）
- **网络连接**（用于拉取两个 Node 依赖包）

Node.js **不需要预先安装**，脚本检测不到时会尝试用 `winget` 自动装。

从 [项目地址](https://github.com/yyzmiao/typora-electron-hook) 下载两个文件，**放在同一个文件夹里**：

```
TyporaActivator.ps1     主程序
TyporaActivator.cmd     启动器
```

> 这两个文件必须同目录，启动器靠相对路径找主程序。分开放会提示找不到文件。

### 1.2 操作步骤

1. **双击 `TyporaActivator.cmd`**
2. UAC 弹窗点「是」
3. 出现黑窗口后，**不要碰鼠标键盘**
4. 等待，中途它会自动打开两次 Typora（第一次取机器码，第二次做验证），每次十几秒
5. 看到绿色大字 **激活成功** 即可

全程约一分钟。除一次 UAC 确认外没有任何需要你输入的地方。

### 1.3 结果确认

打开 Typora，进入：

```
帮助 → 我的许可证
```

页面显示 **Typora 已激活** 即完成。

如果想从命令行确认：

```powershell
Get-ItemProperty "HKCU:\Software\Typora" | Format-List SLicense,IDate
```

`SLicense` 有值且重启 Typora 后**依然在**，说明授权已被接受；被清空则说明没通过。

### 1.4 脚本会做什么

```
定位 Typora 安装目录
  → 检查并准备 Node.js
  → 关闭正在运行的 Typora
  → 备份可执行文件、应用包与注册表
  → 准备 Node 依赖（优先复用已有的）
  → 阶段 1：注入机器码抓取器，启动 Typora 取回机器码
  → 阶段 2：注入授权钩子
  → 写入注册表授权信息
  → 启动 Typora 并验证结果
```

脚本是幂等的，重复运行不会出问题，每次都会重新抓一遍机器码。

### 1.5 两个必须知道的注意事项

**关掉自动更新。** 设置 → 通用 → 自动检查更新 → 关闭。Typora 更新会重装 `Typora.exe` 并覆盖 `resources\app\`，注入的钩子会被清掉。

**别点「取消激活」。** 许可证页面底部那个按钮会清空 `SLicense`，需要重新跑一遍。

### 1.6 回滚

脚本每次运行前会备份到 `%USERPROFILE%\Typora-Backup-<时间戳>\`。想恢复原版：

```powershell
$t = "C:\Program Files\Typora"
Copy-Item "$t\Typora.exe.bak" "$t\Typora.exe" -Force
Remove-Item "$t\resources\app" -Recurse -Force
if (Test-Path "$t\resources\app.asar.bak") {
    Move-Item "$t\resources\app.asar.bak" "$t\resources\app.asar"
}
Remove-Item "HKCU:\Software\Typora" -Recurse -Force
```

---

## 二、手动改造：不用脚本一步步来

如果你想自己动手，或者想知道脚本背后到底在干什么，跟着下面走一遍。所有命令都在**管理员权限的 PowerShell** 里执行。

### 2.1 取机器码

打开 Typora，依次进入：

```
帮助 → 我的许可证 → 输入序列号 → 离线激活 → 复制机器码
```

机器码是一段 Base64 字符串，解码后是这样：

```json
{"v":"win|1.14.10","i":"5hrrhZ0aY0","l":"DESKTOP | User | Windows"}
```

- `l`：设备标识（机器名 | 用户名 | 系统）
- `i`：指纹标识（硬件相关）
- `v`：版本标识

**先复制保存下来**，后面第 2.6 步要用。

> 手动模式下这一步躲不掉。自动模式之所以能免去它，是因为它在主进程里直接调用了 Typora 自己产机器码的那个接口——原理见第三节。

### 2.2 准备 Node.js

```powershell
node -v
```

有输出就跳过。没有的话：

```powershell
winget install OpenJS.NodeJS.LTS
```

装完如果 `node -v` 仍然不认，重开一个 PowerShell 窗口（PATH 需要刷新）。

### 2.3 准备工作目录

不要往 Typora 的安装目录里塞东西——那样会污染它自己的模块解析。单独建一个：

```powershell
$Work = "$env:LOCALAPPDATA\TyporaActivator"
New-Item -ItemType Directory -Force -Path $Work | Out-Null
cd $Work
npm install asar@3.2.0 "@electron/fuses@1.8.0"
```

两个依赖为什么锁版本：`asar` 用来解包，`@electron/fuses` 用来改 electron 开关。**`@electron/fuses` 2.x 起是纯 ESM 包且要求 Node ≥ 22.12**，在较老的 Node 上 `require` 会直接失败，所以锁到 1.8.0（CommonJS）。

### 2.4 准备补丁脚本

把 [patch.js](https://github.com/yyzmiao/typora-electron-hook/blob/main/patch.js) 放到 `$Work` 目录。它接收两个参数：

```
node patch.js "<Typora安装路径>"                    注入机器码抓取器
node patch.js "<Typora安装路径>" "<机器码Base64>"    注入完整激活钩子
```

不带机器码时注入的是「抓取器」——只读取机器码并写盘，不改任何授权逻辑，所以 Typora 能正常启动。带上机器码时注入的是完整的授权钩子。

### 2.5 关闭 Typora

```powershell
Get-Process Typora -ErrorAction SilentlyContinue | Stop-Process -Force
```

文件被占用的话后面的解包和重命名会失败。

### 2.6 第一阶段：解包并抓取机器码

```powershell
cd $Work
node patch.js "C:\Program Files\Typora"
```

这一步会做四件事：

```
解包 resources\app.asar  →  resources\app\
合并 resources\app.asar.unpacked\main.node  →  resources\app\
复制 resources\app\  →  resources\app.bak\      （保留原始副本）
app.asar 改名为 app.asar.bak
翻转 Typora.exe 的 electron fuse（OnlyLoadAppFromAsar = false）
向 app\launch.dist.js 注入抓取器
```

然后启动 Typora 让它把机器码吐出来：

```powershell
Start-Process "C:\Program Files\Typora\Typora.exe"
Start-Sleep -Seconds 25
Get-Process Typora | Stop-Process -Force

Get-Content "$env:APPDATA\Typora\machine_code.txt"
```

应该看到一段 Base64。解码确认：

```powershell
$mc = (Get-Content "$env:APPDATA\Typora\machine_code.txt" -Raw).Trim()
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($mc))
```

> 如果你在第 2.1 步已经手动复制过机器码，这一步可以跳过，直接用那个值。

### 2.7 第二阶段：注入激活钩子

```powershell
cd $Work
node patch.js "C:\Program Files\Typora" $mc
```

这次注入的是完整钩子，它会覆盖 `crypto.publicDecrypt` 并劫持在线续期请求。

### 2.8 写入授权信息

```powershell
$key = "HKCU:\Software\Typora"
if (-not (Test-Path $key)) { New-Item -Path $key -Force | Out-Null }
Set-ItemProperty -LiteralPath $key -Name "SLicense" -Value "RHJlYW1OeWE=#0#1/1/2029" -Type String -Force
Set-ItemProperty -LiteralPath $key -Name "IDate" -Value (Get-Date -Format "MM/dd/yyyy") -Type String -Force
```

`SLicense` 由 `#` 分成三段：

```
RHJlYW1OeWE=  #  0  #  1/1/2029
      ↓            ↓        ↓
  响应中的 msg   状态位   上次在线校验时间
```

**第一段必须和钩子返回的 `msg` 一致。** 钩子里写的是：

```js
JSON.stringify({ success: true, msg: btoa("DreamNya") })
```

`btoa("DreamNya")` 的结果正是 `RHJlYW1OeWE=`。两者对不上，授权会被判定失效并清除。

> **注意**：网上流传的版本写的常常是 `RHJIYW1OeWE=`，那是 OCR 把 `l`（小写 L）认成了 `I`（大写 i），解出来是 `DrHamNya` 而不是 `DreamNya`。写这个值进去必然失败。

### 2.9 验证

```powershell
Start-Process "C:\Program Files\Typora\Typora.exe"
Start-Sleep -Seconds 25
Get-ItemProperty "HKCU:\Software\Typora" | Format-List SLicense
```

或者直接看界面：`帮助 → 我的许可证`。

---

## 三、技术原理

### 3.1 Typora 的授权机制

Typora 是基于 Electron 的桌面应用。授权校验跑在**主进程**里，逻辑打包在 `resources/app.asar`。校验链路大致是这样：

1. 从注册表 `HKCU\Software\Typora` 读出 `SLicense`
2. 用 `crypto.publicDecrypt` 做验签，解出授权 JSON
3. 向 `store.typora.io/api/client/renew` 请求在线续期
4. 比对解出的设备信息与本机实际信息，一致才认为有效

### 3.2 为什么可以绕过

关键在**代码的可替换性**。Electron 有一个 fuse 叫 `OnlyLoadAppFromAsar`，默认开启，强制只从 `app.asar` 加载代码。把它翻成 `false`，Electron 就允许从**未打包的目录** `resources/app/` 加载。

于是攻击面变成：把 asar 解包成目录，往入口文件里塞一段自己的 JS。这段 JS 在 Typora 自己的授权代码之前执行，因此可以：

- **覆盖 `crypto.publicDecrypt`**，让验签环节直接返回一份合法的授权 JSON
- **劫持 `https://store.typora.io/api/client/renew`**，返回伪造的成功响应

两处都拦掉，校验链就断了。

### 3.3 为什么需要两阶段

这里有个先有鸡还是先有蛋的问题：**伪造的授权 JSON 里必须包含本机真实的机器码**。

实验证明这些字段是**参与校验的**，不是展示用的装饰——把它们换成假值，Typora 会立刻清空 `SLicense` 并退回未激活状态。所以机器码必须是真的。

但机器码要通过 Typora 的界面获取（`帮助 → 我的许可证 → 离线激活`），自动化脚本没法点界面。解决方案是分两步：

**阶段一**：注入一个「安静的抓取器」。它只做一件事——拦截 `ipcMain.handle` 的注册，把 `license.machineCode` 这个 channel 的 handler 记下来，等应用启动完成后主动调用它，把结果写盘。因为它完全不碰授权逻辑，Typora 会正常启动。

**阶段二**：读到机器码后，重新注入完整的授权钩子，把真实机器码固化进去。

### 3.4 机器码是怎么自动拿到的

这是整个项目里最有意思的一块。Typora 的许可证页面通过 IPC 向主进程要机器码：

```js
window.Setting.invokeWithCallback("license.machineCode")
```

既然钩子代码本身就跑在主进程里，那么它可以在 Typora 注册这个 handler 的时候把它截下来：

```js
let mcHandler = null;
const ipcMain = electron.ipcMain;
["handle", "handleOnce"].forEach(function (method) {
    const orig = ipcMain[method].bind(ipcMain);
    ipcMain[method] = function (channel, listener) {
        if (channel === "license.machineCode") mcHandler = listener;
        return orig(channel, listener);
    };
});
```

然后在 `app.whenReady()` 之后主动调用：

```js
electron.app.whenReady().then(function () {
    setTimeout(function () {
        const raw = mcHandler({ sender: null }, undefined);
        // raw 可能是 Promise，需要 then 处理
        fs.writeFileSync(out, String(raw), "utf-8");
    }, 6000);
});
```

没有 OCR，没有剪贴板，不依赖界面。

> 这里有个时序陷阱：**不能在 `crypto.publicDecrypt` 里同步调用这个 handler**。Typora 在启动早期就会调用 `publicDecrypt`，那时 handler 可能还没注册；而且在主进程的事件链里同步重入 IPC 会把进程带崩。最初我这么写，结果 Typora 直接启动失败。正确做法是先取出来存盘，再在下一阶段固化进钩子。

### 3.5 fs 重定向为什么是必需的

钩子里有一段看似多余的代码，把对 `resources/app/` 的文件读取重定向到 `resources/app.bak/`：

```js
const FS_FROM = /resources[\\/]app[\\/]/i;
const FS_TO = "resources\\app.bak\\";
```

它**不是可选项**。Typora 会读自己目录下的文件做完整性校验，如果读到的是被我们改过的 `launch.dist.js`，会判定被篡改然后直接退出——现象是进程数 0，窗口一闪而过。

重定向之后：**完整性校验读到原始文件，实际执行的仍是注入版**。两者都满足。

我曾把这段当作冗余精简掉，结果 Typora 起来就退。加回来立刻正常。

### 3.6 几个绕不开的坑

**electron fuse 不翻转 = 打不开。** 如果 `@electron/fuses` 因为版本问题加载失败，asar 已经被改名，Typora 就找不到可执行的代码了。所以这一步失败必须**致命并回滚**，而不是记一行日志继续跑。

**npm 会向上逃逸。** 在工作目录里执行 `npm install` 时，如果该目录没有 `package.json`，npm 会一路向上找到祖先目录的 `package.json` 并把包装到那里。用户家目录里恰好有 `package.json` 的话，包装到家目录去，工作目录依然为空。解法是先立一个 `package.json` 项目边界。

**npm 和 pnpm 会互相破坏。** 同一个 `node_modules` 被两种包管理器轮流用过，它们会把对方的包挪进 `.ignored`，留下谁都不认的目录。表现是"安装报告成功、程序却找不到包"。解法是安装前清空重建。

**原生模块不能放 asar。** Typora 自带的 `native-reg` 是 `.node` 原生模块，必须位于真实目录。它只认 `build/Release/reg.node`，而 npm 装出来的是 `prebuilds/` 结构，需要手动补位。

### 3.7 验证为什么不能只看注册表

一个容易踩的坑：**`SLicense` 是脚本自己写进去的**。如果只用「SLicense 还在不在」判断成功，那么 Typora 压根没启动起来时它当然还在——于是把失败报成成功。

可靠的判据是**两个条件同时满足**：

1. Typora 进程确实存活并出现了主窗口
2. 重启后 `SLicense` 未被清除

这也是本项目在验证环节的设计。

---

## 四、常见问题

**双击 `TyporaActivator.cmd` 窗口一闪而过**

启动器会自动请求提权，原窗口随即关闭。如果 UAC 被拒绝，就不会有新窗口出现。请右键 → 以管理员身份运行。

**提示「未找到 Typora 安装目录」**

脚本会依次查七处来源（参数、App Paths 注册表、桌面/开始菜单快捷方式、卸载注册表、三个默认路径）。绿色版或装在特殊位置时可能都查不到，这时手动指定：

```powershell
.\TyporaActivator.ps1 -InstallPath "D:\Software\Typora"
```

**提示「依赖安装失败」**

脚本会依次尝试 `npm-cli.js` → `npm.cmd` → `pnpm` → `yarn` 四个方案。全部失败通常意味着网络问题，或本机包管理器环境异常。请查看日志里的具体错误。

**提示「机器码读取超时」**

通常是 Typora 首次启动较慢。手动打开一次 Typora 让它完整启动，关掉，再重跑。

**启动后报 `A required module cannot be loaded`**

Typora 自带的 `native-reg` 被破坏了（常见于在 Typora 目录里跑过 `npm install`）。脚本会自动补回，重跑一次即可。

**日志在哪**

`%TEMP%\typora-activator.log`，全流程记录。出问题时把这份日志发出来，比截图有用得多。

---

## 五、致谢

本项目的原始思路和补丁方案来自 **[CSDN @月羲花晨 的这篇教程](https://blog.csdn.net/m0_62312746/article/details/163153769)**。文中给出的 `pj.js` 方案、机器码获取路径、注册表字段结构，是这一切的起点。

我在其基础上做的工作主要是工程化：把交互式脚本改成零输入的全自动流程、修掉依赖安装与原生模块的若干问题、把「假成功」的判定改成双条件验证、补齐异常处理与回滚。

感谢原作者。

---

## 六、免责声明

本文与所附工具仅用于**本地软件结构研究与技术学习**。Typora 是付费软件，长期使用请购买正版授权。

使用者需自行承担因使用本工具产生的一切后果。请勿将本工具用于任何商业用途或二次分发牟利。
