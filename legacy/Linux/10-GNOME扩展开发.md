# GNOME 扩展开发:第一个扩展 Pin On Top

2026-08-07,为解决「贴图窗口不能一键置顶、没有状态提示」写的第一个 GNOME Shell 扩展。
本篇是完整的过程复盘,目标是**下次能独立写出第二个**。

## 0. 成品是什么

顶栏一个图钉图标:
- **点击** → 当前焦点窗口切换置顶
- **全亮** = 焦点窗口已置顶;**半透明** = 未置顶;**更暗** = 焦点不在任何窗口
- 与 `Alt+空格` 菜单、`Super+T` 双向同步

代码:`~/.local/share/gnome-shell/extensions/pin-on-top@wilson.local/`,共 2 个文件,60 行。

## 1. 为什么答案只能是扩展

回顾 [[06-Wayland截图与portal]] 的原则:**特权操作要把代码放进有特权的那一层**。

窗口层级(谁在上面)是合成器的专属权力。GNOME 的合成器(mutter)、窗口管理器、
桌面 UI 跑在**同一个进程** `gnome-shell` 里。官方唯一受支持的进程内注入点就是扩展:

```
┌─ gnome-shell 进程 ──────────────────────────┐
│  mutter(合成器+窗口管理) ← Meta.Window API │
│  Shell UI(顶栏、概览)   ← St / Main       │
│  你的扩展 JS  ←—— 和它们同进程、同权限      │
└─────────────────────────────────────────────┘
```

所以扩展调 `make_above()` 和 `Alt+空格` 菜单走的是**同一条代码路径**——
状态天然一致,不存在同步问题。

一个成立的关键细节:**点顶栏不转移键盘焦点**(顶栏是 shell 自己的 UI,不是应用窗口),
所以点按钮时 `global.display.focus_window` 仍是你正在用的窗口——"点按钮置顶当前窗口"才成立。

## 2. 扩展的解剖结构

```
~/.local/share/gnome-shell/extensions/<uuid>/
├── metadata.json    ← 身份证:uuid、支持的 shell 版本
└── extension.js     ← 全部逻辑
```

### metadata.json 逐字段

```json
{
  "uuid": "pin-on-top@wilson.local",      ← 必须和目录名完全一致
  "name": "Pin On Top",
  "description": "…",
  "shell-version": ["46"],                ← 不含当前版本号 → 拒绝加载(升级保护)
  "url": ""
}
```

### extension.js 的骨架(GNOME 45+ 硬约束:必须 ESM)

```js
import {Extension} from 'resource:///org/gnome/shell/extensions/extension.js';

export default class MyExtension extends Extension {
    enable()  { /* 建 UI、连信号 */ }
    disable() { /* 严格镜像:断信号、毁 UI、置 null */ }
}
```

- `gi://St` = Shell 的 UI 控件库(图标、按钮);`gi://` 前缀是 GObject 库
- `resource:///org/gnome/shell/ui/main.js` = shell 内部模块;`Main.panel` 就是顶栏
- GNOME 44 及以前的教程用 `imports.*` 旧写法,**45 起全部作废**,搜资料时注意年份

## 3. 本次用到的核心 API(只有 4 个)

| API | 作用 |
|---|---|
| `global.display.focus_window` | 当前焦点窗口(`Meta.Window`,可能为 null) |
| `win.above` / `win.make_above()` / `win.unmake_above()` | 读/设置顶状态 |
| `global.display.connect('notify::focus-window', cb)` | 焦点变化信号 |
| `win.connect('notify::above', cb)` | 该窗口置顶状态变化信号 |

`notify::属性名` 是 GObject 的通用机制:任何属性变化都可以这样监听。

## 4. 三个值得记住的设计决策

**① 单一数据源**:点击处理里只调 `make_above()`,**不直接改图标**。
图标刷新全部交给 `notify::above` 信号。这样无论置顶从哪个入口改(我的按钮、
`Alt+空格`、`Super+T`),图标永远正确。自己维护一份"我以为的状态"迟早失步。

**② 信号跟着焦点走**:`notify::above` 挂在具体窗口上,焦点切换时必须
**先断旧窗口的,再连新窗口的**。忘记断 = 幽灵回调 + 内存泄漏。

**③ enable/disable 严格对称**:`disable()` 是 `enable()` 的镜像——
每个 `connect()` 记下返回的 handler id,disable 时逐个 `disconnect()`,控件 `destroy()`。
这是官方审核红线。写坏的后果:禁用扩展后图标还在、回调还在跑。

## 5. 开发流程(可复制的套路)

```
写文件 → 嵌套会话冒烟测试 → 注销重登 → enable → 实测
```

**嵌套会话**是关键一步——在窗口里起一个迷你 GNOME,**不动真实会话**就能验证代码能否加载:

```sh
dbus-run-session -- gnome-shell --nested --wayland
# 在嵌套会话的环境里 enable,然后看它的 stderr 有无 JS ERROR
```

为什么必须注销重登:新扩展只在 shell **启动时**扫描;Wayland 下 shell 就是合成器,
不能热重启(X11 时代的 `Alt+F2 r` 不存在了)。

调试看日志:

```sh
journalctl --user -b | grep -iE '<uuid>|JS ERROR'
```

**风险认知**:回调里的异常会被 shell 捕获并自动禁用扩展,不会崩会话;
但写得烂(死循环、每帧跑重活)会拖慢整个桌面——扩展和合成器共享一个主线程。

## 6. 卸载 = 删目录

```sh
gnome-extensions disable pin-on-top@wilson.local
rm -rf ~/.local/share/gnome-shell/extensions/pin-on-top@wilson.local
```

不装包、不碰系统目录。`gnome-extensions list / info / enable / disable` 是全套管理命令。

## 7. 下一个候选项目(待办)

**用扩展终结 flameshot 的 catch-22**:扩展注册自己的 DBus 接口,在 shell 进程内
弹选区 UI 并返回坐标(不受 `SelectArea is not allowed` 限制),热键脚本拿坐标喂给
`flameshot gui --region WxH+X+Y --pin`(非交互路径,已验证能通)。
约 100–150 行,新知识点:DBus 服务端、复用 shell 内部 UI。
目前 mark-shot 已覆盖需求,此项目纯练手,不急。
