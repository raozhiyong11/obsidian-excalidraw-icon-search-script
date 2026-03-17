/*
![](data:image/svg+xml,<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.35-4.35"/><path d="M11 8a3 3 0 0 0-3 3"/></svg>)

语雀图标搜索 - 在 Excalidraw 中快速搜索语雀图标库并插入到画布

```javascript
*/

// ===== 全局常量 =====
const SCRIPT_NAME = "iconfont-obsidian-search";
const SCRIPT_VERSION = "1.0.0";

// ===== 默认设置 =====
const defaultSettings = {
    // 语雀 Cookie (多行字符串)
    yuqueCookie: "",
    // 上一次的搜索关键词
    lastQuery: ""
};

// ===== 脚本设置（持久化） =====
let settings = { ...defaultSettings };

// ===== 运行时状态（内存） =====
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

// ===== 全局变量 =====
let panelElement = null;         // 面板 DOM 元素
let placingModeCleanup = null;   // 放置模式清理函数
let searchDebounceTimer = null;  // 搜索防抖定时器

// ===== CSS 样式 =====
const styleElement = document.createElement('style');
styleElement.textContent = `
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
`;
document.head.appendChild(styleElement);

// ===== 脚本入口函数 =====
async function main() {
    // 1. 版本检查
    if(!ea.verifyMinimumPluginVersion || !ea.verifyMinimumPluginVersion("1.5.21")) {
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

// ===== 设置管理模块 =====

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
    // 使用简单的 prompt 作为临时方案
    // TODO: 后续可以改进为自定义模态框
    const cookie = prompt(
        "请输入语雀 Cookie\n\n" +
        "获取方法:\n" +
        "1. 在浏览器登录 https://www.yuque.com\n" +
        "2. 按 F12 打开开发者工具\n" +
        "3. 切换到 Application/应用 标签\n" +
        "4. 左侧找到 Cookies → https://www.yuque.com\n" +
        "5. 复制所有 Cookie 值（格式: name=value; name2=value2; ...）",
        settings.yuqueCookie || ""
    );

    if (cookie && cookie.trim()) {
        settings.yuqueCookie = cookie.trim();
        saveSettings();
        new Notice("Cookie 已保存");
    } else if (!settings.yuqueCookie) {
        new Notice("未配置 Cookie，搜索功能可能无法使用");
    }
}

// ===== API 调用模块 =====

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
 * @param {Error} error - 错误对象
 */
function handleApiError(error) {
    if (error.message === "UNAUTHORIZED") {
        new Notice("语雀 Cookie 失效了，请重新获取后更新配置，再继续使用。");
    } else {
        new Notice(`搜索失败: ${error.message}`);
    }
}

// ===== UI 模块 - 面板创建 =====

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

/**
 * 绑定面板事件
 */
function bindPanelEvents() {
    // 搜索输入（防抖）
    const searchInput = panelElement.querySelector('.search-input');
    searchInput.addEventListener('input', (e) => {
        clearTimeout(searchDebounceTimer);
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

/**
 * 执行搜索
 * @param {string} query - 搜索关键词
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
 * 更新状态文本
 * @param {string} text - 状态文本
 */
function updateStatus(text) {
    const statusEl = panelElement.querySelector('.status-text');
    if (statusEl) {
        statusEl.textContent = text;
    }
}

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

        // 移除面板 DOM
        panelElement.remove();
        panelElement = null;
    }

    // 清除防抖定时器
    if (searchDebounceTimer) {
        clearTimeout(searchDebounceTimer);
        searchDebounceTimer = null;
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

/**
 * 清理 SVG 字符串（防止 XSS）
 * @param {string} svgString - 原始 SVG
 * @returns {string} 清理后的 SVG
 */
function sanitizeSvg(svgString) {
    // 创建临时 DOM 解析 SVG
    const parser = new DOMParser();
    const doc = parser.parseFromString(svgString, 'image/svg+xml');

    // 移除潜在的脚本标签
    const scripts = doc.querySelectorAll('script');
    scripts.forEach(script => script.remove());

    // 移除事件处理器属性
    const allElements = doc.querySelectorAll('*');
    allElements.forEach(el => {
        const attributes = el.attributes;
        for (let i = attributes.length - 1; i >= 0; i--) {
            const attr = attributes[i];
            if (attr.name.startsWith('on')) {
                el.removeAttribute(attr.name);
            }
        }
    });

    return doc.documentElement.outerHTML;
}

/**
 * 渲染图标到宫格
 * @param {Array} icons - 图标列表
 * @param {boolean} append - 是否追加到现有内容
 */
function renderIcons(icons, append = false) {
    const grid = panelElement.querySelector('.icons-grid');
    if (!grid) return;

    if (!append) {
        grid.innerHTML = '';
        // 使用事件委托，不在每个图标上绑定事件
        grid._iconClickHandler = null;
    }

    // 事件委托处理器
    if (!grid._iconClickHandler) {
        grid._iconClickHandler = (e) => {
            const iconItem = e.target.closest('.icon-item');
            if (iconItem) {
                const iconDataStr = iconItem.dataset.iconData;
                if (iconDataStr) {
                    try {
                        const icon = JSON.parse(iconDataStr);
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
        iconEl.title = icon.name;

        // 安全处理 SVG
        const safeSvg = sanitizeSvg(icon.show_svg);
        iconEl.innerHTML = safeSvg;

        iconEl.dataset.iconData = JSON.stringify(icon);
        fragment.appendChild(iconEl);
    });

    grid.appendChild(fragment);
}

// ===== 图标选择与插入模块 =====

/**
 * 选择图标，进入放置模式
 * @param {Object} icon - 选中的图标
 */
function selectIcon(icon) {
    // 保存搜索关键词
    const searchInput = panelElement.querySelector('.search-input');
    if (searchInput) {
        settings.lastQuery = searchInput.value;
        saveSettings();
    }

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

    // 画布点击处理 (使用 capture 阶段)
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

/**
 * 退出放置模式（供外部调用）
 */
function exitPlacingMode() {
    if (placingModeCleanup) {
        placingModeCleanup();
    }
}

/**
 * 获取画布坐标
 * @param {MouseEvent} event - 鼠标事件
 * @returns {Object} 画布坐标 {x, y}
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
 * @param {Object} point - 插入位置 {x, y}
 */
async function insertIconToCanvas(point) {
    const icon = state.selectedIcon;
    if (!icon) return;

    const size = 32; // 32x32 像素

    try {
        // 使用清理后的 SVG
        const safeSvg = sanitizeSvg(icon.show_svg);

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
        new Notice(`插入图标失败: ${error.message}`);
        console.error('Icon insertion error:', error);
    }
}

// ===== 工具函数模块 =====

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

// ===== 执行主函数 =====
await main();
