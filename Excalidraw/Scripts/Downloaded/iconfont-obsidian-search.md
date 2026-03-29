/*
![](data:image/svg+xml;base64,PHN2ZyB4bWxucz0nJ2h0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnJycgdmlld0JveD0nJzAgMCAyNCAyNCcnIGZpbGw9Jydub25lJycgc3Ryb2tlPScnY3VycmVudENvbG9yJycgc3Ryb2tlLXdpZHRoPScnMicnIHN0cm9rZS1saW5lY2FwPScncm91bmQnJyBzdHJva2UtbGluZWpvaW49Jydyb3VuZCcnPjxyZWN0IHg9JyczJycgeT0nJzUnJyB3aWR0aD0nJzE4JycgaGVpZ2h0PScnMTQnJyByeD0nJzInJy8+PGNpcmNsZSBjeD0nJzknJyBjeT0nJzEwJycgcj0nJzEuNScnLz48cGF0aCBkPScnTTIxIDE2bC01LjUtNS41TDcgMTknJy8+PC9zdmc+)

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
    isPlacingMode: false,   // 是否处于放置模式
    isInsertingIcon: false  // 是否正在插入图标
};

// ===== 全局变量 =====
let panelElement = null;         // 面板 DOM 元素
let placingModeCleanup = null;   // 放置模式清理函数
let searchDebounceTimer = null;  // 搜索防抖定时器
let app = null;                  // Obsidian app 实例
let vault = null;                // Obsidian vault 实例

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
    overflow: hidden;
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
    cursor: move;
    user-select: none;
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
    overflow: hidden;
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
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
    flex: 1;
    min-height: 0;
    overflow-y: auto;
    align-content: start;
}

/* ===== 单个图标项 ===== */
.icon-item {
    width: 32px;
    height: 32px;
    padding: 4px;
    cursor: pointer;
    border-radius: 4px;
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--background-primary);
    border: 1px solid transparent;
    overflow: hidden;
}

.icon-item:hover {
    background: var(--background-modifier-hover);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    border-color: var(--background-modifier-border);
}

.icon-item.is-selected {
    background: color-mix(in srgb, var(--interactive-accent) 14%, var(--background-primary));
    border-color: var(--interactive-accent);
    box-shadow: 0 0 0 1px color-mix(in srgb, var(--interactive-accent) 35%, transparent);
}

.icon-item svg {
    max-width: 22px;
    max-height: 22px;
    width: auto;
    height: auto;
    fill: currentColor;
    pointer-events: none;
    flex-shrink: 0;
}

.icon-preview-overlay {
    position: fixed;
    width: 92px;
    height: 92px;
    padding: 0;
    box-sizing: border-box;
    border-radius: 16px;
    border: 1px solid var(--background-modifier-border);
    background: color-mix(in srgb, var(--background-primary) 82%, transparent);
    box-shadow: 0 16px 34px rgba(0, 0, 0, 0.22);
    display: flex;
    align-items: center;
    justify-content: center;
    pointer-events: none;
    z-index: 10005;
    opacity: 0;
    transform: scale(0.94);
    transition: opacity 0.12s ease, transform 0.12s ease;
    backdrop-filter: blur(8px);
}

.icon-preview-overlay.is-visible {
    opacity: 1;
    transform: scale(1);
}

.icon-preview-content {
    width: 22px;
    height: 22px;
    display: flex;
    align-items: center;
    justify-content: center;
    transform: scale(3);
    transform-origin: center center;
}

