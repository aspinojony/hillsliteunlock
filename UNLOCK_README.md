# Hills Lite Pro 无感解锁说明

> 适用脚本：`unlock_auto.py`  
> 目标应用：Microsoft Store **Hills Lite**（`Mountains.HillsLite`）  
> 验证版本：约 **1.3.1.x**（`windows_iap_plugin.dll` 偏移基于此版本）  
> 文档日期：2026-07

---

## 1. 这是什么

安装一次后，**直接从开始菜单 / 任务栏 / 搜索打开 Hills Lite** 即可自动处理 Pro 状态，无需每次手动跑脚本，也无需专用 bat。

核心脚本：

| 文件 | 作用 |
|------|------|
| `unlock_auto.py` | 无感解锁主程序（安装 / 监视 / 注入 / 卸载） |
| `unlock_auto.log` | 运行日志（同目录自动生成） |

相关逆向笔记可参考同目录或用户目录下的 `hills_re_writeup.md`（若存在）。

---

## 2. 环境要求

- Windows 10 / 11 **64 位**
- 已从微软商店安装 **Hills Lite**
- **Python 3.10+ 64 位**
- Python 包：**frida**（`pip install "frida>=16,<18"`）

```powershell
python -m pip install "frida>=16,<18"
```

---

## 3. 快速使用

### 3.1 本机安装（推荐）

在脚本所在目录执行：

```powershell
cd C:\Users\86166\hills_re
python unlock_auto.py --install
```

安装会做这些事：

1. 写入 **开机启动**（Startup 下的 `HillsLiteProUnlock.vbs`）
2. 创建计划任务 **`HillsLiteProUnlock`**（登录时启动监视）
3. 尽量注册 **IFEO**（拦截 `HillsLite.exe` 启动，用户级 HKCU）
4. **seed** 本地 Pro 偏好设置
5. 立即拉起后台 **`--watch`** 监视进程

### 3.2 日常使用

安装成功后：

- 正常点开 **Hills Lite** 即可  
- 不必再运行 bat / 不必手动 attach  

### 3.3 卸载

```powershell
python unlock_auto.py --uninstall
```

会移除计划任务、Startup VBS、IFEO Debugger 等。若监视进程仍在，可在任务管理器结束命令行含 `unlock_auto` 的 `python.exe`。

---

## 4. 命令行参数

```text
python unlock_auto.py --install     # 安装开机自启 + IFEO + 写 prefs + 启动监视
python unlock_auto.py --uninstall   # 卸载上述配置
python unlock_auto.py --watch       # 前台/后台监视：发现 HillsLite.exe 就挂钩（常驻）
python unlock_auto.py --launch      # 主动 kill 旧进程并用 frida spawn 带钩启动
python unlock_auto.py --seed        # 仅写入本地 Pro prefs，不启动应用
python unlock_auto.py --ifeo ...    # IFEO 调试器入口（系统自动调用，勿手点）
```

| 参数 | 说明 |
|------|------|
| `--install` | 一键部署无感方案 |
| `--uninstall` | 清理部署 |
| `--watch` | 轮询进程列表，对新 `HillsLite.exe` 做 **单次 attach** 并保持会话 |
| `--launch` | **spawn** 挂起进程 → 注入脚本 → resume（最容易吃到首轮许可证请求） |
| `--seed` | 只改 `shared_preferences.json` |
| `--ifeo` | 被 IFEO 拉起时用；内部再 spawn/attach |

---

## 5. 原理概览

应用会员链路大致为：

```text
Dart ProCubit / LocalEntitlementVerifier
        ↑
  getAddonLicenses / checkPurchase 等
        ↑
  windows_iap_plugin.dll  (Store 许可证)
        ↑
  可选：api.hills.im 在线校验
```

本脚本从三层同时动手：

### 5.1 本地偏好（离线信号）

路径（自动按 Package Family 拼接）：

```text
%LOCALAPPDATA%\Packages\Mountains.HillsLite_jja6j3cje9wfy\
  LocalCache\Roaming\com.mountains\Hills\shared_preferences.json
```

写入（示例字段）：

