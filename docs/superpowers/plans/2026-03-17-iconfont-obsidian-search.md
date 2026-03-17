# Iconfont Obsidian Search Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建一个 Excalidraw Automate 脚本，允许用户搜索语雀图标库并将图标插入到 Excalidraw 画布中

**Architecture:** 使用 Excalidraw Automate Script Engine，通过原生 JavaScript 实现悬浮搜索面板、API 调用、图标渲染和画布插入功能

**Tech Stack:** 原生 JavaScript (ES6+), Excalidraw Automate API, CSS Grid, Obsidian requestUrl

---

## File Structure

```
Excalidraw/Scripts/Downloaded/
└── iconfont-obsidian-search.md    ← 单个脚本文件包含所有代码
    ├── 脚本元数据（YAML frontmatter）
    ├── 脚本描述和图标
    ├── CSS 样式（<style> 标签）
    └── JavaScript 代码（<script> 标签）
        ├── 全局变量和状态
        ├── 初始化和设置管理
        ├── API 调用模块
        ├── UI 创建和事件绑定
        ├── 图标渲染
        ├── 放置模式
        └── 工具函数
```

---

## Chunk 1: 脚本基础结构和元数据

### Task 1: 创建脚本文件结构

**Files:**
- Create: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 创建脚本文件并添加元数据**

```markdown
/*
![](data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.35-4.35"/><path d="M11 8a3 3 0 0 0-3 3"/></svg>)

语雀图标搜索 - 在 Excalidraw 中快速搜索语雀图标库并插入到画布

```javascript
*/
```

- [ ] **Step 2: 添加 CSS 样式**

在元数据后添加 `<style>` 标签：

```javascript
/*
<style>
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

.panel-body {
    padding: 0;
    overflow-y: auto;
    flex: 1;
}

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

.icons-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(32px, 1fr));
    gap: 8px;
    padding: 0 16px 16px;
    max-height: 350px;
    overflow-y: auto;
}

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

.status-text {
    text-align: center;
    padding: 12px;
    color: var(--text-muted);
    font-size: 12px;
}

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
</style>
*/
```

- [ ] **Step 3: 开始 JavaScript 代码部分**

添加全局变量和状态：

```javascript
// 脚本元数据
const SCRIPT_NAME = "语雀图标搜索";
const SCRIPT_VERSION = "1.0.0";

// 默认设置
const defaultSettings = {
    yuqueCookie: "",
    lastQuery: ""
};

// 运行时设置
let settings = { ...defaultSettings };

// 运行时状态
const state = {
    isOpen: false,
    isLoading: false,
    icons: [],
    currentPage: 1,
    hasMore: true,
    selectedIcon: null,
    isPlacingMode: false
};

// 面板 DOM 元素
let panelElement = null;

// 放置模式清理函数
let placingModeCleanup = null;

// 防抖定时器
let searchDebounceTimer = null;
```

- [ ] **Step 4: 提交基础结构**

```bash
git add Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "feat: 创建脚本基础结构和样式"
```

---

### Task 2: 实现设置管理模块

**Files:**
- Modify: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 添加加载设置函数**

```javascript
/**
 * 从 Excalidraw 加载设置
 */
function loadSettings() {
    try {
        const saved = ea.getScriptSettings();
        settings = { ...defaultSettings, ...saved };
    } catch (error) {
        console.error('Failed to load settings:', error);
        settings = { ...defaultSettings };
    }
}
```

- [ ] **Step 2: 添加保存设置函数**

```javascript
/**
 * 保存设置到 Excalidraw
 */