.icon-preview-content svg {
    width: 22px;
    height: 22px;
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

    // 2. 尝试获取 Obsidian API（可选）
    app = ea.app;
    if (app) {
        vault = app.vault;
    }

    // 3. 加载设置
    loadSettings();

    // 4. Cookie 检查
    if (!settings.yuqueCookie) {
        await promptForCookie();
    }

    // 5. 打开搜索面板
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
    return new Promise((resolve) => {
        // 创建模态对话框
        const modal = document.createElement('div');
        modal.style.cssText = `
            position: fixed;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: rgba(0, 0, 0, 0.5);
            display: flex;
            align-items: center;
            justify-content: center;
            z-index: 10000;
        `;

        modal.innerHTML = `
            <div style="
                background: var(--background-secondary);
                border: 1px solid var(--background-modifier-border);
                border-radius: 8px;
                padding: 24px;
                max-width: 600px;
                width: 90%;
                box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
            ">
                <h2 style="margin: 0 0 16px 0; color: var(--text-normal);">配置语雀 Cookie</h2>
                <div style="color: var(--text-muted); font-size: 13px; line-height: 1.6; margin-bottom: 16px;">
                    <p style="margin: 0 0 8px 0;"><strong>获取方法：</strong></p>
                    <ol style="margin: 0; padding-left: 20px;">
                        <li>在浏览器登录 <a href="https://www.yuque.com" target="_blank">https://www.yuque.com</a></li>
                        <li>按 F12 打开开发者工具</li>
                        <li>切换到 Application/应用 标签</li>
                        <li>左侧找到 Cookies → https://www.yuque.com</li>
                        <li>复制所有 Cookie 值（格式: name=value; name2=value2; ...）</li>
                    </ol>
                </div>
                <textarea id="cookie-input" style="
                    width: 100%;
                    min-height: 100px;
                    padding: 8px;
                    border: 1px solid var(--background-modifier-border);
                    border-radius: 4px;
                    background: var(--background-primary);
                    color: var(--text-normal);
                    font-family: monospace;
                    font-size: 12px;
                    resize: vertical;
                    margin-bottom: 16px;
                " placeholder="粘贴 Cookie 内容...">${settings.yuqueCookie || ''}</textarea>
                <div style="display: flex; gap: 8px; justify-content: flex-end;">
                    <button id="cancel-btn" style="
                        padding: 8px 16px;
                        border: 1px solid var(--background-modifier-border);
                        border-radius: 4px;
                        background: var(--background-primary);
                        color: var(--text-normal);
                        cursor: pointer;
                    ">取消</button>
                    <button id="save-btn" style="
                        padding: 8px 16px;
                        border: none;
                        border-radius: 4px;
                        background: var(--interactive-accent);
                        color: var(--text-on-accent);
                        cursor: pointer;
                    ">保存</button>
                </div>
            </div>
        `;

        document.body.appendChild(modal);

        const textarea = modal.querySelector('#cookie-input');
        const cancelBtn = modal.querySelector('#cancel-btn');
        const saveBtn = modal.querySelector('#save-btn');

        cancelBtn.addEventListener('click', () => {
            modal.remove();
            if (!settings.yuqueCookie) {
                new Notice("未配置 Cookie，搜索功能可能无法使用");
            }
            resolve();
        });

        saveBtn.addEventListener('click', () => {
            const cookie = textarea.value.trim();
            if (cookie) {
                settings.yuqueCookie = cookie;
                saveSettings();
                new Notice("Cookie 已保存");
            } else if (!settings.yuqueCookie) {
                new Notice("未配置 Cookie，搜索功能可能无法使用");
            }
            modal.remove();
            resolve();
        });

        textarea.focus();
    });
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
// Override the earlier implementation because Obsidian requestUrl does not reliably apply a `query` option.
async function searchIcons(query, page = 1, pageSize = 100) {
    const normalizedQuery = (query || "").trim();
    if (!normalizedQuery) return [];

    const requestUrlWithParams = `https://www.yuque.com/api/editor/iconfont?${new URLSearchParams({
        query: normalizedQuery,
        page_size: String(pageSize),
        page: String(page)
    }).toString()}`;

    try {
        const response = await requestUrl({
            url: requestUrlWithParams,
            method: "GET",
            headers: {
                Cookie: settings.yuqueCookie,
                "User-Agent": "Obsidian Excalidraw IconSearch/1.0"
            }
        });

        if (response.status === 401) {
            throw new Error("UNAUTHORIZED");
        }

        const data = response.json || {};
        const icons = data && data.data && Array.isArray(data.data.icons) ? data.data.icons : [];

        state.hasMore = icons.length === pageSize;
        return icons;
    } catch (error) {
        handleApiError(error);
        return [];
    }
}

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

    if (state.icons && state.icons.length > 0) {
        renderIcons(state.icons);
        updateStatus("");
    }
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
            <span class="panel-title">图标搜索</span>
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

function bindPanelDragEvents() {
    const header = panelElement && panelElement.querySelector('.panel-header');
    if (!header || !panelElement) return;

    let isDragging = false;
    let startX = 0;
    let startY = 0;
    let startLeft = 0;
    let startTop = 0;

    const onMouseMove = (event) => {
        if (!isDragging) return;

        const nextLeft = startLeft + (event.clientX - startX);
        const nextTop = startTop + (event.clientY - startY);
        const maxLeft = Math.max(0, window.innerWidth - panelElement.offsetWidth);
        const maxTop = Math.max(0, window.innerHeight - panelElement.offsetHeight);

        panelElement.style.left = `${Math.min(Math.max(0, nextLeft), maxLeft)}px`;
        panelElement.style.top = `${Math.min(Math.max(0, nextTop), maxTop)}px`;
        panelElement.style.right = 'auto';
    };

    const stopDragging = () => {
        if (!isDragging) return;
        isDragging = false;
        document.removeEventListener('mousemove', onMouseMove);
        document.removeEventListener('mouseup', stopDragging);
    };

    header.addEventListener('mousedown', (event) => {
        if (event.button !== 0) return;
        if (event.target.closest('.close-btn')) return;

        const rect = panelElement.getBoundingClientRect();
        isDragging = true;
        startX = event.clientX;
        startY = event.clientY;
        startLeft = rect.left;
        startTop = rect.top;

        panelElement.style.left = `${rect.left}px`;
        panelElement.style.top = `${rect.top}px`;
        panelElement.style.right = 'auto';

        document.addEventListener('mousemove', onMouseMove);
        document.addEventListener('mouseup', stopDragging);
        event.preventDefault();
    });
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
    searchInput.addEventListener('keydown', (e) => {
        if (e.key !== 'Enter') return;
        clearTimeout(searchDebounceTimer);
        performSearch(e.target.value);
    });

    const grid = panelElement.querySelector('.icons-grid');
    grid.addEventListener('scroll', () => {
        if (!state.isLoading && state.hasMore) {
            const scrollBottom = grid.scrollTop + grid.clientHeight;
            if (scrollBottom >= grid.scrollHeight - 50) {
                loadMoreIcons();
            }
        }
    });

    // 关闭按钮
    panelElement.querySelector('.close-btn').addEventListener('click', closePanel);
    bindPanelDragEvents();
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
    hideIconPreviewOverlay();
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

    // 重置 UI 状态（保留放置模式相关的状态）
    state.isOpen = false;
    state.isLoading = false;

    // 只有在没有进入放置模式时才重置 selectedIcon
    if (!state.isPlacingMode) {
        state.selectedIcon = null;
    }
}

/**
 * 清理并优化 SVG 字符串
 * @param {string} svgString - 原始 SVG
 * @returns {string} 优化后的 SVG
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

    const svgElement = doc.documentElement;

    // 如果没有 viewBox，从 SVG 的 width/height 属性推算，保留原始宽高比
    if (!svgElement.getAttribute('viewBox')) {
        const w = parseFloat(svgElement.getAttribute('width')) || 1024;
        const h = parseFloat(svgElement.getAttribute('height')) || 1024;
        svgElement.setAttribute('viewBox', `0 0 ${w} ${h}`);
    }

    // 设置 width 和 height 为 100%，确保填充容器
    svgElement.setAttribute('width', '100%');
    svgElement.setAttribute('height', '100%');

    // 添加 preserveAspectRatio 确保图标均匀缩放
    svgElement.setAttribute('preserveAspectRatio', 'xMidYMid meet');

    return svgElement.outerHTML;
}

/**
 * 获取图标的宽高比
 * @param {Object} icon - 图标对象（需包含 show_svg 字段）
 * @returns {number} 宽高比 (width / height)，默认 1（正方形）
 */
function getIconAspectRatio(icon) {
    if (!icon || !icon.show_svg) return 1;

    try {
        const parser = new DOMParser();
        const doc = parser.parseFromString(icon.show_svg, 'image/svg+xml');
        const svg = doc.documentElement;

        // 优先从 viewBox 获取
        const viewBox = svg.getAttribute('viewBox');
        if (viewBox) {
            const parts = viewBox.trim().split(/[\s,]+/);
            if (parts.length === 4) {
                const w = parseFloat(parts[2]);
                const h = parseFloat(parts[3]);
                if (w > 0 && h > 0) return w / h;
            }
        }

        // 从 width/height 属性获取
        const w = parseFloat(svg.getAttribute('width'));
        const h = parseFloat(svg.getAttribute('height'));
        if (w > 0 && h > 0) return w / h;
    } catch (e) {
        // ignore parse errors
    }

    // 从 icon 对象的 width/height 字段获取
    if (icon.width > 0 && icon.height > 0) return icon.width / icon.height;

    return 1; // 默认正方形
}

/**
 * 根据宽高比计算图标的显示尺寸
 * 以 baseSize 为短边，按比例计算长边
 * @param {number} baseSize - 基础尺寸（短边）
 * @param {number} aspectRatio - 宽高比 (width / height)
 * @returns {{width: number, height: number}} 计算后的尺寸
 */
function calculateIconSize(baseSize, aspectRatio) {
    if (aspectRatio >= 1) {
        // 宽 >= 高，高度为 baseSize，宽度按比例
        return { width: baseSize * aspectRatio, height: baseSize };
    } else {
        // 高 > 宽，宽度为 baseSize，高度按比例
        return { width: baseSize, height: baseSize / aspectRatio };
    }
}

function ensureIconPreviewOverlay() {
    let overlay = document.querySelector('.icon-preview-overlay');
    if (overlay) return overlay;

    overlay = document.createElement('div');
    overlay.className = 'icon-preview-overlay';
    document.body.appendChild(overlay);
    return overlay;
}

function hideIconPreviewOverlay() {
    const overlay = document.querySelector('.icon-preview-overlay');
    if (!overlay) return;
    overlay.classList.remove('is-visible');
}

function showIconPreviewOverlay(iconItem) {
    if (!iconItem) return;

    const overlay = ensureIconPreviewOverlay();
    overlay.innerHTML = `<div class="icon-preview-content">${iconItem.innerHTML}</div>`;

    const rect = iconItem.getBoundingClientRect();
    const overlayWidth = 92;
    const overlayHeight = 92;
    const gap = 10;
    const preferredLeft = rect.right + gap;
    const fallbackLeft = rect.left - overlayWidth - gap;
    const left = preferredLeft + overlayWidth <= window.innerWidth - 8
        ? preferredLeft
        : Math.max(8, fallbackLeft);
    const top = Math.min(
        Math.max(8, rect.top + rect.height / 2 - overlayHeight / 2),
        window.innerHeight - overlayHeight - 8
    );

    overlay.style.left = `${left}px`;
    overlay.style.top = `${top}px`;
    overlay.classList.add('is-visible');
}

/**
 * 渲染图标到宫格
 * @param {Array} icons - 图标列表
 * @param {boolean} append - 是否追加到现有内容
 */
function renderIcons(icons, append = false) {
    const grid = panelElement.querySelector('.icons-grid');

    if (!grid) {
        console.error("[ERROR] .icons-grid element not found!");
        return;
    }

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
                        console.error('[ERROR] Failed to parse icon data:', err);
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
        iconEl.addEventListener('mouseenter', () => showIconPreviewOverlay(iconEl));
        iconEl.addEventListener('mouseleave', hideIconPreviewOverlay);

        iconEl.dataset.iconData = JSON.stringify(icon);
        fragment.appendChild(iconEl);
    });

    grid.appendChild(fragment);
    highlightSelectedIcon(state.selectedIcon);
}