| Key | 值含义 |
|-----|--------|
| `flutter.isPro` | `true` |
| `flutter.hasValidEntitlement` | `true` |
| `flutter.microsoftStoreIdKey` | 本地占位 store id |
| `flutter.lastEntitlementConfirmedAt` / `...Ms` | 当前时间戳 |
| `flutter.addonLicenses` | JSON 数组，SKU **`9NPROHILLS01`**，`inCollection/isActive=true` |

首次 seed 前若原文件存在，可能备份为 `shared_preferences.json.bak_unlock`。

### 5.2 Frida 运行时挂钩（`windows_iap_plugin.dll`）

在 IAP 模块上：

| RVA（约 1.3.1） | 作用 |
|-----------------|------|
| `0x89B0` | `inCollection` 类包装：`onLeave` 强制返回 1 |
| `0x89E0` | `isTrial` 类包装：强制返回 0 |
| `0xC080` | 解析方法名，识别 `getAddonLicenses` / `checkPurchase` 等并 arm |
| `0xD0D0` | 协程路径 arm licenses |
| `0x14C14` | **IAP 专用** SendResponse 调用点：改写返回缓冲区 |

许可证假数据（StandardMethodCodec 风格）：

- 成功 + **Map**：`{ "9NPROHILLS01": { inCollection, isActive, isTrial, storeId, ... } }`  
- 空许可证常见原文：`Success({})` → 字节头约 `00 0d 00`  
- `checkPurchase` → `Success(true)`  
- 另可对 `getCustomerCollectionsId` 返回空成功包  

**Boot 窗口**：attach 后约 20s 内，即使错过方法名，只要在 `0x14C14` 看到空 Map，也会按 licenses 注入（该点仅 IAP 使用，避免误伤其它 Flutter 插件）。

**DNS**：对主机名含 `hills.im` 的 `getaddrinfo` 返回失败，减轻服务端撤销影响。

### 5.3 无感启动两条路径

```text
用户点 Hills Lite
       │
       ├─► IFEO（若生效）→ python unlock_auto.py --ifeo → spawn + 注入
       │
       └─► 正常启动进程
                │
                └─► 后台 --watch 发现 HillsLite.exe → attach + 注入
```

| 路径 | 优点 | 注意 |
|------|------|------|
| IFEO | 启动即注入，易赶上首轮 `getAddonLicenses` | 商店 AppX 启动有时不走 IFEO |
| `--watch` | 兼容从开始菜单打开 | 需开机已运行；attach 略晚时靠 prefs + boot 空 Map |

单 PID 互斥（`Local\HillsLiteHookPid-<pid>`），避免 IFEO 与 watch **双注入**导致崩溃。  
全局只允许一个 `--watch`（`Local\HillsLiteProUnlockWatchdog`）。

---

## 6. 安装后系统改动

| 项目 | 位置 / 名称 |
|------|-------------|
| Startup | `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\HillsLiteProUnlock.vbs` |
| 计划任务 | `HillsLiteProUnlock`（ONLOGON） |
| IFEO | `HKCU\...\Image File Execution Options\HillsLite.exe` → `Debugger` |
| 日志 | 脚本目录 `unlock_auto.log` |
| Prefs | 见 §5.1 |

卸载：`python unlock_auto.py --uninstall`。

---

## 7. 日志怎么看

`unlock_auto.log` 中常见行：

| 日志 | 含义 |
|------|------|
| `auto-unlock watchdog running` | 监视已启动 |
| `watch hook thread for <pid>` | 发现进程并准备挂钩 |
| `attach <pid>` / `hooks on <pid>` | Frida 已注入 |
| `iap@0x...` | 找到 `windows_iap_plugin.dll` |
| `HOOKS READY` | JS 钩子装好 |
| `expect licenses (...)` | 即将拦截许可证回复 |
| `IAP-REPLY kind=licenses ...` | 看到 IAP 回包 |
| `INJECT #n licenses-map` | **成功注入假许可证** |
| `DNS-BLOCK api.hills.im` | 拦截校验域名 |
| `detached ... process-terminated` | 应用退出或崩溃 |
| `skip <pid>: already hooked` | 已有会话，跳过重复注入 |

**成功标志**：出现 `INJECT #1 licenses-map`，且进程保持存活；prefs 中 `flutter.isPro == true`。

