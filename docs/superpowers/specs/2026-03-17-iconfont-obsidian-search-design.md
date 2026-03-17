# Iconfont Obsidian Search - 设计文档

**项目名称**: Iconfont Obsidian Search
**版本**: 1.0.0
**日期**: 2026-03-17
**作者**: Claude Code
**状态**: 设计阶段

---

## 1. 项目概述

### 1.1 目标

为 Obsidian Excalidraw 创建一个图标搜索插件，提供类似语雀画板的图标搜索功能。用户可以通过关键词搜索语雀图标库中的图标，并将选中的图标插入到 Excalidraw 画布中。

### 1.2 核心价值

- **快速图标检索**: 无需离开 Excalidraw 画布即可搜索图标
- **无缝集成**: 作为 Excalidraw Automate 脚本，完美集成到现有工作流
- **开源友好**: 原生 JavaScript 实现，无外部依赖，易于二次开发

### 1.3 参考项目

- [语雀画板图标搜索](https://www.yuque.com/) - UI/UX 参考
- [zsviczian/obsidian-excalidraw-plugin](https://github.com/zsviczian/obsidian-excalidraw-plugin) - 技术基础

---

## 2. 技术栈

| 组件 | 技术 |
|------|------|
| 脚本引擎 | Excalidraw Automate Script Engine |
| 编程语言 | 原生 JavaScript (ES6+) |
| DOM 操作 | 原生 DOM API |
| 布局 | CSS Grid |
| HTTP 请求 | Obsidian `requestUrl` API |
| 持久化 | Excalidraw `ea.setScriptSettings()` |
| 图标处理 | Excalidraw `ea.svgToExcalidrawElements()` |

**设计决策**: 选择原生 JavaScript 而非框架，以确保与 Excalidraw 脚本生态保持一致，降低用户学习成本，避免外部依赖问题。

---

## 3. 功能需求

### 3.1 核心功能

| 功能编号 | 功能描述 | 优先级 |
|---------|---------|--------|
| F1 | 搜索语雀图标库 | P0 |
| F2 | 显示搜索结果（宫格布局） | P0 |
| F3 | 点击选择图标并插入画布 | P0 |
| F4 | Cookie 认证管理 | P0 |
| F5 | 滚动加载更多结果 | P1 |
| F6 | 保存上次搜索关键词 | P1 |

### 3.2 用户故事

**US1**: 作为 Excalidraw 用户，我希望在画布中快速搜索图标，以便丰富我的绘图内容。

**US2**: 作为 Excalidraw 用户，我希望能够通过点击和放置的方式插入图标，操作应该简单直观。

**US3**: 作为插件用户，我希望只需要配置一次 Cookie，之后就能直接使用搜索功能。

### 3.3 非功能需求

| 需求 | 指标 |
|------|------|
| 响应时间 | 搜索 API 调用 < 2s |
| 用户体验 | UI 流畅，无卡顿 |
| 兼容性 | Excalidraw >= 1.5.21 |
| 可维护性 | 代码模块化，注释完整 |
| 开源友好 | 无外部依赖，MIT 许可证 |

---

## 4. 架构设计

### 4.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Excalidraw 画布                          │
│                                                              │
│  ┌── 悬浮搜索面板 (HTML/CSS) ────────────────────┐         │
│  │  搜索输入框                                    │         │
│  │  ┌───┐┌───┐┌───┐┌───┐  (图标宫格)            │         │
│  │  │...││...││...││...│                         │         │
│  │  └───┘└───┘└───┘└───┘                         │         │
│  └───────────────────────────────────────────────────┘     │
│                                                              │
│  [画布内容...]                         [+图标搜索]         │
└─────────────────────────────────────────────────────────────┘

                 ┌─────────────────┐
                 │ Excalidraw API  │
                 │   (ea 对象)     │
                 └────────┬────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
    ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
    │ Cookie  │     │ 搜索API  │     │ 图标插入 │
    │ 管理    │     │ 调用     │     │         │
    └─────────┘     └─────────┘     └─────────┘
```

### 4.2 模块划分

```
iconfont-obsidian-search.md
├── 脚本元数据
├── CSS 样式定义
├── JavaScript 模块
│   ├── 初始化模块 (init)
│   ├── 设置管理模块 (settings)
│   ├── API 调用模块 (api)
│   ├── UI 模块 (ui)
│   │   ├── 面板创建
│   │   ├── 事件绑定
│   │   └── 图标渲染
│   ├── 放置模式模块 (placing)
│   └── 工具函数模块 (utils)
```

### 4.3 数据流

```
用户输入关键词
    │
    ▼
[防抖 300ms]
    │
    ▼
调用语雀 API
    │
    ├─→ 成功 → 解析图标数据 → 更新宫格显示
    │
    └─→ 失败 → Notice 提示错误

用户点击图标
    │
    ▼
保存搜索关键词 + 关闭面板
    │
    ▼
进入放置模式 (鼠标变十字)
    │
    ├─→ 点击画布 → 插入图标 → 退出
    │
    └─→ 按 ESC → 取消 → 退出
```

---

## 5. 数据结构

### 5.1 脚本设置 (持久化)

```javascript
const defaultSettings = {
    // 语雀 Cookie (多行字符串)
    yuqueCookie: "",

    // 上一次的搜索关键词
    lastQuery: ""
};
```

### 5.2 运行时状态 (内存)

```javascript
const state = {
    // UI 状态
    isOpen: false,          // 面板是否打开
    isLoading: false,       // 是否正在加载

    // 搜索状态
    icons: [],              // 当前加载的图标列表
    currentPage: 1,         // 当前页码
    hasMore: true,          // 是否有更多结果

    // 选中状态
    selectedIcon: null,     // 选中的图标数据
    isPlacingMode: false    // 是否处于放置模式
};
```

### 5.3 图标数据结构

```javascript
// 语雀 API 返回的图标数据
interface YuqueIcon {
    id: number;              // 图标 ID
    name: string;            // 图标名称
    preview_image: string;   // 预览图 URL
    show_svg: string;        // SVG 代码
    width: number;           // 原始宽度
    height: number;          // 原始高度
}
```

---

## 6. 模块详细设计

### 6.1 初始化模块

```javascript
/**
 * 脚本入口函数
 */
async function main() {
    // 1. 版本检查
    if (!ea.verifyMinimumPluginVersion("1.5.21")) {
        new Notice("此脚本需要 Excalidraw 1.5.21 或更高版本");
        return;
    }

    // 2. 加载设置
    loadSettings();

    // 3. Cookie 检查
    if (!settings.yuqueCookie) {
        await promptForCookie();
    }

    // 4. 打开搜索面板
    openSearchPanel();
}
```

### 6.2 设置管理模块

```javascript
/**
 * 从 Excalidraw 加载设置
 */
function loadSettings() {
    const saved = ea.getScriptSettings();
    settings = { ...defaultSettings, ...saved };
}

/**
 * 保存设置到 Excalidraw
 */
function saveSettings() {
    ea.setScriptSettings(settings);
}

/**
 * 提示用户输入 Cookie
 */
async function promptForCookie() {
    const cookie = await utils.inputText(
        "请输入语雀 Cookie",
        "在浏览器登录语雀后:\n" +
        "1. 按 F12 打开开发者工具\n" +
        "2. 切换到 Application/应用 标签\n" +
        "3. 左侧找到 Cookies → https://www.yuque.com\n" +
        "4. 复制所有 Cookie 值（格式: name=value; name2=value2; ...）",
        settings.yuqueCookie || ""
    );

    if (cookie) {
        settings.yuqueCookie = cookie;
        saveSettings();
        new Notice("Cookie 已保存");
    }
}
```

### 6.3 API 调用模块

```javascript
/**
 * 搜索语雀图标
 * @param {string} query - 搜索关键词
 * @param {number} page - 页码
 * @param {number} pageSize - 每页数量
 * @returns {Promise<YuqueIcon[]>} 图标列表
 */
async function searchIcons(query, page = 1, pageSize = 100) {
    if (!query.trim()) return [];

    try {
        const response = await requestUrl({
            url: 'https://www.yuque.com/api/editor/iconfont',
            method: 'GET',
            headers: {
                'Cookie': settings.yuqueCookie,
                'User-Agent': 'Obsidian Excalidraw IconSearch/1.0'
            },
            query: {
                query: query,
                page_size: pageSize,
                page: page
            }
        });

        // 认证检查
        if (response.status === 401) {
            throw new Error("UNAUTHORIZED");
        }

        const data = response.json;
        state.hasMore = data.data.icons.length === pageSize;
        return data.data.icons || [];

    } catch (error) {
        handleApiError(error);
        return [];
    }
}

/**
 * 处理 API 错误
 */
function handleApiError(error) {
    if (error.message === "UNAUTHORIZED") {
        new Notice("语雀 Cookie 失效了，请重新获取后更新配置，再继续使用。");
    } else {
        new Notice(`搜索失败: ${error.message}`);
    }
}
```

### 6.4 UI 模块 - 面板创建

```javascript
/**
 * 打开搜索面板
 */
function openSearchPanel() {
    if (state.isOpen) return;

    // 创建面板 DOM
    panelElement = createPanelDOM();
    document.body.appendChild(panelElement);
    state.isOpen = true;

    // 初始化搜索框
    const searchInput = panelElement.querySelector('.search-input');
    if (settings.lastQuery) {
        searchInput.value = settings.lastQuery;
    }
    searchInput.focus();

    // 绑定事件
    bindPanelEvents();
}

/**
 * 创建面板 DOM 结构
 */
function createPanelDOM() {
    const panel = document.createElement('div');
    panel.className = 'icon-search-panel';
    panel.innerHTML = `
        <div class="panel-header">
            <span class="panel-title">语雀图标搜索</span>
            <button class="close-btn" title="关闭">×</button>
        </div>
        <div class="panel-body">
            <input type="text" class="search-input" placeholder="输入图标名称搜索...">
            <div class="icons-grid"></div>
            <div class="status-text"></div>
        </div>
    `;
    return panel;
}

/**
 * 绑定面板事件
 */
function bindPanelEvents() {
    // 搜索输入（防抖）
    const searchInput = panelElement.querySelector('.search-input');
    let debounceTimer;
    searchInput.addEventListener('input', (e) => {
        clearTimeout(debounceTimer);
        debounceTimer = setTimeout(() => {
            performSearch(e.target.value);
        }, 300);
    });

    // 滚动加载更多
    const panelBody = panelElement.querySelector('.panel-body');
    panelBody.addEventListener('scroll', () => {
        if (!state.isLoading && state.hasMore) {
            const scrollBottom = panelBody.scrollTop + panelBody.clientHeight;
            if (scrollBottom >= panelBody.scrollHeight - 50) {
                loadMoreIcons();
            }
        }
    });

    // 关闭按钮
    panelElement.querySelector('.close-btn').addEventListener('click', closePanel);
}

/**
 * 执行搜索
 */
async function performSearch(query) {
    if (state.isLoading) return;

    state.isLoading = true;
    updateStatus("加载中...");

    const icons = await searchIcons(query, 1);
    state.icons = icons;
    state.currentPage = 1;

    renderIcons(icons);
    updateStatus(icons.length === 0 ? "未找到相关图标" : "");

    state.isLoading = false;
}

/**
 * 加载更多图标
 */
async function loadMoreIcons() {
    if (state.isLoading || !state.hasMore) return;

    state.isLoading = true;
    updateStatus("加载更多...");

    const searchInput = panelElement.querySelector('.search-input');
    const newIcons = await searchIcons(searchInput.value, state.currentPage + 1);

    state.icons.push(...newIcons);
    state.currentPage++;
    renderIcons(newIcons, true); // 追加模式

    updateStatus("");
    state.isLoading = false;
}

/**
 * 渲染图标到宫格
 * @param {YuqueIcon[]} icons - 图标列表
 * @param {boolean} append - 是否追加到现有内容
 */
function renderIcons(icons, append = false) {
    const grid = panelElement.querySelector('.icons-grid');
    if (!append) grid.innerHTML = '';

    icons.forEach(icon => {
        const iconEl = document.createElement('div');
        iconEl.className = 'icon-item';
        iconEl.title = icon.name;
        iconEl.innerHTML = icon.show_svg;
        iconEl.dataset.iconData = JSON.stringify(icon);

        iconEl.addEventListener('click', () => selectIcon(icon));
        grid.appendChild(iconEl);
    });
}

/**
 * 更新状态文本
 */
function updateStatus(text) {
    const statusEl = panelElement.querySelector('.status-text');
    statusEl.textContent = text;
}

/**
 * 关闭面板
 */
function closePanel() {
    if (panelElement) {
        panelElement.remove();
        panelElement = null;
    }
    state.isOpen = false;
}
```

### 6.5 图标选择与插入模块

```javascript
/**
 * 选择图标，进入放置模式
 * @param {YuqueIcon} icon - 选中的图标
 */
function selectIcon(icon) {
    // 保存搜索关键词
    const searchInput = panelElement.querySelector('.search-input');
    settings.lastQuery = searchInput.value;
    saveSettings();

    // 设置状态
    state.selectedIcon = icon;
    state.isPlacingMode = true;

    // 关闭面板
    closePanel();

    // 进入放置模式
    enterPlacingMode();
}

/**
 * 进入放置模式
 */
function enterPlacingMode() {
    // 更改鼠标样式
    document.body.style.cursor = 'crosshair';

    // 显示提示
    const tip = new Notice("点击画布位置插入图标 (按 ESC 取消)", 0);

    // 清理函数
    const cleanup = () => {
        document.removeEventListener('click', handleCanvasClick);
        document.removeEventListener('keydown', handleEscape);
        document.body.style.cursor = '';
        tip.hide();
    };

    // 画布点击处理
    const handleCanvasClick = async (e) => {
        // 确保点击的是画布区域
        const view = ea.targetView;
        if (!view || !view.containerEl.contains(e.target)) return;

        const point = getCanvasCoordinates(e);
        await insertIconToCanvas(point);

        cleanup();
        state.isPlacingMode = false;
    };

    // ESC 取消处理
    const handleEscape = (e) => {
        if (e.key === 'Escape') {
            cleanup();
            state.isPlacingMode = false;
        }
    };

    document.addEventListener('click', handleCanvasClick);
    document.addEventListener('keydown', handleEscape);
}

/**
 * 获取画布坐标
 * @param {MouseEvent} event - 鼠标事件
 * @returns {Point} 画布坐标
 */
function getCanvasCoordinates(event) {
    const view = ea.targetView;
    const rect = view.containerEl.getBoundingClientRect();
    return {
        x: event.clientX - rect.left,
        y: event.clientY - rect.top
    };
}

/**
 * 将图标插入到画布
 * @param {Point} point - 插入位置
 */
async function insertIconToCanvas(point) {
    const icon = state.selectedIcon;
    if (!icon) return;

    const size = 32; // 32x32 像素

    try {
        // 使用 Excalidraw API 转换 SVG
        const elements = await ea.svgToExcalidrawElements(icon.show_svg, {
            x: point.x - size / 2,
            y: point.y - size / 2,
            width: size,
            height: size,
            strokeColor: '#000000',
            fillColor: 'transparent'
        });

        ea.elements = elements;
        await ea.addElementsToView(false, true, true);
        ea.selectElementsInView(ea.getElements());

    } catch (error) {
        new Notice(`插入图标失败: ${error.message}`);
    }
}
```

### 6.6 工具函数模块

```javascript
/**
 * 防抖函数
 * @param {Function} func - 要防抖的函数
 * @param {number} wait - 等待时间
 * @returns {Function} 防抖后的函数
 */
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

/**
 * 转义 HTML 特殊字符
 * @param {string} text - 原始文本
 * @returns {string} 转义后的文本
 */
function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}
```

---

## 7. 交互设计

### 7.1 用户流程图

```
┌──────────────┐
│  触发脚本     │
│ (工具栏/快捷键)│
└──────┬───────┘
       │
       ▼
┌──────────────┐    没有
│ 检查 Cookie  ├──────────┐
└──────┬───────┘          │
       │ 有               │
       ▼                  ▼
┌──────────────┐   ┌──────────────┐
│ 打开搜索面板  │   │ 提示输入Cookie│
└──────┬───────┘   └──────┬───────┘
       │                  │
       │                  └──────────┐
       │                             │
       ▼                             ▼
       ┌──────────────────────────────┐
       │      输入搜索关键词            │
       └──────────────┬───────────────┘
                      │
                      ▼
              ┌───────────────┐
              │ 防抖 300ms    │
              └───────┬───────┘
                      │
                      ▼
              ┌───────────────┐
              │ 调用语雀 API  │
              └───────┬───────┘
                      │
         ┌────────────┴────────────┐
         │                         │
         ▼                         ▼
    ┌─────────┐              ┌─────────┐
    │ 成功    │              │ 失败    │
    └────┬────┘              └────┬────┘
         │                        │
         ▼                        ▼
  ┌─────────────┐        ┌─────────────┐
  │ 显示图标宫格 │        │ Notice 提示  │
  └──────┬──────┘        └─────────────┘
         │
         ├─────────────┐
         │             │
         ▼             ▼
    ┌─────────┐   ┌─────────┐
    │ 点击图标 │   │ 滚动底部 │
    └────┬────┘   └────┬────┘
         │             │
         │             ▼
         │      ┌─────────────┐
         │      │ 加载下一页   │
         │      └─────────────┘
         │
         ▼
  ┌─────────────────┐
  │ 保存搜索关键词   │
  │ 关闭搜索面板     │
  └────────┬────────┘
           │
           ▼
  ┌─────────────────┐
  │ 进入放置模式     │
  │ (鼠标变十字)     │
  └────────┬────────┘
           │
      ┌────┴────┐
      │         │
      ▼         ▼
  ┌──────┐  ┌──────┐
  │点击画布│  │ESC取消│
  └──┬───┘  └───────┘
     │
     ▼
  ┌─────────┐
  │ 插入图标 │
  └─────────┘
```

### 7.2 状态机

```
                    [初始化]
                        │
                        ▼
                   ┌─────────┐
                   │  IDLE   │
                   └────┬────┘
                        │ 触发脚本
                        ▼
                   ┌─────────┐
                   │ SEARCH  │
                   │ 搜索中  │
                   └────┬────┘
                        │
           ┌────────────┴────────────┐
           │                         │
           ▼                         ▼
      ┌─────────┐              ┌─────────┐
      │RESULTS  │              │  ERROR  │
      │浏览结果 │              │  错误   │
      └────┬────┘              └─────────┘
           │
           │ 点击图标
           ▼
      ┌─────────┐
      │ SELECT  │
      │已选择   │
      └────┬────┘
           │
           ▼
      ┌─────────┐
      │ PLACING │
      │放置模式 │
      └────┬────┘
           │
      ┌────┴────┐
      │         │
      ▼         ▼
  ┌───────┐ ┌───────┐
  │INSERT │ │CANCEL │
  │已插入 │ │已取消 │
  └───┬───┘ └───────┘
      │
      ▼
   [IDLE]
```

---

## 8. 样式设计

### 8.1 CSS 完整定义

```css
/* ===== 悬浮面板 ===== */
.icon-search-panel {
    position: fixed;
    top: 100px;
    right: 50px;
    width: 400px;
    max-height: 500px;
    background: var(--background-secondary);
    border: 1px solid var(--background-modifier-border);
    border-radius: 8px;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
    z-index: 1000;
    display: flex;
    flex-direction: column;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

/* ===== 面板头部 ===== */
.panel-header {
    padding: 12px 16px;
    border-bottom: 1px solid var(--background-modifier-border);
    display: flex;
    justify-content: space-between;
    align-items: center;
    background: var(--background-modifier-hover);
    border-radius: 8px 8px 0 0;
}

.panel-title {
    font-weight: 600;
    font-size: 14px;
    color: var(--text-normal);
}

.close-btn {
    background: none;
    border: none;
    font-size: 20px;
    color: var(--text-muted);
    cursor: pointer;
    padding: 0;
    width: 24px;
    height: 24px;
    line-height: 1;
    transition: color 0.2s;
}

.close-btn:hover {
    color: var(--text-normal);
}

/* ===== 面板主体 ===== */
.panel-body {
    padding: 0;
    overflow-y: auto;
    flex: 1;
}

/* ===== 搜索输入框 ===== */
.search-input {
    width: calc(100% - 32px);
    margin: 12px 16px;
    padding: 8px 12px;
    border: 1px solid var(--background-modifier-border);
    border-radius: 4px;
    background: var(--background-primary);
    color: var(--text-normal);
    font-size: 14px;
    outline: none;
    transition: border-color 0.2s;
}

.search-input:focus {
    border-color: var(--interactive-accent);
}

/* ===== 图标宫格 ===== */
.icons-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(32px, 1fr));
    gap: 8px;
    padding: 0 16px 16px;
    max-height: 350px;
    overflow-y: auto;
}

