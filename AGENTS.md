# AGENTS.md — Markor 三端项目 · AI 总入口

> **用途：AI 读完本文件即可开工，不必重读全部源码。** 详细分层文档在各端内部（见 §2）。
> 本仓库 = 三个源码包（zip）：Android / PC / Server。AI 改代码时解压/checkout 对应目录。
> 需求→改动点映射见 §4；每条需求附可行性结论。

---

## 1. 30 秒速览

| 端 | 包 | 技术栈 | 定位 |
|---|---|---|---|
| 手机端 | `Markor-Android-src.zip` → `Markor-Android/` | Java + Gradle（基于 gsantner/markor，Apache 2.0） | 笔记编辑 App，内置 Syncthing（`libsyncthing.so` 子进程，仅前台），内置同步管理 GUI（应用内 WebView） |
| PC 端 | `Markor-PC-2.17.0-src.zip` → `Markor-PC-2.17.0/` | Qt 6 Widgets + C++17 + WebView2 预览 | 桌面编辑器（Markor for Windows v2.17.0），内嵌 Syncthing（syncthing.exe 子进程）+ 文件锁客户端 |
| 服务端 | `Markor-Server-src.zip` → `Markor-Server/` | Python 标准库 + systemd + Syncthing | NAS 上：文件锁服务（lockserver:8910）+ Syncthing 同步中心（星型拓扑的"星"） |

**同步架构（三端约定，不可破坏）**：星型——手机/PC 只连服务器（NAS）；共享文件夹 ID `markor-vault`；同步根 = 笔记本目录（三端结构 `Markor/{QuickNote.md, To-Do.md, Journal/, Snippets/, file/}`）；文件锁协议 `POST /api/v1/open|heartbeat|close`（Bearer Key）。

## 2. 各端知识库（已有，优先读这些）

| 端 | AI 入口 | 详细文档 |
|---|---|---|
| PC | `Markor-PC-2.17.0/AGENTS.md` | `docs/{architecture,directory,data-model,conventions,decisions,api-index}.md` ★齐全 |
| Android | `Markor-Android/AGENTS.md`（含代码地图） | 无独立 docs/，本文件 §3 补目录图 |
| Server | `Markor-Server/AGENTS.md` | `docs/` ★齐全（README-server.md 为运维文档） |

**读码优先级（PC）**：`src/app/MainWindow.cpp`（全局接线）> `src/core/*` > `src/preview/PreviewWidget.cpp` > 其余按需。
**读码优先级（Android）**：`sync/SyncManager.java` > `activity/SettingsActivity.java` + `res/xml/preferences_master.xml` > `model/AppSettings.java` > 其余按需。

## 3. 三端目录地图（速查）

### PC（`Markor-PC-2.17.0/`，Qt/C++，55 个源文件）
```
src/main.cpp              入口：QApplication、CLI、主题预应用
src/app/       MainWindow.cpp   ★接线中枢：标签页/菜单/快捷键/导出/文件操作
               AppSettings.*    全部持久化设置（QSettings INI）——新增设置必经此处
               ThemeManager.*   主题与全部主题色出口
src/core/      Document/MarkdownConverter/FormatDetector/TodoTxt/CodeLexer（无 UI 依赖）
src/editor/    EditorWidget(QPlainTextEdit 子类)、FormatActions、MarkdownHighlighter、LineNumberArea
src/preview/   PreviewWidget（渲染管线/滚动同步）、HtmlPreviewView（WebView2 封装）
src/render/    MathRenderer、MermaidRenderer
src/search/    SearchDialog（查找/替换/跨文件搜索）
src/sidebar/   FileSidebar、OutlinePanel、SidebarHandle
src/sync/      SyncManager（syncthing.exe 子进程 + REST 配置）、LockClient（文件锁）
src/ui/        SettingsDialog、MoreDialog、TodoTxtFilterDialog
resources/     qss 三主题、图标、i18n；third_party/webview2/  WebView2 SDK
```

### Android（`Markor-Android/app/src/main/java/net/gsantner/`）
```
markor/sync/          SyncManager.java（引擎启停/REST 配置/仅前台）、SyncWebGuiActivity（应用内 Web GUI）、LockClient.java
markor/activity/      SettingsActivity（设置页+同步 pref 处理）、DocumentEditAndViewFragment（编辑/预览/文件锁）、MainActivity、MarkorBaseActivity（SyncManager 挂接）
markor/model/         AppSettings.java（设置门面）、Document.java
markor/format/        各格式（markdown/todotxt/…）高亮/动作/转换
markor/frontend/      对话框工厂、文本视图工具
res/xml/preferences_master.xml   ★设置界面结构（含同步 PreferenceCategory）
res/values/string-not_translatable.xml  ★pref key 定义
opoc/                 工具库（偏好后端 GsSharedPreferencesPropertyBackend 等）
```

### Server（`Markor-Server/`）
```
lockserver/main.py            文件锁 HTTP 服务（:8910，标准库实现）
markor-syncthing.service      NAS Syncthing systemd 单元
markor-lockserver.service     lockserver systemd 单元
enable-versioning.py          ★一次性脚本：开启回收站版本控制（与"无版本控制"需求冲突，见 §4.5）
docker-compose.yml、.env.example、README-server.md（锁协议文档 §五）
```

## 4. 需求 → 改动点映射（本轮需求，含可行性）

> 状态：以下为**分析结论**，等用户确认理解后再动手。

