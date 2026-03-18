# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个 Obsidian Excalidraw 图标搜索插件，旨在为 Excalidraw 画布提供类似语雀画板的图标搜索功能。

### 核心功能需求

- **图标搜索**：调用语雀 iconfont API (`https://www.yuque.com/api/editor/iconfont`) 进行图标搜索
- **Cookie 认证管理**：提供多行输入框存储语雀登录 Cookie，支持验证和更新
- **悬浮搜索面板**：宫格布局显示搜索结果，自动适配面板宽度
- **拖拽/点击插入**：将搜索到的图标插入到 Excalidraw 画布中

### 参考项目

[zsviczian/obsidian-excalidraw-plugin](https://github.com/zsviczian/obsidian-excalidraw-plugin)

## 开发环境设置

```bash
# 安装依赖
npm install

# 开发模式（热重载）
npm run dev

# 生产构建
npm run build

# 代码格式化
npm run code:fix
```

### 构建工具

- **Rollup**：主要构建工具
- **TypeScript**：主要开发语言
- **React**：UI 组件（部分功能）
- **Node.js**：要求 >= 20.0.0
- **npm**：要求 >= 10.0.0

## 插件架构

### 目录结构（参考标准 Obsidian Excalidraw 插件）

```
src/
├── constants/          # 常量定义
├── core/              # 核心功能
│   ├── editor/        # 编辑器处理
│   └── managers/      # 各种管理器
├── shared/            # 共享组件
│   ├── Dialogs/       # 对话框组件
│   └── Suggesters/    # 建议器组件
├── types/             # 类型定义
├── utils/             # 工具函数
└── view/              # 视图组件
```

### Excalidraw Automate API (ea)

插件通过 `ea` 对象与 Excalidraw 交互，主要功能包括：

- **元素操作**：`ea.getElements()`, `ea.addElementsToView()`, `ea.selectElementsInView()`
- **样式设置**：`ea.style.strokeColor`, `ea.style.backgroundColor` 等
- **视图交互**：`ea.getViewSelectedElements()`, `ea.targetView`
- **设置管理**：`ea.getScriptSettings()`, `ea.setScriptSettings()`
- **工具函数**：`ea.addEllipse()`, `ea.addRectangle()`, `ea.addToGroup()`

### 插件脚本系统

Excalidraw 插件支持自定义脚本，脚本文件特征：
- 以 `/*` 开头
- 包含 Markdown 格式的说明
- 使用 JavaScript 编写
- 通过 `ea` API 访问 Excalidraw 功能

示例脚本结构：
```javascript
/*
![](icon-url)

脚本描述

```javascript
*/

// 脚本代码
if(!ea.verifyMinimumPluginVersion || !ea.verifyMinimumPluginVersion("1.5.21")) {
  new Notice("This script requires a newer version of Excalidraw.");
  return;
}

// 脚本逻辑
```

## 关键技术点

### 1. HTTP 请求处理

使用 `requestUrl` API 调用语雀接口：
```typescript
const response = await requestUrl({
  url: 'https://www.yuque.com/api/editor/iconfont',
  method: 'GET',
  headers: {
    'Cookie': userCookie,
    'User-Agent': 'Obsidian Excalidraw Plugin'
  },
  query: {
    query: searchTerm,
    page_size: 100,
    page: 1
  }
});
```

### 2. SVG 图标处理

将语雀返回的 SVG 字符串转换为 Excalidraw 可用的格式：
- 提取 `show_svg` 字段中的 SVG 代码
- 解析 SVG 路径和属性
- 使用 Excalidraw 的 SVG 导入功能

### 3. UI 组件开发

使用 Obsidian 的 Modal 和 Suggest 组件：
- `FuzzySuggestModal`：搜索建议模态框
- `Modal`：自定义悬浮面板
- 样式使用 `styles.css` 文件定义

### 4. 拖拽功能

利用 Excalidraw 的拖放 API 或鼠标事件：
```typescript
// 监听鼠标事件
api.addEventListener('click', (event) => {
  // 处理图标点击和插入
});
```

## 语雀 API 说明

### 认证要求

- 需要语雀登录 Cookie
- 未认证返回：`{"status":401,"message":"Unauthorized"}`
- Cookie 可能过期，需要循环验证

### API 响应格式

```json
{
  "data": {
    "icons": [
      {
        "id": 11372714,
        "name": "微博",
        "preview_image": "t/icon_poster2_768/11372714/xxx.png",
        "show_svg": "<svg>...</svg>",
        "width": 1024,
        "height": 1024
      }
    ],
    "count": 1030
  }
}
```

### 关键字段

- `name`：图标名称
- `show_svg`：SVG 代码字符串（用于插入画布）
- `preview_image`：预览图片 URL

## 插件设置

需要存储以下用户配置：
- **语雀 Cookie**：多行文本输入
- **搜索历史**：最近搜索关键词
- **面板设置**：宽度、每行显示数量等

## 开发注意事项

1. **国际化**：支持中英文界面
2. **错误处理**：处理 Cookie 过期、网络错误等
3. **性能优化**：搜索结果缓存、虚拟滚动（大量结果）
4. **用户体验**：
   - 搜索关键字保留
   - 图标预览加载状态
   - 鼠标悬停效果
   - 拖拽插入反馈

## 测试方法

1. 在 Obsidian 中启用插件开发者模式
2. 将构建后的插件复制到 Obsidian 插件目录
3. 重启 Obsidian
4. 打开 Excalidraw 画布测试功能

## 相关资源

- [Obsidian 插件开发文档](https://docs.obsidian.md/Plugins/Getting+started/Build+a+plugin)
- [Excalidraw Automation 文档](https://zsviczian.github.io/obsidian-excalidraw-plugin/ExcalidrawScriptsEngine.html)
- [Excalidraw 插件 GitHub](https://github.com/zsviczian/obsidian-excalidraw-plugin)