/* ===== 单个图标项 ===== */
.icon-item {
    width: 32px;
    height: 32px;
    padding: 8px;
    cursor: pointer;
    border-radius: 4px;
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--background-primary);
    border: 1px solid transparent;
}

.icon-item:hover {
    background: var(--background-modifier-hover);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    transform: scale(1.1);
    border-color: var(--background-modifier-border);
}

.icon-item svg {
    width: 16px;
    height: 16px;
    fill: currentColor;
    pointer-events: none;
}

/* ===== 状态文本 ===== */
.status-text {
    text-align: center;
    padding: 12px;
    color: var(--text-muted);
    font-size: 12px;
}

/* ===== 滚动条样式 ===== */
.panel-body::-webkit-scrollbar,
.icons-grid::-webkit-scrollbar {
    width: 6px;
}

.panel-body::-webkit-scrollbar-track,
.icons-grid::-webkit-scrollbar-track {
    background: transparent;
}

.panel-body::-webkit-scrollbar-thumb,
.icons-grid::-webkit-scrollbar-thumb {
    background: var(--background-modifier-border);
    border-radius: 3px;
}

.panel-body::-webkit-scrollbar-thumb:hover,
.icons-grid::-webkit-scrollbar-thumb:hover {
    background: var(--text-muted);
}
```

### 8.2 响应式适配

| 屏幕宽度 | 面板宽度 | 每行图标数 |
|---------|---------|-----------|
| > 1200px | 400px | 自动 (约9-10个) |
| 768-1200px | 350px | 自动 (约8-9个) |
| < 768px | 300px | 自动 (约7-8个) |

---

## 9. 错误处理

### 9.1 错误类型与处理策略

| 错误场景 | 检测方式 | 处理策略 | 用户提示 |
|---------|---------|---------|---------|
| Cookie 未配置 | `settings.yuqueCookie` 为空 | 首次运行时提示输入 | "首次使用请先配置语雀 Cookie" |
| Cookie 失效 | API 返回 401 | Notice 提示，不阻止后续操作 | "语雀 Cookie 失效了，请重新获取后更新配置，再继续使用。" |
| 网络连接失败 | `requestUrl` 抛出错误 | Notice 提示 | "网络连接失败，请检查网络后重试" |
| 搜索无结果 | API 返回空数组 | 面板内显示提示 | "未找到相关图标" |
| API 其他错误 | 非 401 状态码 | Notice 提示具体错误 | "搜索失败: {错误信息}" |
| SVG 解析失败 | `svgToExcalidrawElements` 失败 | Notice 提示 | "插入图标失败: {错误信息}" |

### 9.2 错误恢复机制

```
错误发生
    │
    ├─→ Cookie 失效 ──→ 提示用户 ──→ 用户可继续操作，下次搜索时重试
    │
    ├─→ 网络错误 ────→ 提示用户 ──→ 用户可重试搜索
    │
    ├─→ 解析错误 ────→ 提示用户 ──→ 用户可选择其他图标
    │
    └─→ 其他错误 ────→ 记录日志 ──→ Notice 提示
