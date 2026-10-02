# Linux 剪贴板机制

## 核心：剪贴板不是一块内存，是一次「协商传输」

```
Windows:
   复制 → 数据拷进系统全局剪贴板内存
        → 源程序关掉也没事                          ✅

Linux (X11 和 Wayland 都一样):
   复制 → 源应用向显示服务器宣告「我持有这些格式的数据」
        → 成为 selection owner
        → 数据【仍在源应用内存里】，没有拷到任何地方
   粘贴 → 目标应用向源应用【实时索取】字节
        → 源应用已退出？→ 没人应答 → 空             ❌
```

**一句话**：Linux 剪贴板存的是「谁持有数据」，不是数据本身。

## 典型症状

`gnome-screenshot -a -c` 截图成功，但粘贴出来是空的。

原因：它截完图 → 宣告持有 → **立刻退出**。等你粘贴时持有者已经没了。

对照：GNOME 自带 `PrtSc` 复制到剪贴板一直好用，因为持有者是 **gnome-shell，它永远不退出**。

## 解法：让一个进程留下来守着

`wl-copy`（`wl-clipboard` 包）就是为这件事设计的 —— 读完数据后 **fork 到后台常驻**，专门应答粘贴请求。

```bash
sudo apt install wl-clipboard
```

```bash
wl-copy --type image/png < shot.png      # 复制图片
wl-copy < file.txt                       # 复制文本
wl-paste                                 # 粘贴出来
wl-paste --list-types                    # ★ 看剪贴板里到底有哪些格式
```

`wl-paste --list-types` 是排查剪贴板问题的第一条命令 —— 它直接告诉你持有者宣告了什么格式，不用靠反复粘贴去猜。

> X11 下的对应工具是 `xclip` / `xsel`，同样会 fork 常驻，原理一致。

## 「一个格式，多种表示」

selection owner 宣告的是**一组 MIME 类型**，粘贴方挑一个它认识的来要。所以同一次复制，粘到不同地方结果可能不同：

- 从浏览器复制 → 同时宣告 `text/html` 和 `text/plain`
- 粘到富文本编辑器 → 拿 `text/html`，带格式
- 粘到终端 → 拿 `text/plain`，纯文本

这解释了很多「为什么粘过去格式变了」的困惑。

## 三个 selection（X11 遗产，Wayland 保留了前两个）

| 名字 | 怎么复制 | 怎么粘贴 |
|---|---|---|
| **CLIPBOARD** | Ctrl+C / 菜单复制 | Ctrl+V |
| **PRIMARY** | **选中文本即自动进入** | **鼠标中键** |
| SECONDARY | 基本没人用 | — |

**PRIMARY 是 Linux 独有的效率点**：鼠标划一段文字，到别处按中键就粘上了，完全不经过 Ctrl+C/V，而且不会覆盖你 CLIPBOARD 里的东西。用顺了很难回去。

```bash
wl-paste --primary        # 看 PRIMARY 里是什么
```

## 想要 Windows 式体验（关掉源程序也不丢）

需要一个**剪贴板管理器**常驻接管所有复制内容：
- GPaste
- GNOME 扩展 Clipboard Indicator

它顺带给你剪贴板历史。代价是一个常驻进程，且**所有复制过的内容都会被它记录**（包括密码管理器复制的密码 —— 大多数有排除规则，值得配一下）。

## 相关

- 实战案例见 [[06-Wayland截图与portal]]
- 「谁持有资源」这个模式和 [[02-配置文件的归属权]] 是同一种思路
