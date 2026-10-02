# Wayland 截图与 portal

## 最终方案（本机已生效）

| 需求 | 方案 |
|---|---|
| **截图 + 标注 + 贴图** | **mark-shot**（开源，自带热键，不依赖任何 portal 绕行技巧） |
| 窗口 / 全屏 | `PrtSc` / `Alt+PrtSc` / `Shift+PrtSc`（GNOME 内置，已绑好） |
| **贴图窗口置顶** | `Super+T`（出厂未绑，需先执行下面这条） |
| **贴图窗口移动** | **按住 `Super` 左键拖**（GNOME 全局规则，对任何窗口有效） |

```sh
gsettings set org.gnome.desktop.wm.keybindings toggle-above "['<Super>t']"
```

> **关键认知：贴图窗口「不能置顶 / 不能拖动」不是截图工具的缺陷，是 Wayland 的规定。**
> Wayland 禁止**应用擅自**声明置顶、擅自决定自己的屏幕坐标——这两条同时也是伪装系统 UI 的必要条件。
> 但它没禁止**用户主动指挥窗口管理器**做同样的事。所以正解是走合成器的 `toggle-above` 和
> `mouse-button-modifier`（默认 `<Super>`），而不是去找一个「能自己置顶的截图工具」——
> **那样的工具在 Wayland 上不存在。** 和截图那条是同一个原则的两次体现，见文末。

> **2026-08-07 重装系统后原样复现全部结论**，又把 flameshot 的路完整重走一遍并再次撞墙。
> 既然 mark-shot 一个工具覆盖截图/标注/贴图，本机已把 flameshot、ksnip、gnome-screenshot 全部卸载。

<details>
<summary>备选：纯 GNOME 方案（无标注、无贴图，但零第三方依赖）</summary>

旧机器上用的就是这套，绑 F11/F9：
```sh
sh -c 'd=~/Pictures/Screenshots; mkdir -p "$d"; f="$d/$(date +%Y%m%d-%H%M%S).png"; gnome-screenshot -a -f "$f" && wl-copy --type image/png < "$f"'
```
依赖：`gnome-screenshot`、`wl-clipboard`（`wl-copy` 的原因见 [[07-Linux剪贴板机制]]）
</details>

---

## 为什么 Flameshot 的快捷键在 GNOME 46 上不可能work

### 三层障碍，逐层剥开

**第一层：buffer 隔离**（见 [[03-Wayland与X11架构]]）
截图要读别的应用的 buffer，Wayland 下客户端读不到，必须走 `xdg-desktop-portal` 向合成器申请。

**第二层：交互覆盖层**
X11 下 Flameshot 靠 `override-redirect` 窗口 + `XGrabKeyboard` 铺一个实时全屏选区层。Wayland 删除了这些能力（它们同时也是钓鱼窗口和键盘记录器的必要条件）。

> **但这一层 Flameshot 12 绕过去了**：它先通过 portal 拿一张**静态全屏图**，再把图显示在自己的普通窗口里让你在图上拖框。选区发生在自己窗口内部，不需要特权。所以 `flameshot gui` 从终端跑是通的。

**第三层：catch-22 —— 这才是真正堵死的地方**

```
① 要截图 → portal 需要用户授权
② 要授权 → 必须弹一个系统授权对话框
③ 要弹对话框 → 调用方必须是当前【焦点应用】（GNOME 45+ 反钓鱼措施）
④ 但热键唤起的截图工具，天生不是焦点应用
```

日志原文：
```
Failed to associate portal window with parent window ''
Failed to show access dialog: AccessDenied:
    Only the focused app is allowed to show a system access dialog
```

### 两条修法及其证伪（重要）

| 修法 | 结果 | 排除了什么 |
|---|---|---|
| 往权限库预写 `org.flameshot.Flameshot: ['yes']` | ❌ 无效 | 排除「权限没配好」 |
| 改用 `flameshot gui --delay 500`（上游 desktop 文件自己的做法） | ❌ 无效 | 排除「时序竞态」 |
| 设 `XDG_CURRENT_DESKTOP=sway` 骗 flameshot 改走 portal 后端<br>（并同时覆盖 DBus service 文件和 `.desktop` 文件，让 daemon 也继承） | ❌ 无效 | 排除「选错截屏后端」——**后端选对了照样卡在焦点这一层** |
| 换 **ksnip**（另一个走 portal 的标注截图工具）绑热键 | ❌ **截出纯黑图** | 排除「换个 portal 客户端就行」——**catch-22 与具体应用无关，是 GNOME 的架构决定** |
| 让 daemon 常驻并带 `sway` 环境，热键只发 DBus 请求 | ❌ 无效 | 排除「daemon 环境没配对」——见下方「交互 / 非交互」那条分界线 |
| 用 `gdbus` 调 `SelectArea` 拿选区坐标，再喂给 `flameshot gui --region … --pin` | ❌ `SelectArea is not allowed` | 排除「自己拼一个选区器」——**GNOME 只把选区接口开给白名单，而白名单里的 `gnome-screenshot` 不吐坐标** |

### ★ 真正的分界线：交互 vs 非交互

2026-08-07 用四组对照实验切出来的：

