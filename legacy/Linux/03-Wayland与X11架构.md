# Wayland 与 X11 架构

## 首先纠正一个误解

> **Wayland 不是一个程序，它是一套「协议」** —— 一份规定「应用和显示系统之间该发什么消息」的文档。
> 真正跑着的程序叫**合成器（compositor）**。GNOME 的合成器是 **mutter**。

在 Wayland 下，合成器 = 显示服务器 + 窗口管理器 + 合成器，三合一。

## 核心差别：谁在画？

```
X11 时代：
   应用 ──「帮我在(100,50)画个字 好」──> X Server ──> 屏幕
                                          ↑
                                     X Server 负责画

Wayland 时代：
   应用 ──自己画好一整块 buffer──> 合成器 mutter ──> 屏幕
     ↑                              ↑
   应用自己负责画                合成器只负责「拼贴」
```

**这一条推导出后面所有现象。** Wayland 下画字的是应用自己，所以「画得清不清楚」变成每个应用各自的事，系统管不了。

## 设计哲学

> **X11 默认「大家都能看见彼此」，Wayland 默认「谁也看不见谁，需要什么就明确申请」。**

X11 时代任何应用都能：
- 读取整个屏幕内容（包括别人的窗口）
- 用 `override-redirect` 窗口绕过窗口管理器，自定绝对坐标和层级
- `XGrabKeyboard` / `XGrabPointer` 全局抢占键鼠

这三条既是很多工具好用的原因，**也正好是钓鱼窗口和键盘记录器的全部必要条件**。Wayland 把它们全删了。

## 这解释了哪些「怪现象」

### 1. 截图工具失效

截图 = 要读**别的应用**的 buffer。Wayland 里 buffer 互相隔离，客户端读不到。

**正解**：走 `xdg-desktop-portal` 向合成器申请，由合成器把合成结果交出来。

**更关键的是**：像 Flameshot 这类工具还需要「铺一个全屏交互层选区 + 抢占键鼠」，这**不是权限问题，是协议里根本没有这个能力**。起一个 root 进程也没用 —— **Wayland 协议里没有「root 可以读所有 buffer」的后门**，权限是 portal 按**应用**授予的，不按 uid。

**成熟方案 = 职责分离**：
```
截图（需要特权）─────> 合成器 / portal 负责
                          │  传出一张普通 PNG
标注编辑（不需特权）───> 普通应用负责
```
GNOME 42+ 把截图 UI 做进了 gnome-shell —— 这不是妥协，是架构上唯一正确的位置：合成器天然拥有所有 buffer、天然在所有窗口之上、天然先收到键鼠事件。

### 2. 输入法要知道光标在哪

候选框得跟着光标走。X11 下随便查；Wayland 下必须靠 **text-input-v3** 协议，由应用主动上报位置。

这就是为什么 fcitx5 不用设任何环境变量就能工作，而只有 X11 前端的搜狗（`fcitx-module-x11`）在 Wayland 下会出问题。详见 [[05-fcitx5输入法架构]]。

### 3. 分数缩放下有的软件字糊

见 [[04-字体渲染与显示缩放]]。根因同样是「应用自己负责画」。

## XWayland

一个**跑在 Wayland 上的 X Server** 兼容层，让老 X11 应用继续能用。代价：它拿不到分数缩放的好处，缩放≠100% 时必糊。

**查谁在 XWayland 上：**
```bash
xlsclients -l
```

## 生态分裂：wlroots 系 vs GNOME 系

搜 Wayland 方案时**先确认对方是哪一系**，否则会浪费大量时间：

| | wlroots 系（Sway / Hyprland） | GNOME 系（mutter） |
|---|---|---|
| 截图协议 | `wlr-screencopy`（私有扩展） | 只认 `xdg-desktop-portal` |
| 典型工具 | `grim` + `slurp` | portal / gnome-shell 内置 |

**`grim` 在 GNOME 上不能用**，mutter 不实现 `wlr-screencopy` 且明确表示不会实现。网上大量 Wayland 截图教程默认你在 wlroots 系。

## 常用诊断

```bash
echo $XDG_SESSION_TYPE                      # wayland / x11
xlsclients -l                               # 谁在 XWayland 上
systemctl --user list-units | grep portal   # portal 服务状态
journalctl --user -f -u xdg-desktop-portal-gnome.service   # 实时看 portal 日志
```
