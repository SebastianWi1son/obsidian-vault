# `.deb` 与包管理

## 核心认知转换

`.deb` **不是** `setup.exe`。它不是程序，双击或 `chmod +x` 都没用。

| | Windows `setup.exe` | Linux `.deb` |
|---|---|---|
| 本质 | 一个**可执行程序**（它自己就是安装器） | 一个**压缩包 + 一张清单** |
| 谁安装 | 它自己 | 系统的 `dpkg` / `apt` |
| 依赖 | 自带一份，或弹窗让你去装运行库 | 清单里声明依赖名，**apt 自动解析下载** |
| 装到哪 | 它说了算（Program Files / 注册表） | 清单决定，且**系统记录每一个文件** |
| 卸载 | 靠它留下的 uninstaller，常卸不干净 | `apt remove`，按记录逐个删 |

## `.deb` 拆开是三部分

- **control** — 元信息：包名、版本、架构、**依赖谁**
- **data** — 要铺进系统的文件树（`./usr/bin/xxx`、`./opt/xxx`…）
- **maintainer scripts** — `postinst`（装完跑）、`prerm`（卸载前跑）等钩子

## 装之前先看包（只读，不会安装）

```bash
dpkg-deb -I package.deb      # Info：看 control 元信息，最重要
dpkg-deb -c package.deb      # Contents：看它要往系统里铺哪些文件
```

装完之后反查：

```bash
dpkg -L 包名                  # 这个包装了哪些文件
dpkg -S /某个/文件路径         # 这个文件属于哪个包
```

> Windows 上你永远不知道 setup.exe 动了什么，这里一目了然。养成装陌生 deb 前先 `-I` 的习惯。

## 正确的安装命令

```bash
sudo apt install ./package.deb
```

**两个细节：**

1. **必须写 `./`** — 不加路径的话，apt 会当成「从仓库找一个叫这名字的包」。凡是含 `/` 的参数才被识别为本地文件。

2. **不要用 `sudo dpkg -i`** — `dpkg` 是底层安装器，缺依赖就直接失败退出，留下半装状态，还要 `sudo apt -f install` 补救。

## 分层：dpkg vs apt

```
apt    ← 上层：依赖解析 + 从仓库下载 + 调度
 └─ dpkg ← 底层：装/删单个包，记录文件清单
```

这个「底层工具 + 上层调度」的模式在 Linux 里到处都是（后面 `gcc` 与 `make`、`dpkg` 与 `apt`、`ld` 与 `gcc` 全是同一结构）。

## 三种软件来源模型

| 来源 | 装到哪 | 更新方式 |
|---|---|---|
| **apt**（系统仓库） | `/usr`，共享依赖 | `apt upgrade` 统一升 |
| **snap**（沙盒） | `/snap/<name>/<rev>`，squashfs 只读挂载 | snapd 自动 |
| **手动 .deb / tarball** | 自己指定（常见 `/opt`） | 自己管 |

## 实战案例：搜狗输入法的 deb

`dpkg-deb -I sogoupinyin_4.2.1.145_amd64.deb` 读出：

```
Depends: fcitx (>= 1:4.2.8), fcitx-frontend-gtk2, fcitx-frontend-gtk3,
         fcitx-frontend-qt5, fcitx-module-x11, ...
```

三个结论，全部来自这一行元信息：
1. 依赖 **fcitx 4**，而系统装的是 fcitx 5 —— 两套独立框架会冲突（见 [[05-fcitx5输入法架构]]）
2. 只有 `fcitx-module-x11`，**没有 Wayland 支持** —— 见 [[03-Wayland与X11架构]]
3. 前端只到 qt5，没有 qt6

**这就是「装之前先看包」的价值**：三分钟看清一个包会怎么改造你的系统。

## 一个更重要的习惯

> Windows 上「没有某功能」= 去官网下载安装包。
> Linux 上「没有」很可能只是「**装了但没启用**」或「仓库里一条命令的事」。

遇事先查已装列表、先 `apt search`，再开浏览器。

```bash
dpkg -l | grep 关键字        # 已装的
apt search 关键字            # 仓库里有的
apt-mark showmanual          # 我主动装的（区别于被拖进来的依赖）
```