function highlightSelectedIcon(icon) {
    if (!panelElement) return;

    const grid = panelElement.querySelector('.icons-grid');
    if (!grid) return;

    grid.querySelectorAll('.icon-item.is-selected').forEach((element) => {
        element.classList.remove('is-selected');
    });

    if (!icon) return;

    const target = Array.from(grid.querySelectorAll('.icon-item')).find((element) => {
        const iconDataStr = element.dataset.iconData;
        if (!iconDataStr) return false;

        try {
            const parsed = JSON.parse(iconDataStr);
            return parsed.id === icon.id;
        } catch (error) {
            return false;
        }
    });

    if (target) {
        target.classList.add('is-selected');
    }
}

// ===== 图标选择与插入模块 =====

/**
 * 选择图标，直接插入到屏幕中心
 * @param {Object} icon - 选中的图标
 */
async function selectIcon(icon) {
    if (state.isInsertingIcon) {
        return;
    }

    // 保存搜索关键词
    const searchInput = panelElement.querySelector('.search-input');
    if (searchInput) {
        settings.lastQuery = searchInput.value;
        saveSettings();
    }

    state.selectedIcon = icon;
    highlightSelectedIcon(icon);
    state.isInsertingIcon = true;

    // 保持面板打开，便于连续试多个图标
    try {
        await insertIconToScreenCenter(icon);
    } finally {
        state.isInsertingIcon = false;
    }
}

/**
 * 将图标插入到屏幕可视区域中心
 * @param {Object} icon - 要插入的图标对象
 */
async function legacyInsertIconToScreenCenter(icon) {
    if (!icon) {
        console.error("[INSERT] icon is empty");
        return;
    }

    try {
        // 使用清理后的 SVG
        const safeSvg = sanitizeSvg(icon.show_svg);

        // 获取 Excalidraw API
        const view = ea.targetView;
        const api = view.excalidrawAPI;

        if (!api) {
            throw new Error("无法获取 Excalidraw API");
        }

        const appState = api.getAppState();

        // 生成唯一的文件 ID
        const timestamp = Date.now();
        const randomId = Math.floor(Math.random() * 1000).toString().padStart(3, '0');
        const fileId = `${SCRIPT_NAME}_${icon.name.replace(/[^a-zA-Z0-9]/g, '_')}_${timestamp}_${randomId}`;

        // 将 SVG 转换为 data URL
        const base64Svg = btoa(unescape(encodeURIComponent(safeSvg)));
        const dataUrl = `data:image/svg+xml;base64,${base64Svg}`;

        // 获取当前文件系统
        const files = api.getFiles();

        // 创建新文件对象
        const newFile = {
            id: fileId,
            mimeType: "image/svg+xml",
            dataURL: dataUrl,
            size: dataUrl.length,
            created: timestamp,
            lastRetrieved: timestamp
        };

        // 注册文件
        if (typeof api.addFiles === 'function') {
            api.addFiles({ [fileId]: newFile });
        } else {
            files[fileId] = newFile;
        }

        // 获取现有元素
        const existingElements = api.getSceneElements();

        // 计算图标位置（使用现有元素作为参考）
        const displaySize = 32; // 用户看到的显示大小
        const zoom = appState.zoom.value;
        const iconSize = displaySize / zoom;

        let iconX, iconY;

        if (existingElements.length > 0) {
            // 使用最后一个元素的位置作为基准
            const lastElement = existingElements[existingElements.length - 1];
            iconX = lastElement.x;
            iconY = lastElement.y + lastElement.height + 50; // 在最后一个元素下方50px
        } else {
            // 空画布：使用画布原点附近
            iconX = 100;
            iconY = 100;
        }

        // 创建图片元素
        const elementId = `${fileId}_el`;
        const imageElement = {
            id: elementId,
            type: "image",
            x: iconX,
            y: iconY,
            width: iconSize,
            height: iconSize,
            fileId: fileId,
            status: "saved",
            scale: [1, 1],
            angle: 0,
            strokeColor: "transparent",
            backgroundColor: "transparent",
            strokeWidth: 0,
            strokeStyle: "solid",
            roughness: 1,
            opacity: 100,
            groupIds: [],
            frameId: null,
            index: "a" + existingElements.length,
            roundness: null,
            seed: Math.floor(Math.random() * 100000),
            version: 1,
            versionNonce: Math.floor(Math.random() * 1000000),
            isDeleted: false,
            boundElements: null,
            updated: timestamp,
            link: null,
            locked: false
        };

        // 插入到画布
        api.updateScene({
            elements: [...existingElements, imageElement],
            files: { ...files, [fileId]: newFile }
        });

        // 选中新添加的元素
        api.selectElements([imageElement]);

        // 强制刷新画布
        api.refresh();

        new Notice(`图标 "${icon.name}" 已插入到屏幕中心`);

    } catch (error) {
        console.error("[INSERT] failed:", error);
        new Notice(`插入图标失败: ${error.message}`);
    }
}

/**
 * 进入放置模式
 */
