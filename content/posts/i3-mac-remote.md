---
title: "Mac 远程控制 i3wm：如何通过 UI 按钮解决快捷键冲突？"
date: 2026-01-31
draft: false
tags:
  - "i3wm"
  - "linux"
  - "mac"
  - "remote-desktop"
categories:
  - "tutorials"
---

![i3 Status Bar Preview](/images/i3_bar_right.png)

<!--more-->

如果你和我一样，喜欢在 Linux 服务器上使用 **i3wm** 这种高效的平铺式窗口管理器，同时又必须使用 **Mac** 通过 VNC 或远程桌面软件来连接它，你一定遇到过这个痛点：**快捷键冲突**。

Mac 的 `Command`、`Option`、`Control` 键位映射在远程桌面上往往是一场灾难。i3 及其依赖的 `Mod` 键（通常是 Alt 或 Win）经常和 Mac 本地的快捷键打架。原本行云流水的 `Mod+Shift+Q`（关闭窗口）或 `Mod+Enter`（打开终端），在远程环境下变得极度别扭。

与其在键位映射的泥潭里挣扎，不如换个思路：**既然是远程操作，既然手边就在用鼠标/触控板，为什么不把常用的高频操作变成状态栏上的按钮呢？**

本文将介绍如何通过配置 `i3blocks`，将 i3 改造为一个对 Mac 远程用户友好的“鼠标流”环境。

## 1. 准备工作：安装依赖

我们需要安装字体以支持图标，以及一个好用的应用启动器 `rofi`（比默认的 dmenu 更适合鼠标操作）。

```bash
# Debian/Ubuntu
sudo apt-get install fonts-font-awesome rofi
```

然后，在 i3 配置文件（`~/.config/i3/config`）中启用字体：

```text
font pango:monospace 8, FontAwesome 8
```

## 2. 核心组件：Clickable 脚本

为了方便管理点击事件，我们不在配置文件里写复杂的命令，而是建立一个通用的脚本。
创建文件 `~/.config/i3blocks/clickable`：

```bash
#!/bin/bash
# 用法: clickable "标签" "左键i3命令" "颜色" "右键i3命令"

LABEL="$1"
CMD_LEFT="$2"
COLOR="${3:-#FFFFFF}"
CMD_RIGHT="$4"

# 输出给 i3blocks
echo "$LABEL"
echo "$LABEL"
echo "$COLOR"

# 事件处理
if [ "$BLOCK_BUTTON" == "1" ]; then
    # 发送 i3 内部命令（如 exec, layout toggle 等）
    i3-msg "$CMD_LEFT" >/dev/null 2>&1 &
elif [ "$BLOCK_BUTTON" == "3" ] && [ -n "$CMD_RIGHT" ]; then
    i3-msg "$CMD_RIGHT" >/dev/null 2>&1 &
fi
```

## 3. 进阶组件：自动新建工作区脚本

这是实现“一键新建工作区”逻辑的关键脚本。它会自动寻找当前最大的工作区编号并 +1。
创建文件 `~/.config/i3blocks/new_workspace`：

```bash
#!/bin/bash
# 如果已安装 FontAwesome，可将图标换回 
echo " ➕ "  
echo " New "
echo "#00FF00" # 绿色

if [ "$BLOCK_BUTTON" == "1" ]; then
    # 获取当前最大工作区号，处理空值，计算下一个号码
    MAX_WS=$(i3-msg -t get_workspaces | tr ',' '\n' | grep '"num":' | awk -F: '{print $2}' | sort -n | tail -1)
    [ -z "$MAX_WS" ] && MAX_WS=0
    NEW_WS=$((MAX_WS + 1))
    i3-msg workspace number $NEW_WS >/dev/null 2>&1
fi
```

**重要：** 别忘了给脚本加上执行权限！
```bash
chmod +x ~/.config/i3blocks/clickable ~/.config/i3blocks/new_workspace
```

## 4. 配置状态栏按钮

编辑 `~/.config/i3blocks/config`。为了方便阅读，这里暂时使用通用 Emoji 图标。
*(注：如果你按步骤 1 安装了 FontAwesome，可以将图标替换为 `` `` 等专业图标)*

请注意：下面的路径使用了 `~`，如果你的 i3blocks 不支持波浪号展开，请写绝对路径（例如 `/home/yourname/...`）。

```ini
# 全局默认
command=/usr/share/i3blocks/$BLOCK_NAME
separator_block_width=15
markup=none

# 1. 应用启动器
[launcher]
# 如果已安装 FontAwesome 可用: 
full_text= 📱 
command=echo " 📱 "; [[ -n "$BLOCK_BUTTON" ]] && (rofi -show drun &)
interval=once
separator=true

# 2. 终端 (青色)
[term]
interval=once
# FontAwesome: 
command=~/.config/i3blocks/clickable " 📟 " "exec i3-sensible-terminal" "#00FFFF"
separator=true

# 3. 关闭窗口 (红色)
[kill]
interval=once
# FontAwesome: 
command=~/.config/i3blocks/clickable " ❌ " "kill" "#FF5555"
separator=true

# 4. 布局切换 (绿色)
[layout]
interval=once
# FontAwesome: 
command=~/.config/i3blocks/clickable " 🔲 " "layout toggle split" "#00FF00" "layout tabbed"
separator=true

# 5. 新建工作区
[new_workspace]
interval=once
# FontAwesome: 
command=~/.config/i3blocks/new_workspace
separator=true

# 6. 重启 i3 (橙色)
[restart]
interval=once
# FontAwesome: 
command=~/.config/i3blocks/clickable " 🔄 " "restart" "#FF8800"
separator=true
```

## 5. Mac 专属优化（位置与鼠标习惯）

修改 `~/.config/i3/config`：

**1. 状态栏移到顶部**
避开 Mac 底部的 Dock 栏，防止误触。
```text
bar {
        status_command i3blocks
        position top  # 改为 top
}
```

**2. 右键标题栏关闭窗口**
这是最爽的一点。i3 默认没有关闭按钮？那就把**整个标题栏**变成关闭按钮。
```text
# 释放右键时关闭窗口
bindsym --release button3 kill
```

## 6. 生效

最后，使用快捷键 `Mod+Shift+R` 重启 i3。
现在，你的顶部状态栏应该有一排漂亮的图标，你的鼠标右键也有了实际用途。对于 VNC/远程桌面用户来说，这套配置能极大降低对复杂快捷键的依赖。