function saveSettings() {
    try {
        ea.setScriptSettings(settings);
    } catch (error) {
        console.error('Failed to save settings:', error);
        new Notice("保存设置失败");
    }
}
```

- [ ] **Step 3: 添加 Cookie 提示函数**

```javascript
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

    if (cookie && cookie.trim()) {
        settings.yuqueCookie = cookie.trim();
        saveSettings();
        new Notice("Cookie 已保存");
        return true;
    }
    return false;
}
```

- [ ] **Step 4: 提交设置管理模块**

```bash
git add Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "feat: 添加设置管理模块"
```

---

### Task 3: 实现 API 调用模块

**Files:**
- Modify: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 添加搜索图标函数**

```javascript
/**
 * 搜索语雀图标
 * @param {string} query - 搜索关键词
 * @param {number} page - 页码
 * @param {number} pageSize - 每页数量
 * @returns {Promise<Array>} 图标列表
 */
async function searchIcons(query, page = 1, pageSize = 100) {
    if (!query || !query.trim()) return [];

    try {
        const response = await requestUrl({
            url: 'https://www.yuque.com/api/editor/iconfont',
            method: 'GET',
            headers: {
                'Cookie': settings.yuqueCookie,
                'User-Agent': 'Obsidian Excalidraw IconSearch/1.0'
            },
            query: {
                query: query.trim(),
                page_size: pageSize,
                page: page
            }
        });

        // 认证检查
        if (response.status === 401) {
            throw new Error("UNAUTHORIZED");
        }

        const data = response.json;

        if (!data || !data.data || !Array.isArray(data.data.icons)) {
            console.error('Unexpected API response:', data);
            return [];
        }

        state.hasMore = data.data.icons.length === pageSize;
        return data.data.icons;

    } catch (error) {
        handleApiError(error);
        return [];
    }
}
```

- [ ] **Step 2: 添加错误处理函数**

```javascript
/**
 * 处理 API 错误
 * @param {Error} error - 错误对象
 */
function handleApiError(error) {
    if (error.message === "UNAUTHORIZED") {
        new Notice("语雀 Cookie 失效了，请重新获取后更新配置，再继续使用。");
    } else {
        console.error('API Error:', error);
        new Notice(`搜索失败: ${error.message}`);
    }
}
```

- [ ] **Step 3: 提交 API 模块**

```bash
git add Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "feat: 添加 API 调用模块"
```

---

### Task 4: 实现工具函数模块

**Files:**
- Modify: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 添加 SVG 清理函数（安全重要）**

```javascript
/**
 * 清理 SVG 字符串防止 XSS 攻击
 * @param {string} svgString - 原始 SVG
 * @returns {string} 清理后的 SVG
 */
function sanitizeSvg(svgString) {
    try {
        // 创建临时 DOM 解析 SVG
        const parser = new DOMParser();
        const doc = parser.parseFromString(svgString, 'image/svg+xml');

        // 移除危险的标签
        const dangerousTags = ['script', 'iframe', 'object', 'embed', 'form'];
        dangerousTags.forEach(tag => {
            const elements = doc.querySelectorAll(tag);
            elements.forEach(el => el.remove());
        });

        // 移除所有事件处理器属性
        const allElements = doc.querySelectorAll('*');
        allElements.forEach(el => {
            const attributes = el.attributes;
            for (let i = attributes.length - 1; i >= 0; i--) {
                const attr = attributes[i];
                // 移除 on* 事件属性
                if (attr.name.toLowerCase().startsWith('on')) {
                    el.removeAttribute(attr.name);
                }
                // 移除 javascript: 协议的 href
                if (attr.name.toLowerCase() === 'href' &&
                    attr.value.toLowerCase().startsWith('javascript:')) {
                    el.removeAttribute(attr.name);
                }
            }
        });

        return doc.documentElement.outerHTML;
    } catch (error) {
        console.error('SVG sanitization error:', error);
        // 如果清理失败，返回空字符串以防止安全问题
        return '';
    }
}
```

- [ ] **Step 2: 添加防抖工具函数**

```javascript
/**
 * 防抖函数
 * @param {Function} func - 要防抖的函数
 * @param {number} wait - 等待时间（毫秒）
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
```

- [ ] **Step 3: 提交工具函数模块**

```bash
git add Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "feat: 添加工具函数模块（含 SVG 安全清理）"
```

---

### Task 5: 实现 UI 创建模块

**Files:**
- Modify: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 添加创建面板 DOM 函数**

```javascript
/**
 * 创建面板 DOM 结构
 * @returns {HTMLElement} 面板元素
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
```

- [ ] **Step 2: 添加打开面板函数**

```javascript
/**
 * 打开搜索面板
 */