```

---

## 10. 性能考虑

### 10.1 性能目标

| 指标 | 目标值 |
|------|--------|
| 搜索响应时间 | < 2s |
| 面板打开时间 | < 100ms |
| 图标渲染时间 | < 500ms (100个) |
| 内存占用 | < 10MB |

### 10.2 优化策略

**1. 搜索防抖**
```javascript
// 输入后 300ms 才执行搜索，避免频繁 API 调用
const debouncedSearch = debounce(performSearch, 300);
```

**2. DOM 批量操作**
```javascript
// 使用 DocumentFragment 批量插入
const fragment = document.createDocumentFragment();
icons.forEach(icon => {
    const el = createIconElement(icon);
    fragment.appendChild(el);
});
grid.appendChild(fragment);
```

**3. 事件委托**
```javascript
// 在宫格容器上使用事件委托，而非每个图标绑定事件
grid.addEventListener('click', (e) => {
    const iconItem = e.target.closest('.icon-item');
    if (iconItem) {
        const icon = JSON.parse(iconItem.dataset.iconData);
        selectIcon(icon);
    }
});
```

**4. 懒加载**
- 滚动到底部时才加载下一页
- 已加载的图标缓存在 `state.icons` 中

**5. 清理资源**
```javascript
// 关闭面板时清理事件监听
function closePanel() {
    if (panelElement) {
        // 移除所有事件监听器
        panelElement.remove();
        panelElement = null;
    }
    // 重置状态
    state.icons = [];
    state.currentPage = 1;
    state.isOpen = false;
}
```

---

## 11. 安全考虑

### 11.1 XSS 防护

```javascript
// 转义用户输入
function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