// Override the earlier implementation so insertion always targets the visible viewport center.
async function insertIconToScreenCenter(icon) {
    if (!icon) {
        console.error("[INSERT] icon is empty");
        return;
    }

    try {
        const safeSvg = sanitizeSvg(icon.show_svg);
        const view = ea.targetView;
        const api = view && view.excalidrawAPI;

        if (!api) {
            throw new Error("Failed to get Excalidraw API");
        }

        const appState = api.getAppState();
        const timestamp = Date.now();
        const randomId = Math.floor(Math.random() * 1000).toString().padStart(3, "0");
        const fileId = `${SCRIPT_NAME}_${icon.name.replace(/[^a-zA-Z0-9]/g, "_")}_${timestamp}_${randomId}`;
        const base64Svg = btoa(unescape(encodeURIComponent(safeSvg)));
        const dataUrl = `data:image/svg+xml;base64,${base64Svg}`;
        const files = api.getFiles();

        const newFile = {
            id: fileId,
            mimeType: "image/svg+xml",
            dataURL: dataUrl,
            size: dataUrl.length,
            created: timestamp,
            lastRetrieved: timestamp
        };

        if (typeof api.addFiles === "function") {
            api.addFiles({ [fileId]: newFile });
        } else {
            files[fileId] = newFile;
        }

        const existingElements = api.getSceneElements();
        const displaySize = 32;
        const zoom = (appState.zoom && appState.zoom.value) || 1;
        const iconSize = displaySize / zoom;
        const container = (view.containerEl && view.containerEl.querySelector && view.containerEl.querySelector(".excalidraw")) || view.containerEl;
        const rect = container && container.getBoundingClientRect ? container.getBoundingClientRect() : null;

        let viewportCenterX;
        let viewportCenterY;

        if (rect) {
            // Excalidraw transform: viewport = scene * zoom + scroll
            viewportCenterX = (rect.width / 2 - appState.scrollX) / zoom;
            viewportCenterY = (rect.height / 2 - appState.scrollY) / zoom;
        } else {
            viewportCenterX = -appState.scrollX / zoom;
            viewportCenterY = -appState.scrollY / zoom;
        }

        const iconX = viewportCenterX - iconSize / 2;
        const iconY = viewportCenterY - iconSize / 2;

        const elementId = `${fileId}_el`;
        const imageElement = {
            id: elementId,
            type: "image",
            x: iconX,
            y: iconY,
            width: iconSize,
            height: iconSize,
            fileId: fileId,
            status: "saved",
            scale: [1, 1],
            angle: 0,
            strokeColor: "transparent",
            backgroundColor: "transparent",
            strokeWidth: 0,
            strokeStyle: "solid",
            roughness: 1,
            opacity: 100,
            groupIds: [],
            frameId: null,
            index: "a" + existingElements.length,
            roundness: null,
            seed: Math.floor(Math.random() * 100000),
            version: 1,
            versionNonce: Math.floor(Math.random() * 1000000),
            isDeleted: false,
            boundElements: null,
            updated: timestamp,
            link: null,
            locked: false
        };

        api.updateScene({
            elements: [...existingElements, imageElement],
            files: { ...files, [fileId]: newFile }
        });

        api.selectElements([imageElement]);
        api.refresh();

        new Notice(`Icon "${icon.name}" inserted at viewport center`);
    } catch (error) {
        console.error("[INSERT] failed:", error);
        new Notice(`Insert failed: ${error.message}`);
    }
}

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
 * 获取画布坐标（简化版 - 只返回容器相对位置和缩放比例）
 * @param {MouseEvent} event - 鼠标事件
 * @returns {Object} 包含容器相对位置 {x, y} 和缩放比例 zoom
 */
function getCanvasCoordinates(event) {
    const view = ea.targetView;
    const api = view.excalidrawAPI;

    // 获取 Excalidraw 的完整状态
    const appState = api.getAppState();
    const zoom = appState.zoom.value;
    const scrollX = appState.scrollX;
    const scrollY = appState.scrollY;

    // 获取 Excalidraw 容器的位置
    const excalidrawContainer = view.containerEl.querySelector('.excalidraw-container');
    if (!excalidrawContainer) {
        console.warn("找不到 Excalidraw 容器");
        return { x: event.clientX, y: event.clientY, zoom };
    }

    const rect = excalidrawContainer.getBoundingClientRect();

    // 只需要容器相对位置（用于确定点击在屏幕上的相对位置）
    const containerX = event.clientX - rect.left;
    const containerY = event.clientY - rect.top;

    return {
        containerX,
        containerY,
        containerWidth: rect.width,
        containerHeight: rect.height,
        zoom,
        scrollX,
        scrollY
    };
}

/**
 * 将图标插入到画布
 * @param {Object} point - 插入位置 {x, y}
 */