| 调用 | 需要弹选区 UI | 结果 |
|---|---|---|
| `flameshot full`（客户端带 `sway` 环境） | 否 | ✅ |
| `flameshot gui --region 400x300+100+100` | 否 | ✅ |
| `flameshot gui`（客户端带 `sway` 环境） | **是** | ❌ |
| `flameshot gui`（客户端原生 GNOME 环境） | **是** | ❌ |

**决定成败的是「要不要弹交互界面」，不是环境变量、不是后端、不是 daemon。**
非交互截屏能通——但非交互给不了你选区，对实际使用毫无价值。
这条也解释了为什么「先手动跑通一次」会误导：手动测试常用 `full` 或 `--region`，走的根本不是热键那条路。

**关键推论**：空 app id `''` 本来就已经是 `yes`，portal 却依然要弹框 → 说明 GNOME 对**非交互式全屏截图强制每次确认**，存储的授权根本不参与决策。这是刻意设计：不允许任何应用静默截屏。

> **假设被证伪也是结论。** 这两条排除了两种最常见的可能，才让「架构性堵死」这个判断站得住。

### gnome-screenshot 为什么能work

它**不走 portal**，直接调 `org.gnome.Shell.Screenshot` D-Bus 接口（它是该接口的历史合法调用方，在 gnome-shell 的白名单里）。整个 catch-22 绕开了。

## `org.gnome.Shell.Screenshot` 接口

```
SelectArea(out i x, y, w, h);          ← 十字准星，一拉成型，返回矩形
ScreenshotArea(in i x, y, w, h, ...);
Screenshot / ScreenshotWindow / InteractiveScreenshot / PickColor / FlashArea
```

接口一直在，GNOME 42 只是把 `SelectArea` 从**默认交互**里拿掉了。但有**调用方白名单**：

```bash
gdbus call --session --dest org.gnome.Shell \
  --object-path /org/gnome/Shell/Screenshot \
  --method org.gnome.Shell.Screenshot.FlashArea 100 100 400 300
# → AccessDenied: FlashArea is not allowed   （普通 shell 不在白名单里）
```

## 踩坑记录

- **`grim` 在 GNOME 上不可用** —— 它依赖 `wlr-screencopy` 协议，那是 wlroots 系（Sway/Hyprland）的东西，mutter 不实现且明确不会实现。网上大量 Wayland 截图教程默认你在 wlroots 系。实测报错原文：`compositor doesn't support wlr-screencopy-unstable-v1`。
- **「截出纯黑图」= 没有截屏权限，不要当成显卡/驱动问题去查。** flameshot 至少会明确报 `Unable to capture screen`；ksnip 不报错，直接给你一张纯黑 PNG。Wayland 下这两种表现是同一个病。
- **修 daemon 类程序的启动环境要找对文件。** flameshot 的 daemon 有两个可能的启动源：`/usr/share/dbus-1/services/*.service`（DBus 激活）和 `/usr/share/applications/*.desktop`（gnome-shell 启动）。日志里 `Application launched by gnome-shell` 说明走的是后者，改前者无效。判断依据看 `journalctl --user` 里那行 `Started app-gnome-*.scope - Application launched by X`。两者都能用 `~/.local/share/` 下的同名文件覆盖，不需要 sudo。
- **GNOME 内置区域截图的交互**：先给一个默认矩形让你调，不是「按下即起点」。这是 GNOME 42 重做 UI 后的既定行为，改不了。
- `gnome-screenshot` 上游已废弃，Ubuntu 未来版本可能移除。届时的替代路是 **GNOME 扩展**（代码跑在 gnome-shell 进程内，天然有全部特权，不受焦点限制，可实现任意交互）。

## 一个反复出现的原则

> **Wayland 下要做特权操作，正路是把代码放进有特权的那一层，而不是给外面的程序开后门。**

「起一个隐藏的特权进程」在 Wayland 上不成立 —— **协议里没有「root 可以读所有 buffer」这种后门**，权限按应用授予，不按 uid。这个思路在 GNOME 上的正确形态就是**扩展**；在嵌入式硬件访问上的正确形态是 **udev 规则 + 用户组**，而不是 sudo。

### 这个原则的第二种形态：让用户下令，而不是让应用自作主张

截图、置顶、定位窗口，被禁的都是「**应用自己**要求做」。同样的操作由**用户**通过合成器发起就是合法的：

| 应用想干的事 | Wayland 禁止 | 用户经合成器做同一件事 |
|---|---|---|
| 读别的窗口画面 | ❌ | `PrtSc`（gnome-shell 自己截） |
| 声明自己永远置顶 | ❌ | `Super+T`（`toggle-above`）/ `Alt+空格` 窗口菜单 |
| 决定自己的屏幕坐标 | ❌ | 按住 `Super` 拖 / `Alt+F7` |

**遇到「这个功能在 Wayland 上没了」，先问一句：是不是只是换成由我来发起？** 大多数情况是。
真正被彻底删掉的能力很少，被删掉的是**应用擅自发起**的那条路径。

## 排查用到的命令

```bash
journalctl --user -f -u xdg-desktop-portal-gnome.service -u xdg-desktop-portal.service
gdbus introspect --session --dest org.gnome.Shell --object-path /org/gnome/Shell/Screenshot
gdbus call --session --dest org.freedesktop.impl.portal.PermissionStore \
  --object-path /org/freedesktop/impl/portal/PermissionStore \
  --method org.freedesktop.impl.portal.PermissionStore.Lookup screenshot screenshot
```

方法论见 [[08-系统自省方法]]。