// 使用 textContent 而非 innerHTML 插入用户内容
element.textContent = userContent;
```

### 11.2 Cookie 安全

- Cookie 仅存储在用户本地 Excalidraw 设置中
- 不传输到任何第三方服务器
- 在 Cookie 输入提示中告知用户隐私影响

### 11.3 API 请求

- 仅请求必要的字段
- 不缓存敏感数据
- 请求失败不泄露用户信息

---

## 12. 可访问性

### 12.1 键盘导航

| 操作 | 快捷键 |
|------|--------|
| 打开搜索面板 | 用户自定义快捷键 |
| 聚焦搜索框 | 面板打开后自动聚焦 |
| 取消放置模式 | ESC |
| 关闭面板 | ESC (非放置模式时) |

### 12.2 屏幕阅读器

```html
<!-- 添加 ARIA 标签 -->
<div class="icon-search-panel" role="dialog" aria-label="图标搜索面板">
    <input type="text" class="search-input" aria-label="搜索图标">
    <div class="icons-grid" role="list" aria-label="搜索结果">
        <div class="icon-item" role="listitem" aria-label="微博">
            <!-- SVG -->
        </div>
    </div>
</div>
```

---

## 13. 测试计划

### 13.1 功能测试

| 测试用例 | 步骤 | 预期结果 |
|---------|------|---------|
| 首次使用 | 1. 清空设置 2. 运行脚本 | 提示输入 Cookie |
| Cookie 验证 | 1. 输入有效 Cookie 2. 搜索 | 成功返回结果 |
| Cookie 失效 | 1. 输入无效 Cookie 2. 搜索 | Notice 提示 Cookie 失效 |
| 搜索功能 | 1. 输入关键词 2. 等待 | 显示匹配图标 |
| 空搜索 | 1. 输入不存在的关键词 | 显示"未找到" |
| 滚动加载 | 1. 搜索有大量结果 2. 滚动到底部 | 自动加载下一页 |
| 图标选择 | 1. 点击图标 2. 点击画布 | 图标插入到画布 |
| 取消放置 | 1. 点击图标 2. 按 ESC | 取消插入 |
| 搜索历史 | 1. 搜索"微信" 2. 关闭 3. 重新打开 | 搜索框显示"微信" |

### 13.2 边界测试

| 测试用例 | 步骤 | 预期结果 |
|---------|------|---------|
| 空关键词 | 搜索框留空后搜索 | 不执行搜索或提示 |
| 特殊字符 | 输入特殊字符搜索 | 正常执行或提示 |
| 超长结果 | 搜索结果超过100个 | 显示前100个，支持滚动加载 |
| 快速输入 | 快速连续输入不同关键词 | 防抖生效，仅最后一次生效 |
| 重复点击 | 快速多次点击同一图标 | 仅触发一次选择 |

### 13.3 性能测试

| 测试用例 | 测量指标 | 目标 |
|---------|---------|------|
| 搜索响应 | API 调用时间 | < 2s |
| 面板渲染 | 100个图标渲染时间 | < 500ms |
| 内存占用 | 脚本运行时内存 | < 10MB |

### 13.4 兼容性测试

| 环境 | 版本 | 状态 |
|------|------|------|
| Obsidian | 1.5.7+ | ✅ |
| Excalidraw | 1.5.21+ | ✅ |
| 操作系统 | Windows/macOS/Linux | ✅ |
| 浏览器 | Electron (Obsidian 内置) | ✅ |

---

## 14. 部署与安装

### 14.1 安装方式

**方式一：通过 Excalidraw 脚本库安装**
1. 打开 Excalidraw 画布
2. 点击面板菜单 → "Install or update Excalidraw Scripts"
3. 找到 "语雀图标搜索" 或搜索 "iconfont"
4. 点击 "Install this script"
5. 重启 Obsidian

**方式二：手动安装**
1. 将 `iconfont-obsidian-search.md` 复制到 vault 的 `Excalidraw/Scripts/Downloaded/` 目录
2. 重启 Obsidian
3. 在 Excalidraw 设置中启用脚本

### 14.2 首次配置

1. 运行脚本
2. 按照提示输入语雀 Cookie
3. 获取 Cookie 方法：
   - 在浏览器登录 https://www.yuque.com
   - 按 F12 打开开发者工具
   - Application → Cookies → https://www.yuque.com
   - 复制所有 Cookie (格式: `name=value; name2=value2`)

### 14.3 快捷键设置

1. 打开 Obsidian 设置
2. 找到 "Excalidraw" → "Script Commands"
3. 找到 "语雀图标搜索"
4. 点击右侧设置快捷键 (如 `Ctrl+Shift+I`)

---

## 15. 未来扩展

### 15.1 潜在功能

| 功能 | 描述 | 优先级 |
|------|------|--------|
| 多图标源 | 支持其他图标库 (Font Awesome, Material Icons) | P2 |
| 图标收藏 | 收藏常用图标 | P2 |
| 批量插入 | 一次选择多个图标 | P3 |
| 图标编辑 | 插入后可修改颜色、大小 | P3 |
| 离线模式 | 缓存常用图标供离线使用 | P3 |
| 预览跟随 | 放置模式时图标跟随鼠标 | P3 |

### 15.2 技术债务

| 项目 | 描述 | 计划 |
|------|------|------|
| 虚拟滚动 | 当结果非常多时优化性能 | V1.1 |
| 单元测试 | 添加自动化测试 | V1.1 |
| 国际化 | 支持英文界面 | V1.2 |

---

## 16. 附录

### 16.1 术语表

| 术语 | 定义 |
|------|------|
| Excalidraw | Obsidian 的手绘风格绘图插件 |
| Excalidraw Automate | Excalidraw 的脚本自动化引擎 |
| ea | Excalidraw Automate 的全局对象 |
| Cookie | 浏览器存储的认证信息 |
| SVG | 可缩放矢量图形格式 |

### 16.2 参考资料

- [Excalidraw Automate 文档](https://zsviczian.github.io/obsidian-excalidraw-plugin/ExcalidrawScriptsEngine.html)
- [语雀 iconfont API](https://www.yuque.com/api/editor/iconfont)
- [Obsidian 插件开发文档](https://docs.obsidian.md/Plugins/Getting+started/Build+a+plugin)

### 16.3 变更历史

| 版本 | 日期 | 变更内容 | 作者 |
|------|------|---------|------|
| 1.0.0 | 2026-03-17 | 初始设计文档 | Claude Code |

---

**文档状态**: ✅ 设计完成，待审查
