# fcitx5 输入法架构

## 三层结构

```
   应用程序 (Chrome / CLion / GNOME 文本框)
        ↑  ← 第三层：前端 frontend
        │     gtk3 / gtk4 / qt5 / qt6 各一个模块
        │     Wayland 下还多一条：text-input-v3 协议直连合成器
        │
   fcitx5 守护进程
        ↑  ← 第二层：框架 framework
        │     由 ~/.xinputrc 里的 run_im fcitx5 拉起（im-config 写的）
        │
   输入法引擎 (addon)   ← 第一层：拼音怎么变成字
        pinyin / rime / shuangpin / cangjie / wbpy ...
        定义文件在 /usr/share/fcitx5/inputmethod/*.conf
```

排查时先定位在哪一层。**「哪个框架开机启动」是第二层的事（`im-config`），「用哪个引擎」是第一层的事（`fcitx5-config-qt`）**，两者常被混淆。

## 坑 1：输入法引擎 ≠ 键盘布局

配置界面的可用列表里混着两类东西，名字里都可能带 `pinyin`：

| | 键盘布局 | 输入法引擎 |
|---|---|---|
| 例子 | `keyboard-cn-altgr-pinyin` | `pinyin` |
| 显示名 | **Keyboard - Chinese (AltGr pinyin)** | **Pinyin / 拼音**（带「拼」字图标） |
| 本质 | 一个 XKB 键盘布局 | 真正的输入法 |
| 作用 | 按 AltGr 打出 ā á ǎ à | 拼音 → 候选词 → 汉字 |
| 定义在 | `/usr/share/X11/xkb/rules/evdev.xml` | `/usr/share/fcitx5/inputmethod/pinyin.conf` |

> **一眼分辨**：显示名以 **`Keyboard - `** 开头的都是键盘布局，不是输入法。

选错了会表现为「切换正常但打不出汉字、没有候选框」。

## 坑 2：profile 不能直接编辑

`~/.config/fcitx5/profile` 由正在运行的 fcitx5 托管，直接改会被覆盖。详见 [[02-配置文件的归属权]]。

正确的配置文件长这样：
```ini
[Groups/0]
Default Layout=us
DefaultIM=pinyin

[Groups/0/Items/0]
Name=keyboard-us        ← 第一位 = 默认状态

[Groups/0/Items/1]
Name=pinyin             ← 第二位，切换键在一二位之间跳
```

**顺序有讲究**：英文放第一位 → 新窗口默认英文状态，写代码更舒服。

## Pinyin vs Rime：不是同一类东西

| | **Pinyin**（fcitx5-pinyin） | **Rime**（中州韵） |
|---|---|---|
| 定位 | 一个**成品输入法** | 一个**输入法引擎框架** |
| 类比 | 装好就能开的整车 | 发动机+变速箱，车壳自己配 |
| 底层 | libime，自带 n-gram 统计语言模型 | librime，需选/写「方案 schema」 |
| 默认行为 | 简体、开箱即用 | 取决于方案；默认 `luna_pinyin` 是**繁体** |
| 配置方式 | GUI 点几下 | 手写 YAML patch |

**Rime 默认出繁体不是 bug**，是默认方案面向港台用户。

**Rime 的真正价值**：同一引擎能跑拼音/双拼/五笔/仓颉/粤拼，且**三平台配置文件通用**（Windows 小狼毫 / macOS 鼠须管 / Linux fcitx5-rime），全本地不联网。代价是要写 YAML。

**要一个顺手的简体拼音 → 选 Pinyin。**

## 云拼音

**机制**：本地出候选的同时，把**拼音串发到百度/Google 的 API**，云端返回一个候选插进列表。

配置 `~/.config/fcitx5/conf/cloudpinyin.conf`：
```ini
MinimumPinyinLength=4     # 拼音串≥4字符才触发
Backend=Baidu             # GoogleCN 在国内连不上，要选 Baidu
```

- **收益**：长句、新词、人名地名、流行语。本地词库静态，云端是活的。搜狗当年好用很大部分靠这个。
- **代价**：每次输入都有网络请求，**你打的拼音会离开本机**；要联网；有延迟。
- **建议**：日常开着，但记住敲密码/敏感内容时它是开的。在意的话把 `MinimumPinyinLength` 调到 6–8。

## 可以只要搜狗的词库，不要搜狗的框架

系统自带转换工具：
```bash
ls /usr/bin/libime_*
# libime_pinyindict 能把词库转成 fcitx5 的二进制格式
```
fcitx5 支持导入**搜狗细胞词库（.scel）**，入口在 `fcitx5-config-qt` → Addons → Pinyin 的词典管理。

> 这是 Linux 很典型的思路：**组件可以拆开挑**，不像 Windows 只能整包吞下。

## 为什么搜狗输入法在 Wayland 下有问题

它的 deb 依赖里写着 `fcitx-module-x11`，**只有 X11 前端**（见 [[01-deb与包管理]] 的实战案例）。X11 下输入法可以随便查光标位置；Wayland 下必须靠 **text-input-v3** 协议由应用主动上报（见 [[03-Wayland与X11架构]]）。

fcitx5 不用设 `GTK_IM_MODULE`/`QT_IM_MODULE` 任何环境变量就能工作，正是因为它走的是原生 Wayland 协议。**如果某个输入法教程让你去 `/etc/environment` 加一堆 IM 环境变量，那是 X11 时代的做法。**

## 常用命令

```bash
fcitx5-config-qt                        # 配置 GUI
fcitx5 -r -d                            # 重启（-r 替换现有实例，-d 后台）
fcitx5-remote -e                        # 退出 daemon
cat ~/.config/fcitx5/profile            # 当前启用了什么
ls /usr/share/fcitx5/inputmethod/       # 系统里有哪些引擎可用
im-config                               # 改「开机启动哪个输入法框架」
```
