# Iconfont Obsidian Search

为 Obsidian Excalidraw 提供语雀图标搜索功能的脚本。

## 功能特性

- 🔍 搜索语雀图标库（10万+ 图标）
- 🎨 宫格布局展示搜索结果
- 🖱️ 点击放置，操作简单
- 📦 滚动加载更多结果
- 💾 保存搜索历史

## 安装

### 方法一：通过 Excalidraw 脚本库（推荐）

1. 打开 Excalidraw 画布
2. 点击面板菜单 → "Install or update Excalidraw Scripts"
3. 找到 "语雀图标搜索"
4. 点击 "Install this script"
5. 重启 Obsidian

### 方法二：手动安装

1. 下载 `iconfont-obsidian-search.md`
2. 复制到 vault 的 `Excalidraw/Scripts/Downloaded/` 目录
3. 重启 Obsidian
4. 在 Excalidraw 设置中启用脚本

## 配置

首次使用需要配置语雀 Cookie：

1. 在浏览器登录 [语雀](https://www.yuque.com)
2. 按 `F12` 打开开发者工具
3. 切换到 `Application` / `应用` 标签
4. 左侧找到 `Cookies` → `https://www.yuque.com`
5. 复制所有 Cookie 值（格式: `name=value; name2=value2; ...`）
6. 运行脚本，按提示粘贴 Cookie

## 使用方法

1. 在 Excalidraw 画布中运行脚本（工具栏图标或快捷键）
2. 在搜索框中输入关键词（如"微信"、"微博"等）
3. 浏览搜索结果，点击选中的图标
4. 鼠标变为十字光标，点击画布位置插入图标
5. 按 `ESC` 或右键取消放置

## 快捷键设置

1. 打开 Obsidian 设置
2. 找到 "Excalidraw" → "Script Commands"
3. 找到 "语雀图标搜索"
4. 点击右侧设置快捷键（如 `Ctrl+Shift+I`）

## 技术栈

- Excalidraw Automate Script Engine
- 原生 JavaScript (ES6+)
- CSS Grid
- Obsidian requestUrl API

## 安全说明

- Cookie 仅存储在本地 Excalidraw 设置中
- 所有 SVG 内容经过安全清理，防止 XSS 攻击
- 不收集任何用户数据

## 许可证

MIT License

## 问题反馈

如有问题请在 GitHub 提交 Issue。