function openSearchPanel() {
    // 防止重复打开
    if (state.isOpen) {
        // 如果面板存在但不在 DOM 中，重新添加
        if (panelElement && !document.body.contains(panelElement)) {
            document.body.appendChild(panelElement);
        }
        return;
    }

    // 清理可能存在的旧面板
    if (panelElement) {
        panelElement.remove();
    }

    // 创建新面板
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
```

- [ ] **Step 3: 添加关闭面板函数**

```javascript
/**
 * 关闭面板并清理资源
 */
function closePanel() {
    if (panelElement) {
        // 移除事件委托处理器
        const grid = panelElement.querySelector('.icons-grid');
        if (grid && grid._iconClickHandler) {
            grid.removeEventListener('click', grid._iconClickHandler);
            grid._iconClickHandler = null;
        }

        // 清理防抖定时器
        if (searchDebounceTimer) {
            clearTimeout(searchDebounceTimer);
            searchDebounceTimer = null;
        }

        // 移除面板 DOM
        panelElement.remove();
        panelElement = null;
    }

    // 重置所有状态
    state.isOpen = false;
    state.isLoading = false;
    state.icons = [];
    state.currentPage = 1;
    state.selectedIcon = null;

    // 如果在放置模式下关闭面板，退出放置模式
    if (state.isPlacingMode) {
        exitPlacingMode();
    }
}
```

- [ ] **Step 4: 提交 UI 创建模块**

```bash
git add Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "feat: 添加 UI 创建模块"
```

---

### Task 6: 实现事件绑定模块

**Files:**
- Modify: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 添加面板事件绑定函数**

```javascript
/**
 * 绑定面板事件
 */
function bindPanelEvents() {
    if (!panelElement) return;

    // 搜索输入（防抖）
    const searchInput = panelElement.querySelector('.search-input');
    searchInput.addEventListener('input', (e) => {
        if (searchDebounceTimer) {
            clearTimeout(searchDebounceTimer);
        }
        searchDebounceTimer = setTimeout(() => {
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
```

- [ ] **Step 2: 添加执行搜索函数**

```javascript
/**
 * 执行搜索
 * @param {string} query - 搜索关键词
 */
async function performSearch(query) {
    if (state.isLoading) return;

    // 保存当前搜索关键词
    settings.lastQuery = query;
    saveSettings();

    state.isLoading = true;
    updateStatus("加载中...");

    const icons = await searchIcons(query, 1);
    state.icons = icons;
    state.currentPage = 1;

    renderIcons(icons, false);
    updateStatus(icons.length === 0 ? "未找到相关图标" : "");

    state.isLoading = false;
}
```

- [ ] **Step 3: 添加加载更多函数**

```javascript
/**
 * 加载更多图标
 */
async function loadMoreIcons() {
    if (state.isLoading || !state.hasMore) return;

    state.isLoading = true;
    updateStatus("加载更多...");

    const searchInput = panelElement.querySelector('.search-input');
    const newIcons = await searchIcons(searchInput.value, state.currentPage + 1);

    if (newIcons.length > 0) {
        state.icons.push(...newIcons);
        state.currentPage++;
        renderIcons(newIcons, true);
    }

    updateStatus("");
    state.isLoading = false;
}
```

- [ ] **Step 4: 添加更新状态文本函数**

```javascript
/**
 * 更新状态文本
 * @param {string} text - 状态文本
 */
function updateStatus(text) {
    if (!panelElement) return;
    const statusEl = panelElement.querySelector('.status-text');
    if (statusEl) {
        statusEl.textContent = text;
    }
}
```

- [ ] **Step 5: 提交事件绑定模块**

```bash
git add Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "feat: 添加事件绑定模块"
```

---

### Task 7: 实现图标渲染模块

**Files:**
- Modify: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 添加渲染图标函数**

```javascript
/**
 * 渲染图标到宫格
 * @param {Array} icons - 图标列表
 * @param {boolean} append - 是否追加到现有内容
 */
function renderIcons(icons, append = false) {
    if (!panelElement) return;

    const grid = panelElement.querySelector('.icons-grid');
    if (!grid) return;

    if (!append) {
        grid.innerHTML = '';
        // 重置事件委托处理器
        grid._iconClickHandler = null;
    }

    // 设置事件委托处理器
    if (!grid._iconClickHandler) {
        grid._iconClickHandler = (e) => {
            const iconItem = e.target.closest('.icon-item');
            if (iconItem) {
                const iconData = iconItem.dataset.iconData;
                if (iconData) {
                    try {
                        const icon = JSON.parse(iconData);
                        selectIcon(icon);
                    } catch (err) {
                        console.error('Failed to parse icon data:', err);
                    }
                }
            }
        };
        grid.addEventListener('click', grid._iconClickHandler);
    }

    // 使用 DocumentFragment 批量插入
    const fragment = document.createDocumentFragment();

    icons.forEach(icon => {
        const iconEl = document.createElement('div');
        iconEl.className = 'icon-item';
        iconEl.title = icon.name || '未知图标';

        // 安全处理 SVG
        const safeSvg = sanitizeSvg(icon.show_svg);
        if (safeSvg) {
            iconEl.innerHTML = safeSvg;
            iconEl.dataset.iconData = JSON.stringify(icon);
            fragment.appendChild(iconEl);
        }
    });

    grid.appendChild(fragment);
}
```

- [ ] **Step 2: 添加选择图标函数**

```javascript
/**
 * 选择图标，进入放置模式
 * @param {Object} icon - 选中的图标
 */
function selectIcon(icon) {
    if (!icon) return;

    // 保存搜索关键词
    if (panelElement) {
        const searchInput = panelElement.querySelector('.search-input');
        if (searchInput && searchInput.value) {
            settings.lastQuery = searchInput.value;
            saveSettings();
        }
    }

    // 设置状态
    state.selectedIcon = icon;

    // 关闭面板
    closePanel();

    // 进入放置模式
    enterPlacingMode();
}
```

- [ ] **Step 3: 提交图标渲染模块**

```bash
git add Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "feat: 添加图标渲染模块"
```

---

### Task 8: 实现放置模式模块

**Files:**
- Modify: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 添加进入放置模式函数**

```javascript
/**
 * 进入放置模式
 */
function enterPlacingMode() {
    if (!state.selectedIcon) {
        new Notice("没有选中的图标");
        return;
    }

    // 更改鼠标样式
    document.body.style.cursor = 'crosshair';

    // 显示提示
    const tip = new Notice("点击画布位置插入图标 (按 ESC 或右键取消)", 0);

    // 获取画布容器
    const view = ea.targetView;
    if (!view || !view.containerEl) {
        new Notice("无法获取画布引用");
        return;
    }

    // 清理函数
    const cleanup = () => {
        document.removeEventListener('click', handleCanvasClick, { capture: true });
        document.removeEventListener('keydown', handleEscape);
        document.removeEventListener('contextmenu', handleContextMenu);
        document.body.style.cursor = '';
        tip.hide();
        state.isPlacingMode = false;
        state.selectedIcon = null;
        placingModeCleanup = null;
    };

    // 保存清理函数供外部调用
    placingModeCleanup = cleanup;

    // 画布点击处理（使用 capture 阶段）
    const handleCanvasClick = async (e) => {
        // 只响应左键点击
        if (e.button !== 0) return;

        // 确保点击的是画布区域内的实际画布元素
        const excalidrawContainer = view.containerEl.querySelector('.excalidraw-container');
        if (!excalidrawContainer || !excalidrawContainer.contains(e.target)) {
            return;
        }

        // 阻止事件继续传播
        e.stopPropagation();
        e.stopImmediatePropagation();

        const point = getCanvasCoordinates(e);
        await insertIconToCanvas(point);

        cleanup();
    };

    // ESC 取消处理
    const handleEscape = (e) => {
        if (e.key === 'Escape') {
            e.preventDefault();
            cleanup();
        }
    };

    // 右键取消
    const handleContextMenu = (e) => {
        e.preventDefault();
        cleanup();
    };

    // 使用 capture 阶段确保优先处理
    document.addEventListener('click', handleCanvasClick, { capture: true });
    document.addEventListener('keydown', handleEscape);
    document.addEventListener('contextmenu', handleContextMenu);

    state.isPlacingMode = true;
}
```

- [ ] **Step 2: 添加退出放置模式函数**

```javascript
/**
 * 退出放置模式（供外部调用）
 */
function exitPlacingMode() {
    if (placingModeCleanup) {
        placingModeCleanup();
        placingModeCleanup = null;
    }
}
```

- [ ] **Step 3: 添加获取画布坐标函数**

```javascript
/**
 * 获取画布坐标
 * @param {MouseEvent} event - 鼠标事件
 * @returns {Object} 画布坐标 {x, y}
 */
function getCanvasCoordinates(event) {
    const view = ea.targetView;
    if (!view || !view.containerEl) {
        return { x: event.clientX, y: event.clientY };
    }

    const rect = view.containerEl.getBoundingClientRect();
    return {
        x: event.clientX - rect.left,
        y: event.clientY - rect.top
    };
}
```

- [ ] **Step 4: 添加插入图标到画布函数**

```javascript
/**
 * 将图标插入到画布
 * @param {Object} point - 插入位置 {x, y}
 */
async function insertIconToCanvas(point) {
    const icon = state.selectedIcon;
    if (!icon) return;

    const size = 32; // 32x32 像素

    try {
        // 使用清理后的 SVG
        const safeSvg = sanitizeSvg(icon.show_svg);
        if (!safeSvg) {
            throw new Error("SVG 清理失败");
        }

        // 使用 Excalidraw API 转换 SVG
        const elements = await ea.svgToExcalidrawElements(safeSvg, {
            x: point.x - size / 2,
            y: point.y - size / 2,
            width: size,
            height: size,
            strokeColor: '#000000',
            fillColor: 'transparent'
        });

        if (!elements || elements.length === 0) {
            throw new Error("SVG 转换失败");
        }

        ea.elements = elements;
        await ea.addElementsToView(false, true, true);
        ea.selectElementsInView(ea.getElements());

        new Notice("图标已插入");

    } catch (error) {
        console.error('Icon insertion error:', error);
        new Notice(`插入图标失败: ${error.message}`);
    }
}
```

- [ ] **Step 5: 提交放置模式模块**

```bash
git add Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "feat: 添加放置模式模块"
```

---

### Task 9: 实现主入口函数

**Files:**
- Modify: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 添加主入口函数**

```javascript
/**
 * 脚本主入口函数
 */
async function main() {
    // 1. 版本检查
    if (!ea.verifyMinimumPluginVersion || !ea.verifyMinimumPluginVersion("1.5.21")) {
        new Notice("此脚本需要 Excalidraw 1.5.21 或更高版本");
        return;
    }

    // 2. 加载设置
    loadSettings();

    // 3. Cookie 检查
    if (!settings.yuqueCookie) {
        const hasCookie = await promptForCookie();
        if (!hasCookie) {
            new Notice("需要配置语雀 Cookie 才能使用此脚本");
            return;
        }
    }

    // 4. 打开搜索面板
    openSearchPanel();
}
```

- [ ] **Step 2: 调用主函数**

```javascript
// 执行主函数
main();
```

- [ ] **Step 3: 提交主入口函数**

```bash
git add Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "feat: 添加主入口函数"
```

---

### Task 10: 最终测试和文档

**Files:**
- Modify: `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

- [ ] **Step 1: 完整性检查**

检查脚本是否包含：
- ✅ 脚本元数据（图标、描述）
- ✅ CSS 样式（`<style>` 标签）
- ✅ 所有必需的函数
- ✅ 全局变量定义
- ✅ 主入口函数调用

- [ ] **Step 2: 添加使用说明注释**

在脚本开头添加详细的使用说明：

```markdown
/*
## 使用说明

### 安装
1. 将此文件复制到 vault 的 `Excalidraw/Scripts/Downloaded/` 目录
2. 重启 Obsidian
3. 在 Excalidraw 设置中启用脚本

### 配置
首次运行时需要输入语雀 Cookie：
1. 在浏览器登录 https://www.yuque.com
2. 按 F12 打开开发者工具
3. Application → Cookies → https://www.yuque.com
4. 复制所有 Cookie（格式: name=value; name2=value2）

### 使用
1. 在 Excalidraw 画布中运行此脚本（工具栏图标或快捷键）
2. 输入搜索关键词
3. 点击选中的图标
4. 点击画布位置插入图标
5. 按 ESC 或右键取消

### 快捷键设置
1. 打开 Obsidian 设置
2. 找到 "Excalidraw" → "Script Commands"
3. 找到 "语雀图标搜索" 并设置快捷键

```javascript
*/
```

- [ ] **Step 3: 创建 README 文档**

创建 `README.md` 文件：

```markdown
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
```

- [ ] **Step 4: 最终提交**

```bash
git add README.md Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md
git commit -m "docs: 添加使用说明和 README 文档"
```

---

## Testing Instructions

### 手动测试清单

- [ ] **首次使用**
  - [ ] 运行脚本，提示输入 Cookie
  - [ ] 输入有效 Cookie，保存成功
  - [ ] 搜索面板打开

- [ ] **搜索功能**
  - [ ] 输入"微信"，显示相关图标
  - [ ] 输入不存在的关键词，显示"未找到"
  - [ ] 清空搜索框，不执行搜索

- [ ] **滚动加载**
  - [ ] 搜索结果超过100个
  - [ ] 滚动到底部，自动加载下一页
  - [ ] 新图标追加到现有列表

- [ ] **图标选择和插入**
  - [ ] 点击图标，面板关闭
  - [ ] 鼠标变为十字光标
  - [ ] 点击画布，图标插入
  - [ ] 图标大小为 32x32

- [ ] **取消操作**
  - [ ] 按 ESC，退出放置模式
  - [ ] 右键点击，退出放置模式

- [ ] **Cookie 失效**
  - [ ] 使用无效 Cookie
  - [ ] 搜索时提示"Cookie 失效"

- [ ] **搜索历史**
  - [ ] 搜索"微博"
  - [ ] 关闭面板
  - [ ] 重新打开，搜索框显示"微博"

- [ ] **重复打开**
  - [ ] 面板已打开时再次运行脚本
  - [ ] 不创建重复面板

### 边界测试

- [ ] 空字符串搜索
- [ ] 特殊字符搜索
- [ ] 超长搜索关键词
- [ ] 快速连续输入不同关键词
- [ ] 网络断开时搜索

---

## Notes for Implementation

1. **重要**: SVG 清理函数是安全关键，必须正确实现
2. **事件委托**: 使用事件委托而不是在每个图标上绑定事件
3. **状态管理**: 确保所有状态在面板关闭时正确清理
4. **错误处理**: 所有异步操作都要有 try-catch
5. **防抖**: 搜索输入必须防抖，避免频繁 API 调用
6. **坐标计算**: 插入位置要减去图标大小的一半，使图标居中

---

## Completion Criteria

- [ ] 所有任务完成
- [ ] 手动测试清单全部通过
- [ ] 无控制台错误
- [ ] Cookie 安全存储
- [ ] SVG 正确清理
- [ ] 图标成功插入画布
- [ ] 所有交互正常工作
