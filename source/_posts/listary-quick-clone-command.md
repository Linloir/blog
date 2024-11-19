---
title: Listary 命令分享 - 快捷 clone 仓库并使用 VSCode 打开
date: 2024-11-19 14:08:32
tags:
  - 奇技淫巧
categories:
  - 技术
---

## 背景

日常工作中，经常会需要临时 Clone 某个仓库并且用 VSCode 打开，在 Windows 上我一般都是：

1. 用文件资源管理器定位到需要 Clone 到的位置然后右键呼出终端
2. `git clone`
3. `cd <cloned repo>`
4. `code .`

这个操作多少有些不便：

1. 要等 Windows 11 呼出右键菜单还是挺慢的
2. 输入各种指令需要从鼠标转换到键盘
3. `code .` 会启动 VSCode 但是不会关掉终端，并且焦点会转到 VSCode，这时候想要关掉终端就又要换回鼠标点一次，怪烦的

P.S. 我当然知道直接一直 terminal 就没这些问题了，或者 `code .; exit` 也可以解决上面的第 3 点，但是 listary 的指令真的用一次就会爱上，所以还是小小捣鼓了下

## 方案

右键 Listary > `选项` > `命令` > 选择添加 (`+` 按钮) > 填入如下配置：

- 关键字: `clode`
- 标题: `Clone and open "{query}"`
- 路径: `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`
- 参数: `-Command "$input = '{query}'; $parts = $input -split ' '; $repo = $parts[0]; $dest = if ($parts.Count -gt 1) { $parts[1] } else { [System.IO.Path]::GetFileNameWithoutExtension($repo) }; git clone $repo $dest; code $dest"`

之后，在文件夹中直接输入 `clode <repo url> [dest folder]` 即可一键 clone

`[dest folder]` 为选填，不填则是默认取 `<repo url>` 末尾仓库名称作为目标文件夹，同直接执行 `git clone <repo url>` 行为保持一致

![Settings](/img/listary-quick-clone-command/listary_settings.png)