---

## 8. 历史问题与对策（实现时踩过的坑）

| 问题 | 原因 | 对策 |
|------|------|------|
| 一点开就闪退 | Frida **probe attach 再 detach 再 attach** 双注入；或 sticky 误改写 | 只 attach 一次；空 Map 仅在 IAP 站点 + boot 窗口处理 |
| 监视进程自己退出 | `pythonw` / 脆弱的 VBS 子进程 | 用 `python.exe` + `DETACHED_PROCESS` / `cmd start` 拉起 |
| 有 hooks 无 INJECT | attach 晚于首轮 `getAddonLicenses` | `--seed` prefs；boot 空 Map；可 `--launch` spawn |
| SKU 写错 | 曾写成应用本体 `9NXNZFRLLWZX` | 统一 Pro SKU **`9NPROHILLS01`** |
| 双 watch 抢同一进程 | 无互斥 | PID 级 + 全局 Mutex |

---

## 9. 其它脚本（同目录，非无感主路径）

以下多为调试 / 一次性方案，**日常请以 `unlock_auto.py` 为准**：

| 脚本 | 说明 |
|------|------|
| `unlock_persist.py` | 旧常驻方案，**已废弃** |
| `unlock_iap_site.py` | 精确 IAP SendResponse 点验证 |
| `unlock_spawn.py` / `unlock_final.py` 等 | 早期 spawn / 注入试验 |

离线补丁产物（若存在）：

- `windows_iap_plugin.patched.dll`：把 `0x89B0`/`0x89E0` 改成恒真/恒假  
- **不能**直接写 `WindowsApps`（ACL / TrustedInstaller），故运行时 Frida 更现实  

---

## 10. 一键安装包（拷到其它电脑）

若使用打包目录 `HillsLiteProUnlock_Installer`：

```text
install.bat      # 一键安装（可自动下便携 Python + frida）
uninstall.bat
status.bat
unlock_auto.py
README.txt
```

前提：目标机已从商店安装 Hills Lite。  
安装落盘：`%LOCALAPPDATA%\HillsLiteProUnlock\`。

---

## 11. 故障排查清单

1. **Hills Lite 是否在跑**  
   任务管理器看 `HillsLite.exe`。
2. **监视是否在跑**  
   命令行含 `unlock_auto.py --watch` 的 `python.exe`。  
   没有则：`python unlock_auto.py --install` 或 `--watch`。
3. **看日志**  
   有无 `HOOKS READY` / `INJECT` / `frida-error` / `detached`。
4. **prefs**  
   `flutter.isPro` 是否为 true，SKU 是否为 `9NPROHILLS01`。
5. **杀软**  
   是否拦截 Frida / 结束 python。
6. **应用大版本升级**  
   DLL RVA 可能全变 → 需重新逆向改偏移后再用。

手动补救：

```powershell
python unlock_auto.py --seed
python unlock_auto.py --launch
```

---

## 12. 免责与范围

- 本文档描述的是本地逆向与运行时挂钩的**技术实现**，仅供学习研究。  
- 请支持正版软件。  
- 勿将解锁包公开发布到 GitHub 等平台传播。  
- 不同版本 Hills Lite 不保证兼容。

---

## 13. 关键路径速查

```text
脚本目录（示例）
  C:\Users\86166\hills_re\unlock_auto.py
  C:\Users\86166\hills_re\unlock_auto.log

安装包（示例）
  C:\Users\86166\HillsLiteProUnlock_Installer\
  C:\Users\86166\HillsLiteProUnlock_Installer.zip

运行时安装目录（安装包模式）
  %LOCALAPPDATA%\HillsLiteProUnlock\
```

---

## 14. 版本备注

| 项 | 值 |
|----|-----|
| Package Family | `Mountains.HillsLite_jja6j3cje9wfy` |
| AppId | `hillsdesktop` |
| Pro SKU | `9NPROHILLS01` |
| 计划任务名 | `HillsLiteProUnlock` |
| 依赖 | Python 3.10+、frida 16/17.x |

应用更新后若失效，优先检查：`windows_iap_plugin.dll` 是否仍存在上述 RVA 行为，以及 prefs 是否被应用覆盖。