### 4.1 PC 复制粘贴偶发"粘贴出上一条复制内容"（bug）
- **可行**。编辑器本体 `EditorWidget`（QPlainTextEdit）无自定义剪贴板代码，复制/粘贴走 Qt 原生 → 可疑点：
  a) Qt `QClipboard` 在 Windows 的延迟渲染缓存（own→paste 竞态）；
  b) 从 WebView2 预览复制后回编辑器粘贴（跨进程写剪贴板，Qt 缓存未刷新）。
- 改动点：`src/editor/EditorWidget.cpp`（如需覆写 paste/copy）、`src/preview/HtmlPreviewView.cpp`（WebView2 剪贴板互通）；必要时 Win32 直读剪贴板兜底。
- **需用户补充复现路径**（在编辑器内复制粘贴？从预览复制？跨应用？）。

### 4.2 PC 打开文件时空"未命名"标签应消失 + 标签栏"+"新建
- **可行**。现状：`MainWindow::openFilePath()` 总是 `createPage()` 新建标签，`ensureUnnamedTabAtEnd()` 反而把未命名标签挪到最右**保留**。
- 改动点：`src/app/MainWindow.cpp` —
  - `openFilePath()`：若当前标签是未命名且无内容（非 dirty、空文本）→ 复用/替换该标签，不再追加；
  - 标签栏加"+"：`QTabWidget::setCornerWidget()` 或末尾"+"占位标签 → `newDocument()`；
  - `ensureUnnamedTabAtEnd()` 的策略相应调整（只在无未命名标签且需要时补）。

### 4.3 PC 同步配置调整 + Ctrl+F 焦点 bug
| 项 | 可行性 | 改动点 |
|---|---|---|
| 同步不需要 GUI 界面 | ✅ 现状已无 GUI 入口（REST 仅内部用 127.0.0.1:8385）| 确认即可；如需彻底可再收紧注释/文档 |
| 同步目录 = 文档库目录 | ✅ 已实现（applyConfig 每次取 `s.vaultDir()`）| 无需改 |
| 无文件版本控制 | ✅ 客户端本就未配置 versioning | 显式清空 `folder.versioning` 兜底；服务端见 §4.5 |
| 默认忽略权限 | ✅ 可实现 | `SyncManager::applyConfig()` folder 增 `ignorePerms: true` |
| 有块索引 | ✅ 引擎内建（分块哈希/差量传输恒开）| 无需配置；如有别的含义待确认 |
| "启用同步"下加"启用自动发现"开关 | ✅ 可实现 | `SettingsDialog.cpp`（同款 CheckBox）、`AppSettings.{h,cpp}` 新键、`SyncManager::applyConfig()` 把 `localAnnounceEnabled` 从写死 false 改为跟随开关（建议默认关） |
| Ctrl+F 打开的是预览区查找 | ✅ 可修 | `MainWindow::findInFile()` + `HtmlPreviewView.cpp`：WebView2 焦点吞 Ctrl+F（点击编辑器后原生 HWND 焦点未真正转移）。修复：Ctrl+F 提升为应用级快捷键并显式 `editor->setFocus()`，或禁用 WebView2 浏览器加速键 |

### 4.4 Android 同步设置调整
| 项 | 可行性 | 改动点 |
|---|---|---|
| 取消"无同步信息不能开 GUI"限制 | ✅ | `SettingsActivity`（`pref_key__sync_open_gui` 点击处的前置判断）/ `SyncWebGuiActivity` 的进入检查 |
| GUI 端口 | ⚠️ **硬约束**：Syncthing Web GUI 必须监听一个 TCP 端口，无法"无端口" | 保持 `127.0.0.1:8385`（仅本机回环、不可从外部访问）——即"对外无端口" |
| 同步移出"通用"，独立为设置入口，只留"同步开关 + 同步管理" | ✅ | `res/xml/preferences_master.xml`（PreferenceCategory 重组/独立屏）、`SettingsActivity` |
| ⚠️ 待确认：现有 5 个服务器信息输入框（设备名/服务器ID/同步地址/锁API/Key）移到哪里？ | —— | 建议：全部并入"同步管理"（Web GUI 里配置），或保留折叠在同步设置内，**需用户拍板** |

### 4.5 服务端是否要改（用户问）
- **建议小改 2 处，其余不动**：
  1. `enable-versioning.py` 与"无文件版本控制"直接冲突 → **删除或停用**（改名 `.disabled`）；
  2. `markor-syncthing.service` 的 `STGUIADDRESS=0.0.0.0:8385` 把管理界面暴露到了整个网络 → 建议收紧为 `127.0.0.1:8385`（NAS 本机管理），如需远程管理再用 SSH 隧道。
- lockserver（`lockserver/main.py`）协议不变，无需改。
- 若启用"自动发现"（§4.3）且服务器与客户端同局域网，服务器端 Syncthing 需同样开启本地发现——记入部署文档即可。

## 5. 红线（三端通用）
1. **服务器信息默认空、不写死**任何 IP/域名/密钥常量（127.0.0.1 除外）。
2. **星型拓扑不可破坏**：设备表只有服务器；introducer 不开；中继/全局发现默认关；引擎自升级关。
3. **改动后更新对应端的 AGENTS.md / docs/**（PC 还需同步 `CMakeLists.txt` 与 `markor-win.pro` 两处构建清单）。
4. **等用户通知才编译构建**；只改代码 + 自查。
5. 文档/UI 文案中文；设置只经 AppSettings/设置门面；主题色只经 ThemeManager（PC）。