async function insertIconToCanvas(point) {
    const icon = state.selectedIcon;
    if (!icon) return;

    try {
        // 使用清理后的 SVG
        const safeSvg = sanitizeSvg(icon.show_svg);

        // 获取 Excalidraw view 的 API
        const view = ea.targetView;
        const api = view.excalidrawAPI;

        if (!api) {
            throw new Error("无法获取 Excalidraw API");
        }

        // 获取 appState（用于坐标计算）
        const appState = api.getAppState();

        // 生成唯一的文件 ID（使用更简单的格式，便于重启后匹配）
        const timestamp = Date.now();
        const randomId = Math.floor(Math.random() * 1000).toString().padStart(3, '0');
        const fileId = `${SCRIPT_NAME}_${icon.name.replace(/[^a-zA-Z0-9]/g, '_')}_${timestamp}_${randomId}`;

        // 将 SVG 转换为 data URL（用于显示）
        const base64Svg = btoa(unescape(encodeURIComponent(safeSvg)));
        const dataUrl = `data:image/svg+xml;base64,${base64Svg}`;

        // 获取当前文件系统
        const files = api.getFiles();

        // 创建新文件对象（与 Excalidraw 的文件结构一致）
        const newFile = {
            id: fileId,
            mimeType: "image/svg+xml",
            dataURL: dataUrl,
            size: dataUrl.length,
            created: timestamp,
            // 添加这些字段以保持兼容性
            lastRetrieved: timestamp
        };

        // 使用 Excalidraw 的 addFiles API 来注册文件
        try {
            if (typeof api.addFiles === 'function') {
                api.addFiles({ [fileId]: newFile });
            } else {
                files[fileId] = newFile;
            }
        } catch (e) {
            console.warn("api.addFiles failed, using direct modification:", e);
            files[fileId] = newFile;
        }

        // 获取当前画布中的元素
        const existingElements = api.getSceneElements();

        // ============================================================================
        // 开始图标插入坐标计算
        // ============================================================================
        try {
            // ============================================================================
            // 步骤1: 计算图标大小（固定32px）
            // ============================================================================
            const iconSize = 32; // 固定图标大小

            // ============================================================================
            // 步骤2: 获取当前视口中心坐标
            // ============================================================================
            // Excalidraw 的 scrollX/scrollY 就是当前视口中心的画布坐标
            const centerX = appState.scrollX;
            const centerY = appState.scrollY;

            // ============================================================================
            // 步骤3: 计算图标位置（使图标中心对齐视口中心）
            // ============================================================================
            const halfIconSize = iconSize / 2;
            const iconX = centerX - halfIconSize;
            const iconY = centerY - halfIconSize;

            // 生成唯一的元素 ID
            const elementId = `${fileId}_el`;

            // 创建完整的图片元素
            const imageElement = {
                id: elementId,
                type: "image",
                x: iconX,
                y: iconY,
                width: iconSize,
                height: iconSize,
                fileId: fileId,
                status: "saved",
                scale: [1, 1],
                angle: 0,
                strokeColor: "transparent",
                backgroundColor: "transparent",
                strokeWidth: 0,
                strokeStyle: "solid",
                roughness: 1,
                opacity: 100,
                groupIds: [],
                frameId: null,
                index: "a" + existingElements.length,
                roundness: null,
                seed: Math.floor(Math.random() * 100000),
                version: 1,
                versionNonce: Math.floor(Math.random() * 1000000),
                isDeleted: false,
                boundElements: null,
                updated: timestamp,
                link: null,
                locked: false
            };

            // 使用 Excalidraw 原生 API 的 updateScene 方法
            api.updateScene({
              elements: [...existingElements, imageElement],
              files: {
                ...files,
                [fileId]: newFile
              }
            });

            // 选中新添加的元素
            api.selectElements([imageElement]);

            // 强制刷新画布
            api.refresh();

            new Notice(`图标已插入 (${iconSize.toFixed(0)}x${iconSize.toFixed(0)}) 在 (${iconX.toFixed(0)}, ${iconY.toFixed(0)})`);

        } catch (error) {
            console.error("[ERROR] Icon insertion failed:", error);
            new Notice(`插入图标失败: ${error.message}`);
        }
    } catch (error) {
        console.error("[ERROR] Icon insertion failed (outer):", error);
        new Notice(`插入图标失败: ${error.message}`);
        console.error('Icon insertion error:', error);
        console.error('Error stack:', error.stack);
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
function formatAttachmentTimestamp(timestamp) {
    const date = new Date(timestamp);
    const pad = (value) => String(value).padStart(2, "0");
    return [
        date.getFullYear(),
        pad(date.getMonth() + 1),
        pad(date.getDate())
    ].join("") + [
        pad(date.getHours()),
        pad(date.getMinutes()),
        pad(date.getSeconds())
    ].join("");
}

async function waitForNextFrame() {
    await new Promise((resolve) => {
        if (typeof requestAnimationFrame === "function") {
            requestAnimationFrame(() => resolve());
            return;
        }

        setTimeout(resolve, 16);
    });
}

async function forceImageElementRerender(api, elementId, files) {
    await waitForNextFrame();
    await waitForNextFrame();

    const sceneElements = api.getSceneElements();
    const targetElement = sceneElements.find((element) => element.id === elementId);
    if (!targetElement) {
        return null;
    }

    const rerenderedElement = {
        ...targetElement,
        version: (targetElement.version || 1) + 1,
        versionNonce: Math.floor(Math.random() * 1000000),
        updated: Date.now()
    };

    api.updateScene({
        elements: sceneElements.map((element) => element.id === rerenderedElement.id ? rerenderedElement : element),
        files: files || api.getFiles()
    });
    api.refresh();

    return rerenderedElement;
}

async function insertPersistentSvgIcon(icon, x, y, size) {
    const view = ea.targetView;
    const api = view && view.excalidrawAPI;
    const obsidianApp = app || ea.app;
    const obsidianVault = vault || (obsidianApp && obsidianApp.vault);

    if (!api) {
        throw new Error("Failed to get Excalidraw API");
    }

    if (!obsidianVault) {
        throw new Error("Failed to get Obsidian vault");
    }

    if (typeof ea.getAttachmentFilepath !== "function" || typeof ea.addImage !== "function") {
        throw new Error("Current Excalidraw Automate version does not support persistent image insertion");
    }

    const safeSvg = sanitizeSvg(icon.show_svg);
    const timestamp = Date.now();
    const randomId = Math.floor(Math.random() * 1000).toString().padStart(3, "0");
    const attachmentName = `Pasted Image ${formatAttachmentTimestamp(timestamp)}_${randomId}.svg`;
    const attachmentPath = await ea.getAttachmentFilepath(attachmentName);
    const normalizedAttachmentPath = attachmentPath.replace(/\\/g, "/");
    const lastSlashIndex = normalizedAttachmentPath.lastIndexOf("/");
    const folderPath = lastSlashIndex >= 0 ? normalizedAttachmentPath.substring(0, lastSlashIndex) : "";

    if (folderPath) {
        try {
            await obsidianVault.adapter.mkdir(folderPath);
        } catch (error) {
            // Folder may already exist.
        }
    }

    const existingAttachment = obsidianVault.getAbstractFileByPath(normalizedAttachmentPath);
    if (existingAttachment && typeof obsidianVault.modify === "function") {
        await obsidianVault.modify(existingAttachment, safeSvg);
    } else {
        await obsidianVault.create(normalizedAttachmentPath, safeSvg);
    }

    const beforeSceneElements = api.getSceneElements();
    const beforeElementIds = new Set(beforeSceneElements.map((element) => element.id));
    const beforeImageFileIds = new Set(
        beforeSceneElements
            .filter((element) => element.type === "image" && element.fileId)
            .map((element) => element.fileId)
    );
    const insertedElementId = await ea.addImage(x, y, normalizedAttachmentPath, false, false);

    if (typeof ea.addElementsToView === "function") {
        await ea.addElementsToView(false, true, true);
    }

    const sceneElements = api.getSceneElements();
    const insertedElement =
        sceneElements.find((element) => element.id === insertedElementId) ||
        sceneElements.find((element) => element.type === "image" && !beforeElementIds.has(element.id));

    if (!insertedElement) {
        throw new Error("Image element was not added to the scene");
    }

    const resizedElement = {
        ...insertedElement,
        x,
        y,
        width: size,
        height: size,
        version: (insertedElement.version || 1) + 1,
        versionNonce: Math.floor(Math.random() * 1000000),
        updated: Date.now()
    };

    api.updateScene({
        elements: sceneElements.map((element) => element.id === resizedElement.id ? resizedElement : element)
    });

    api.selectElements([resizedElement]);
    api.refresh();

    return {
        attachmentPath: normalizedAttachmentPath,
        element: resizedElement
    };
}

// Override the earlier implementation to persist SVGs as real Excalidraw attachments.
async function insertIconToScreenCenter(icon) {
    if (!icon) {
        console.error("[INSERT] icon is empty");
        return;
    }

    try {
        const view = ea.targetView;
        const api = view && view.excalidrawAPI;

        if (!api) {
            throw new Error("Failed to get Excalidraw API");
        }

        const appState = api.getAppState();
        const zoom = (appState.zoom && appState.zoom.value) || 1;
        const iconSize = 32 / zoom;
        const container = (view.containerEl && view.containerEl.querySelector && view.containerEl.querySelector(".excalidraw")) || view.containerEl;
        const rect = container && container.getBoundingClientRect ? container.getBoundingClientRect() : null;

        let viewportCenterX;
        let viewportCenterY;

        if (rect) {
            viewportCenterX = (rect.width / 2 - appState.scrollX) / zoom;
            viewportCenterY = (rect.height / 2 - appState.scrollY) / zoom;
        } else {
            viewportCenterX = -appState.scrollX / zoom;
            viewportCenterY = -appState.scrollY / zoom;
        }

        const iconX = viewportCenterX - iconSize / 2;
        const iconY = viewportCenterY - iconSize / 2;
        const result = await insertPersistentSvgIcon(icon, iconX, iconY, iconSize);

        new Notice(`Icon "${icon.name}" inserted at viewport center`);
    } catch (error) {
        console.error("[INSERT] failed:", error);
        new Notice(`Insert failed: ${error.message}`);
    }
}

// Override the earlier canvas-placement implementation to persist SVGs as real attachments.
async function insertIconToCanvas(point) {
    const icon = state.selectedIcon;
    if (!icon) return;

    try {
        const view = ea.targetView;
        const api = view && view.excalidrawAPI;

        if (!api) {
            throw new Error("Failed to get Excalidraw API");
        }

        const iconSize = 32;
        const centerX = point && typeof point.scrollX === "number" ? point.scrollX : api.getAppState().scrollX;
        const centerY = point && typeof point.scrollY === "number" ? point.scrollY : api.getAppState().scrollY;
        const iconX = centerX - iconSize / 2;
        const iconY = centerY - iconSize / 2;
        const result = await insertPersistentSvgIcon(icon, iconX, iconY, iconSize);

        new Notice(`Icon "${icon.name}" inserted`);
    } catch (error) {
        console.error("[INSERT] canvas placement failed:", error);
        new Notice(`Insert failed: ${error.message}`);
    }
}

function getObsidianAppAndVault() {
    const view = ea && ea.targetView;
    const candidateApps = [
        app,
        ea && ea.app,
        view && view.app,
        typeof globalThis !== "undefined" ? globalThis.app : null,
        typeof window !== "undefined" ? window.app : null
    ].filter(Boolean);

    for (const candidateApp of candidateApps) {
        if (candidateApp && candidateApp.vault) {
            return {
                app: candidateApp,
                vault: candidateApp.vault
            };
        }
    }

    return {
        app: candidateApps[0] || null,
        vault: null
    };
}

async function insertPersistentSvgIcon(icon, x, y, size) {
    const view = ea.targetView;
    const api = view && view.excalidrawAPI;
    const context = getObsidianAppAndVault();
    const obsidianVault = context.vault;

    if (!api) {
        throw new Error("Failed to get Excalidraw API");
    }

    if (!obsidianVault) {
        console.error("[INSERT] Obsidian context unavailable", {
            hasGlobalApp: typeof globalThis !== "undefined" && !!globalThis.app,
            hasWindowApp: typeof window !== "undefined" && !!window.app,
            hasEaApp: !!(ea && ea.app),
            hasViewApp: !!(view && view.app)
        });
        throw new Error("Failed to get Obsidian vault");
    }

    if (typeof ea.getAttachmentFilepath !== "function" || typeof ea.addImage !== "function") {
        throw new Error("Current Excalidraw Automate version does not support persistent image insertion");
    }

    const safeSvg = sanitizeSvg(icon.show_svg);
    const timestamp = Date.now();
    const randomId = Math.floor(Math.random() * 1000).toString().padStart(3, "0");
    const attachmentName = `Pasted Image ${formatAttachmentTimestamp(timestamp)}_${randomId}.svg`;
    const attachmentPath = await ea.getAttachmentFilepath(attachmentName);
    const normalizedAttachmentPath = attachmentPath.replace(/\\/g, "/");
    const lastSlashIndex = normalizedAttachmentPath.lastIndexOf("/");
    const folderPath = lastSlashIndex >= 0 ? normalizedAttachmentPath.substring(0, lastSlashIndex) : "";

    if (folderPath && obsidianVault.adapter && typeof obsidianVault.adapter.mkdir === "function") {
        try {
            await obsidianVault.adapter.mkdir(folderPath);
        } catch (error) {
            // Folder may already exist.
        }
    }

    const existingAttachment = typeof obsidianVault.getAbstractFileByPath === "function"
        ? obsidianVault.getAbstractFileByPath(normalizedAttachmentPath)
        : null;

    if (existingAttachment && typeof obsidianVault.modify === "function") {
        await obsidianVault.modify(existingAttachment, safeSvg);
    } else if (typeof obsidianVault.create === "function") {
        await obsidianVault.create(normalizedAttachmentPath, safeSvg);
    } else if (obsidianVault.adapter && typeof obsidianVault.adapter.write === "function") {
        await obsidianVault.adapter.write(normalizedAttachmentPath, safeSvg);
    } else {
        throw new Error("Vault does not support writing attachment files");
    }

    const beforeElementIds = new Set(api.getSceneElements().map((element) => element.id));
    const insertedElementId = await ea.addImage(x, y, normalizedAttachmentPath, false, false);

    if (typeof ea.addElementsToView === "function") {
        await ea.addElementsToView(false, true, true);
    }

    const sceneElements = api.getSceneElements();
    const insertedElement =
        sceneElements.find((element) => element.id === insertedElementId) ||
        sceneElements.find((element) => element.type === "image" && !beforeElementIds.has(element.id));

    if (!insertedElement) {
        throw new Error("Image element was not added to the scene");
    }

    const resizedElement = {
        ...insertedElement,
        x,
        y,
        width: size,
        height: size,
        version: (insertedElement.version || 1) + 1,
        versionNonce: Math.floor(Math.random() * 1000000),
        updated: Date.now()
    };

    api.updateScene({
        elements: sceneElements.map((element) => element.id === resizedElement.id ? resizedElement : element)
    });

    api.selectElements([resizedElement]);
    api.refresh();

    return {
        attachmentPath: normalizedAttachmentPath,
        element: resizedElement
    };
}

await main();

// ===== 工具按钮入口 =====
/**
 * Excalidraw 工具按钮点击处理函数
 * 当用户点击工具栏的图标按钮时调用
 */
function getVisibleSceneCenter() {
    const view = ea.targetView;
    const api = view && view.excalidrawAPI;

    if (!api) {
        throw new Error("Failed to get Excalidraw API");
    }

    const appState = api.getAppState();
    const zoom = (appState.zoom && appState.zoom.value) || 1;
    const container =
        (view.containerEl && view.containerEl.querySelector && view.containerEl.querySelector(".excalidraw-container")) ||
        (view.containerEl && view.containerEl.querySelector && view.containerEl.querySelector(".excalidraw")) ||
        view.containerEl;
    const rect = container && container.getBoundingClientRect ? container.getBoundingClientRect() : null;

    let centerX;
    let centerY;

    if (rect) {
        centerX = (rect.width / 2 - appState.scrollX) / zoom;
        centerY = (rect.height / 2 - appState.scrollY) / zoom;
    } else {
        centerX = -appState.scrollX / zoom;
        centerY = -appState.scrollY / zoom;
    }

    return { centerX, centerY, zoom };
}

async function insertPersistentSvgIcon(icon, x, y, size) {
    const view = ea.targetView;
    const api = view && view.excalidrawAPI;
    const context = getObsidianAppAndVault();
    const obsidianVault = context.vault;

    if (!api) {
        throw new Error("Failed to get Excalidraw API");
    }

    if (!obsidianVault) {
        throw new Error("Failed to get Obsidian vault");
    }

    if (typeof ea.getAttachmentFilepath !== "function" || typeof ea.addImage !== "function") {
        throw new Error("Current Excalidraw Automate version does not support persistent image insertion");
    }

    const safeSvg = sanitizeSvg(icon.show_svg);
    const base64Svg = btoa(unescape(encodeURIComponent(safeSvg)));
    const dataUrl = `data:image/svg+xml;base64,${base64Svg}`;
    const timestamp = Date.now();
    const randomId = Math.floor(Math.random() * 1000).toString().padStart(3, "0");
    const attachmentName = `Pasted Image ${formatAttachmentTimestamp(timestamp)}_${randomId}.svg`;
    const attachmentPath = await ea.getAttachmentFilepath(attachmentName);
    const normalizedAttachmentPath = attachmentPath.replace(/\\/g, "/");
    const lastSlashIndex = normalizedAttachmentPath.lastIndexOf("/");
    const folderPath = lastSlashIndex >= 0 ? normalizedAttachmentPath.substring(0, lastSlashIndex) : "";

    if (folderPath && obsidianVault.adapter && typeof obsidianVault.adapter.mkdir === "function") {
        try {
            await obsidianVault.adapter.mkdir(folderPath);
        } catch (error) {
            // Folder may already exist.
        }
    }

    const existingAttachment = typeof obsidianVault.getAbstractFileByPath === "function"
        ? obsidianVault.getAbstractFileByPath(normalizedAttachmentPath)
        : null;

    if (existingAttachment && typeof obsidianVault.modify === "function") {
        await obsidianVault.modify(existingAttachment, safeSvg);
    } else if (typeof obsidianVault.create === "function") {
        await obsidianVault.create(normalizedAttachmentPath, safeSvg);
    } else if (obsidianVault.adapter && typeof obsidianVault.adapter.write === "function") {
        await obsidianVault.adapter.write(normalizedAttachmentPath, safeSvg);
    } else {
        throw new Error("Vault does not support writing attachment files");
    }

    const beforeElementIds = new Set(api.getSceneElements().map((element) => element.id));
    const insertedElementId = await ea.addImage(x, y, normalizedAttachmentPath, false, false);

    if (typeof ea.addElementsToView === "function") {
        await ea.addElementsToView(false, true, true);
    }

    let sceneElements = api.getSceneElements();
    const newImageCandidates = sceneElements.filter((element) => {
        if (element.type !== "image") return false;
        if (!beforeElementIds.has(element.id)) return true;
        if (element.fileId && !beforeImageFileIds.has(element.fileId)) return true;
        return false;
    });

    let insertedElement =
        sceneElements.find((element) => element.id === insertedElementId) ||
        newImageCandidates.sort((a, b) => (b.updated || 0) - (a.updated || 0))[0] ||
        null;

    if (!insertedElement) {
        throw new Error("Image element was not added to the scene");
    }

    let hydratedFiles = null;

    if (insertedElement.fileId) {
        const existingFiles = api.getFiles();
        const hydratedFile = {
            id: insertedElement.fileId,
            mimeType: "image/svg+xml",
            dataURL: dataUrl,
            size: dataUrl.length,
            created: timestamp,
            lastRetrieved: timestamp
        };

        if (typeof api.addFiles === "function") {
            api.addFiles({ [insertedElement.fileId]: hydratedFile });
        }

        hydratedFiles = { ...existingFiles, [insertedElement.fileId]: hydratedFile };

        api.updateScene({
            files: hydratedFiles
        });

        sceneElements = api.getSceneElements();
        insertedElement = sceneElements.find((element) => element.id === insertedElement.id) || insertedElement;
    }

    const resizedElement = {
        ...insertedElement,
        x,
        y,
        width: size,
        height: size,
        status: "saved",
        scale: [1, 1],
        version: (insertedElement.version || 1) + 1,
        versionNonce: Math.floor(Math.random() * 1000000),
        updated: Date.now()
    };

    api.updateScene({
        elements: sceneElements.map((element) => element.id === resizedElement.id ? resizedElement : element),
        files: hydratedFiles || api.getFiles()
    });

    const rerenderedElement = await forceImageElementRerender(api, resizedElement.id, hydratedFiles || api.getFiles());
    const finalElement = rerenderedElement || resizedElement;

    api.selectElements([finalElement]);
    api.refresh();

    return {
        attachmentPath: normalizedAttachmentPath,
        element: finalElement
    };
}

async function insertIconToScreenCenter(icon) {
    if (!icon) {
        console.error("[INSERT] icon is empty");
        return;
    }

    try {
        const { centerX, centerY, zoom } = getVisibleSceneCenter();
        const iconSize = 32 / zoom;
        const iconX = centerX - iconSize / 2;
        const iconY = centerY - iconSize / 2;
        let result = await insertPersistentSvgIcon(icon, iconX, iconY, iconSize);
        const latestCenter = getVisibleSceneCenter();
        const finalElement = result.element;
        const correctedX = latestCenter.centerX - finalElement.width / 2;
        const correctedY = latestCenter.centerY - finalElement.height / 2;

        if (Math.abs(finalElement.x - correctedX) > 0.5 || Math.abs(finalElement.y - correctedY) > 0.5) {
            const api = ea.targetView && ea.targetView.excalidrawAPI;
            const sceneElements = api.getSceneElements();
            const correctedElement = {
                ...finalElement,
                x: correctedX,
                y: correctedY,
                version: (finalElement.version || 1) + 1,
                versionNonce: Math.floor(Math.random() * 1000000),
                updated: Date.now()
            };

            api.updateScene({
                elements: sceneElements.map((element) => element.id === correctedElement.id ? correctedElement : element),
                files: api.getFiles()
            });
            api.selectElements([correctedElement]);
            api.refresh();
            result = { ...result, element: correctedElement };
        }

        new Notice(`Icon "${icon.name}" inserted at viewport center`);
    } catch (error) {
        console.error("[INSERT] failed:", error);
        new Notice(`Insert failed: ${error.message}`);
    }
}

async function insertIconToCanvas(point) {
    const icon = state.selectedIcon;
    if (!icon) return;

    try {
        const api = ea.targetView && ea.targetView.excalidrawAPI;
        if (!api) {
            throw new Error("Failed to get Excalidraw API");
        }

        const iconSize = 32;
        const centerX = point && typeof point.scrollX === "number" ? point.scrollX : api.getAppState().scrollX;
        const centerY = point && typeof point.scrollY === "number" ? point.scrollY : api.getAppState().scrollY;
        const iconX = centerX - iconSize / 2;
        const iconY = centerY - iconSize / 2;
        const result = await insertPersistentSvgIcon(icon, iconX, iconY, iconSize);

        new Notice(`Icon "${icon.name}" inserted`);
    } catch (error) {
        console.error("[INSERT] canvas placement failed:", error);
        new Notice(`Insert failed: ${error.message}`);
    }
}

// Final override: use Excalidraw Automate's image insertion flow directly, then resize the returned element by id.
async function insertPersistentSvgIcon(icon, x, y, size) {
    const view = ea.targetView;
    const api = view && view.excalidrawAPI;
    const context = getObsidianAppAndVault();
    const obsidianVault = context.vault;

    if (!api) {
        throw new Error("Failed to get Excalidraw API");
    }

    if (!obsidianVault) {
        throw new Error("Failed to get Obsidian vault");
    }

    if (typeof ea.getAttachmentFilepath !== "function" || typeof ea.addImage !== "function") {
        throw new Error("Current Excalidraw Automate version does not support persistent image insertion");
    }

    const safeSvg = sanitizeSvg(icon.show_svg);
    const timestamp = Date.now();
    const randomId = Math.floor(Math.random() * 1000).toString().padStart(3, "0");
    const attachmentName = `Pasted Image ${formatAttachmentTimestamp(timestamp)}_${randomId}.svg`;
    const attachmentPath = await ea.getAttachmentFilepath(attachmentName);
    const normalizedAttachmentPath = attachmentPath.replace(/\\/g, "/");
    const lastSlashIndex = normalizedAttachmentPath.lastIndexOf("/");
    const folderPath = lastSlashIndex >= 0 ? normalizedAttachmentPath.substring(0, lastSlashIndex) : "";

    if (folderPath && obsidianVault.adapter && typeof obsidianVault.adapter.mkdir === "function") {
        try {
            await obsidianVault.adapter.mkdir(folderPath);
        } catch (error) {
            // Folder may already exist.
        }
    }

    const existingAttachment = typeof obsidianVault.getAbstractFileByPath === "function"
        ? obsidianVault.getAbstractFileByPath(normalizedAttachmentPath)
        : null;

    if (existingAttachment && typeof obsidianVault.modify === "function") {
        await obsidianVault.modify(existingAttachment, safeSvg);
    } else if (typeof obsidianVault.create === "function") {
        await obsidianVault.create(normalizedAttachmentPath, safeSvg);
    } else if (obsidianVault.adapter && typeof obsidianVault.adapter.write === "function") {
        await obsidianVault.adapter.write(normalizedAttachmentPath, safeSvg);
    } else {
        throw new Error("Vault does not support writing attachment files");
    }

    if (typeof ea.clear === "function") {
        ea.clear();
    } else if (typeof ea.reset === "function") {
        ea.reset();
    }

    const insertedElementId = await ea.addImage(x, y, normalizedAttachmentPath, false, false);
    const insertedDraft = typeof ea.getElement === "function" ? ea.getElement(insertedElementId) : null;

    if (insertedDraft) {
        insertedDraft.x = x;
        insertedDraft.y = y;
        insertedDraft.width = size;
        insertedDraft.height = size;
        insertedDraft.scale = [1, 1];
        insertedDraft.status = "saved";
    }

    if (typeof ea.addElementsToView === "function") {
        await ea.addElementsToView(false, true, true);
    }

    await waitForNextFrame();

    const sceneElements = api.getSceneElements();
    const insertedElement = sceneElements.find((element) => element.id === insertedElementId);

    if (!insertedElement) {
        throw new Error("Image element was not added to the scene");
    }

    api.selectElements([insertedElement]);
    api.refresh();

    return {
        attachmentPath: normalizedAttachmentPath,
        element: insertedElement
    };
}

function action() {
    openSearchPanel();
}

// Final stable overrides to avoid hitting earlier partially-edited implementations.
async function insertPersistentSvgIcon(icon, x, y, iconWidth, iconHeight) {
    // iconWidth, iconHeight: already calculated with correct aspect ratio
    const view = ea.targetView;
    const api = view && view.excalidrawAPI;
    const context = getObsidianAppAndVault();
    const obsidianVault = context.vault;

    if (!api) {
        throw new Error("Failed to get Excalidraw API");
    }

    if (!obsidianVault) {
        throw new Error("Failed to get Obsidian vault");
    }

    if (typeof ea.getAttachmentFilepath !== "function" || typeof ea.addImage !== "function") {
        throw new Error("Current Excalidraw Automate version does not support persistent image insertion");
    }

    const safeSvg = sanitizeSvg(icon.show_svg);
    const timestamp = Date.now();
    const randomId = Math.floor(Math.random() * 1000).toString().padStart(3, "0");
    const attachmentName = `Pasted Image ${formatAttachmentTimestamp(timestamp)}_${randomId}.svg`;
    const attachmentPath = await ea.getAttachmentFilepath(attachmentName);
    const normalizedAttachmentPath = attachmentPath.replace(/\\/g, "/");
    const lastSlashIndex = normalizedAttachmentPath.lastIndexOf("/");
    const folderPath = lastSlashIndex >= 0 ? normalizedAttachmentPath.substring(0, lastSlashIndex) : "";

    if (folderPath && obsidianVault.adapter && typeof obsidianVault.adapter.mkdir === "function") {
        try {
            await obsidianVault.adapter.mkdir(folderPath);
        } catch (error) {
            // Folder may already exist.
        }
    }

    const existingAttachment = typeof obsidianVault.getAbstractFileByPath === "function"
        ? obsidianVault.getAbstractFileByPath(normalizedAttachmentPath)
        : null;

    if (existingAttachment && typeof obsidianVault.modify === "function") {
        await obsidianVault.modify(existingAttachment, safeSvg);
    } else if (typeof obsidianVault.create === "function") {
        await obsidianVault.create(normalizedAttachmentPath, safeSvg);
    } else if (obsidianVault.adapter && typeof obsidianVault.adapter.write === "function") {
        await obsidianVault.adapter.write(normalizedAttachmentPath, safeSvg);
    } else {
        throw new Error("Vault does not support writing attachment files");
    }

    if (typeof ea.clear === "function") {
        ea.clear();
    } else if (typeof ea.reset === "function") {
        ea.reset();
    }

    const insertedElementId = await ea.addImage(x, y, normalizedAttachmentPath, false, false);
    const insertedDraft = typeof ea.getElement === "function" ? ea.getElement(insertedElementId) : null;

    if (insertedDraft) {
        insertedDraft.x = x;
        insertedDraft.y = y;
        insertedDraft.width = iconWidth;
        insertedDraft.height = iconHeight;
        insertedDraft.scale = [1, 1];
        insertedDraft.status = "saved";
    }

    if (typeof ea.addElementsToView === "function") {
        await ea.addElementsToView(false, true, true);
    }

    await waitForNextFrame();

    const sceneElements = api.getSceneElements();
    const insertedElement = sceneElements.find((element) => element.id === insertedElementId);

    if (!insertedElement) {
        throw new Error("Image element was not added to the scene");
    }

    api.selectElements([insertedElement]);
    api.refresh();

    return {
        attachmentPath: normalizedAttachmentPath,
        element: insertedElement
    };
}

async function insertIconToScreenCenter(icon) {
    if (!icon) return;

    try {
        const { centerX, centerY, zoom } = getVisibleSceneCenter();
        const baseSize = 32 / zoom;
        const aspectRatio = getIconAspectRatio(icon);
        const { width: iconWidth, height: iconHeight } = calculateIconSize(baseSize, aspectRatio);
        const iconX = centerX - iconWidth / 2;
        const iconY = centerY - iconHeight / 2;
        const result = await insertPersistentSvgIcon(icon, iconX, iconY, iconWidth, iconHeight);
        new Notice(`Icon "${icon.name}" inserted at viewport center`);
        return result;
    } catch (error) {
        console.error("[INSERT] failed:", error);
        new Notice(`Insert failed: ${error.message}`);
        throw error;
    }
}

async function insertIconToCanvas(point) {
    const icon = state.selectedIcon;
    if (!icon) return;

    try {
        const api = ea.targetView && ea.targetView.excalidrawAPI;
        if (!api) {
            throw new Error("Failed to get Excalidraw API");
        }

        const baseSize = 32;
        const aspectRatio = getIconAspectRatio(icon);
        const { width: iconWidth, height: iconHeight } = calculateIconSize(baseSize, aspectRatio);
        const centerX = point && typeof point.scrollX === "number" ? point.scrollX : api.getAppState().scrollX;
        const centerY = point && typeof point.scrollY === "number" ? point.scrollY : api.getAppState().scrollY;
        const iconX = centerX - iconWidth / 2;
        const iconY = centerY - iconHeight / 2;
        const result = await insertPersistentSvgIcon(icon, iconX, iconY, iconWidth, iconHeight);
        new Notice(`Icon "${icon.name}" inserted`);
        return result;
    } catch (error) {
        console.error("[INSERT] canvas placement failed:", error);
        new Notice(`Insert failed: ${error.message}`);
        throw error;
    }
}
