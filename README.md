# Debian 系统 fcitx5 + Rime 输入法配置指南

> 在 Debian GNU/Linux (12/13) + GNOME Wayland 环境下配置 fcitx5 输入法框架 + Rime 雾凇拼音方案的完整记录。
> 涵盖环境变量、自启动、候选词设置、深色主题、ibus 清理等，已在 Debian 12 和 Debian 13 上验证可用。
>
> 其他 Linux 发行版（Ubuntu、Fedora、Arch Linux、openSUSE 等）的配置流程类似，
> 主要区别在于包管理器（`apt` / `dnf` / `pacman` / `zypper`）和包名。
> 环境变量、fcitx5 配置文件、Rime 方案配置等用户级配置完全通用。

---

## 目录

- [1. 适用环境](#1-适用环境)
- [2. ibus 清理](#2-ibus-清理)
- [3. 安装 fcitx5](#3-安装-fcitx5)
- [4. 配置文件](#4-配置文件)
  - [4.1 指定输入法框架](#41-xinputrc--指定输入法框架)
  - [4.2 会话环境变量](#42-configenvironmentdfcitx5conf--会话环境变量)
  - [4.3 自启动](#43-configautostartorgfcitxfcrct5desktop--自启动)
  - [4.4 输入法组配置](#44-configfcitx5profile--把-rime-加入输入法组)
  - [4.5 GTK 注入](#45-gtk-注入gsettings)
  - [4.6 候选窗口深色主题](#46-候选窗口深色主题nord-dark)
- [5. 雾凇拼音（rime-ice）](#5-雾凇拼音rime-ice)
  - [5.1 候选词个数](#51-候选词个数5--9)
  - [5.2 简繁输出](#52-简繁输出)
- [6. 生效方式](#6-生效方式注销重登必须)
- [7. 日常使用与维护](#7-日常使用--维护)
- [8. 回滚](#8-回滚)
- [9. 已知限制](#9-已知限制)

---

## 1. 适用环境

| 项 | 值 | 判定方法 |
|---|---|---|
| 发行版 | Debian 12 (bookworm) / Debian 13 (trixie) | `cat /etc/os-release` |
| 桌面 | GNOME | `dpkg -s gnome-shell` |
| 会话类型 | Wayland（也兼容 X11） | `loginctl show-session <id> -p Type` |

**GNOME Wayland 关键限制**：GNOME Wayland 不实现第三方输入法协议
（`zwp_input_method_v2`），fcitx5 无法像在 KDE / Sway 下那样直接接管输入，
只能通过应用的 GTK / Qt IM module 注入。因此环境变量是**必需**的，且 GNOME
自家的 GTK4 应用（文本编辑器、Nautilus）可能仍然打不出中文 —— 这是 GNOME
的限制，不是配置错误。

> 判定会话类型的正确方法是用 `loginctl` + 进程列表，不要只看 `$XDG_SESSION_TYPE`。
> X11 窗口可能是 Xwayland 兼容层，不代表会话是 X11。

---

## 2. ibus 清理

Debian GNOME 默认预装 ibus，会抢占输入法协议导致 fcitx5 冲突。建议先清理：

```bash
sudo apt purge -y ibus ibus-libpinyin ibus-gtk ibus-gtk3 ibus-gtk4 \
                    ibus-data python3-ibus-1.0
```

> **注意**：
> - **保留** `gir1.2-ibus-1.0` + `libibus-1.0-5`：这是 `gnome-shell` 的**硬依赖**
> - `apt-get -s autoremove` 确认不涉及 gnome / gdm3 / nautilus 后再执行 autoremove

> 残留的旧 fcitx4 栈（`fcitx`、`fcitx-bin`、`fcitx-modules` 等）在 fcitx5 下已无用，
> 属 auto-installed 孤儿。清理时注意 `apt autoremove` 不要误删其他包的依赖。

如需恢复 ibus：

```bash
sudo apt install ibus ibus-libpinyin ibus-data ibus-gtk ibus-gtk3 ibus-gtk4 \
                 python3-ibus-1.0
```

---

## 3. 安装 fcitx5

### Debian / Ubuntu

```bash
sudo apt install fcitx5 fcitx5-rime fcitx5-frontend-gtk3 fcitx5-frontend-gtk4 \
                fcitx5-frontend-qt5 fcitx5-config-qt
sudo apt install git    # 用于 clone 词库
```

### Fedora

```bash
sudo dnf install fcitx5 fcitx5-rime fcitx5-gtk fcitx5-qt fcitx5-configtool git
```

### Arch Linux

```bash
sudo pacman -S fcitx5 fcitx5-rime fcitx5-gtk fcitx5-qt fcitx5-configtool git
```

### openSUSE

```bash
sudo zypper install fcitx5 fcitx5-rime fcitx5-gtk3 fcitx5-qt5 git
```

> **说明**：
> - `fcitx5-rime` 会拉入 `librime` + `rime-data-*` 等依赖
> - `fcitx5-config-qt` / `fcitx5-configtool` 提供图形化配置工具（可选）
> - 各 frontend 包确保 GTK3/GTK4/Qt5 应用都能走 fcitx5
> - 安装后**不要**立即启动 fcitx5，先完成下面的配置文件设置

---

## 4. 配置文件

全部为用户级配置，不需要改动 `/etc`。

### 4.1 `~/.xinputrc` — 指定输入法框架

```bash
im-config -n fcitx5
```

生成的文件内容：

```
run_im fcitx5
```

### 4.2 `~/.config/environment.d/fcitx5.conf` — 会话环境变量

```bash
mkdir -p ~/.config/environment.d
cat > ~/.config/environment.d/fcitx5.conf << 'EOF'
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
EOF
```

> **说明**：Wayland 会话由 systemd user manager 读取 `environment.d/`，
> `/etc/profile.d` 那套对图形会话无效，所以必须放这里。

### 4.3 `~/.config/autostart/org.fcitx.Fcitx5.desktop` — 自启动

```bash
cp /usr/share/applications/org.fcitx.Fcitx5.desktop ~/.config/autostart/
```

> **注意**：Debian 的 `fcitx5` 包**不提供** `/etc/xdg/autostart/` 条目。
> 不手工补这个文件，fcitx5 不会自启动。
> X11 下 im-config 会通过 `/etc/X11/Xsession.d/70im-config_launch` 启动它，
> Wayland 下没有这条路径。

### 4.4 `~/.config/fcitx5/profile` — 把 Rime 加入输入法组

fcitx5 **不会**自动把新装的引擎加进默认组，需要手工追加：

```ini
[Groups/0]
# Group Name
Name=Default
# Layout
Default Layout=us
# Default Input Method
DefaultIM=rime

[Groups/0/Items/0]
# Name
Name=keyboard-us
# Layout
Layout=

[Groups/0/Items/1]
# Name
Name=rime
# Layout
Layout=

[GroupOrder]
0=Default
```

也可以通过 `fcitx5-configtool` 图形界面添加。

### 4.5 GTK 注入（gsettings）

```bash
gsettings set org.gnome.desktop.interface gtk-im-module 'fcitx'
```

### 4.6 候选窗口深色主题（Nord-Dark）

fcitx5 的候选窗口默认白底，可通过 classicui 主题改为深色。

**安装主题**：

```bash
mkdir -p ~/.local/share/fcitx5/themes/Nord-Dark
git clone --depth 1 https://github.com/tonyfettes/fcitx5-nord.git /tmp/fcitx5-nord
cp /tmp/fcitx5-nord/Nord-Dark/* ~/.local/share/fcitx5/themes/Nord-Dark/
rm -rf /tmp/fcitx5-nord
```

**启用主题**：

```bash
mkdir -p ~/.config/fcitx5/conf
cat > ~/.config/fcitx5/conf/classicui.conf << 'EOF'
Theme=Nord-Dark
Vertical Candidate List=False
PerScreenDPI=True
Font="Sans 13"
EOF
```

> **说明**：
> - `Theme=Nord-Dark` 中的名称必须与 `~/.local/share/fcitx5/themes/` 下的目录名一致
> - `Font="Sans 13"` 控制候选窗口字体大小，可按需调整
> - 修改后重启 fcitx5 即生效（不需要重新登录）
> - 其他深色主题可选：[Fcitx5-Material-Color](https://github.com/hosxy/Fcitx5-Material-Color)
>   （`apt install fcitx5-material-color`，手动 `ln -sf ./theme-black.conf theme.conf` 切换配色）

**Nord-Dark 配色参考**：

| 元素 | 颜色 | 色值 |
|---|---|---|
| 候选窗口背景 | 深灰蓝 | `#434c5e` |
| 普通候选字 | 浅蓝灰 | `#81a1c1` |
| 高亮候选字 | 亮蓝 | `#88c0d0` |
| 高亮背景 | 稍深灰蓝 | `#4c566a` |

---

## 5. 雾凇拼音（rime-ice）

[雾凇拼音](https://github.com/iDvel/rime-ice) 是一套开箱即用的 Rime 配置方案，
默认全拼，词库丰富，支持多种双拼方案。

```bash
# 备份原配置（如果有的话）
mv ~/.local/share/fcitx5/rime ~/.local/share/fcitx5/rime.bak

# clone 雾凇拼音
git clone --depth 1 https://github.com/iDvel/rime-ice.git ~/.local/share/fcitx5/rime

# 重启 fcitx5 触发部署
fcitx5-remote -e && setsid fcitx5 -d >/dev/null 2>&1 < /dev/null &
```

部署产物：`build/rime_ice.{prism,reverse,table}.bin` + `rime_ice.schema.yaml`

默认方案：雾凇拼音全拼（**默认输出简体**），附带各种双拼方案
（自然码 / ABC / 微软 / 搜狗 / 小鹤 / 紫光 / 加加）、t9 九键、melt_eng、radical_pinyin。

### 5.1 候选词个数（5 → 9）

候选个数由 rime 方案的 `menu/page_size` 决定，**不是** fcitx5 的设置 ——
fcitx5 只负责渲染 rime 给出的候选。rime-ice 把 `page_size: 5` 写死在 `default.yaml` 里。

新建 `~/.local/share/fcitx5/rime/default.custom.yaml`：

```yaml
patch:
  menu/page_size: 9
```

> **为什么用 `.custom.yaml` 而不是直接改 `default.yaml`**：
> `default.yaml` 属于 rime-ice 仓库，`git pull` 更新词库时会覆盖掉修改；
> `.custom.yaml` 是 rime 官方的用户覆盖层，不会被覆盖。

- **9 是单键选择的上限**：数字键 1-9 正好一一对应
- 改完重启 fcitx5 即生效，不需要重新登录
- 验证：`grep -A2 '^menu:' ~/.local/share/fcitx5/rime/build/default.yaml` → 应为 `page_size: 9`

### 5.2 简繁输出

`rime_ice.schema.yaml` 的 `traditionalization` switch 只定义了 `states: [简, 繁]`，
**没有 `reset`**，按 rime 规则取第一态 → **默认简体**。

要改成默认繁体，新建 `~/.local/share/fcitx5/rime/rime_ice.custom.yaml`：

```yaml
patch:
  switches/@2/reset: 1
```

> `@2` 是 `switches` 列表里的下标：0=ascii_mode、1=ascii_punct、2=traditionalization。
> 用下标 patch 可以只覆盖这一个字段，不会把整个 `switches` 列表替换掉。

临时切换仍可用 `F4` 菜单。

---

## 6. 生效方式：注销重登（必须）

环境变量与自启动项只在**会话启动时**加载，配置完必须注销重新登录一次。
不重登的话 GUI 应用不会走 fcitx5。

ibus 的清理不需要重登，残留进程随注销消失。

---

## 7. 日常使用 / 维护

| 操作 | 按键 |
|---|---|
| 中英切换 | `Ctrl+Space` |
| 切换方案 / 简繁 | `F4` 或 `` Ctrl+` `` |

```bash
# 更新词库（pull 完必须重新部署，否则词库不生效）
git -C ~/.local/share/fcitx5/rime pull
fcitx5-remote -e; nohup setsid fcitx5 -d >/dev/null 2>&1 < /dev/null &
# 也可以直接点托盘图标右键 → 重新部署

# 改了配置后同理：重启 fcitx5 即可，不需要重新登录

# 排障
fcitx5-remote -n        # 当前引擎名（应为 rime）
fcitx5-remote -s rime   # 手动切到 rime
fcitx5-diagnose         # 完整环境诊断
```

**自定义一律写 `*.custom.yaml`，不要改仓库里的文件**（`git pull` 会把改动覆盖掉）：

| 要改的东西 | 写进哪个文件 |
|---|---|
| 全局设置（候选个数、方案列表等） | `~/.local/share/fcitx5/rime/default.custom.yaml` |
| 单个方案（简繁、模糊音等） | `~/.local/share/fcitx5/rime/rime_ice.custom.yaml` |

---

## 8. 回滚

```bash
# 回到原来的 rime 方案
mv ~/.local/share/fcitx5/rime ~/.local/share/fcitx5/rime.ice
mv ~/.local/share/fcitx5/rime.bak ~/.local/share/fcitx5/rime
fcitx5-remote -r
```

```bash
# 回到 ibus 输入法框架
im-config -n ibus
rm ~/.config/autostart/org.fcitx.Fcitx5.desktop
# 并清掉 ~/.config/environment.d/fcitx5.conf
```

---

## 9. 已知限制

1. **GNOME 原生 GTK4 应用**（文本编辑器、Nautilus、GNOME Console 等）在 GNOME Wayland
   下可能无法用 fcitx5 输入中文 —— GNOME 不实现第三方输入法协议所致。
   受影响时只能退回 `ibus-rime`（GNOME 内建支持 ibus，无此限制）。
2. GTK3 / Qt / Electron / Xwayland 应用正常。
3. `ibus` 包已移除时，GNOME「设置 → 键盘 → 输入源」界面可能报错或不再列出引擎；
   xkb 键盘布局本身由 mutter 处理，不受影响。

---

## 参考链接

- [fcitx5 官方仓库](https://github.com/fcitx/fcitx5)
- [雾凇拼音 rime-ice](https://github.com/iDvel/rime-ice)
- [Rime 官方网站](https://rime.im/)
- [fcitx5-nord 主题](https://github.com/tonyfettes/fcitx5-nord)
- [Fcitx5-Material-Color 主题](https://github.com/hosxy/Fcitx5-Material-Color)
