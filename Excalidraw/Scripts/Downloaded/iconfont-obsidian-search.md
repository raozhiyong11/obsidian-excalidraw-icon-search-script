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

    // 重置 UI 状态（保留放置模式相关的状态）
    state.isOpen = false;
    state.isLoading = false;
    state.icons = [];
    state.currentPage = 1;

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

    // 优化 SVG 以确保正确填充容器
    // 设置或更新 viewBox 为 0 0 1024 1024（语雀图标的标准尺寸）
    if (!svgElement.getAttribute('viewBox')) {
        svgElement.setAttribute('viewBox', '0 0 1024 1024');
    }

    // 设置 width 和 height 为 100%，确保填充容器
    svgElement.setAttribute('width', '100%');
    svgElement.setAttribute('height', '100%');

    // 添加 preserveAspectRatio 确保图标均匀缩放
    svgElement.setAttribute('preserveAspectRatio', 'xMidYMid meet');

    return svgElement.outerHTML;
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

    console.log("=== 鼠标点击信息 ===");
    console.log("容器相对位置:", { x: containerX.toFixed(2), y: containerY.toFixed(2) });
    console.log("容器尺寸:", { width: rect.width, height: rect.height });
    console.log("缩放比例:", zoom);
    console.log("滚动偏移:", { x: scrollX.toFixed(2), y: scrollY.toFixed(2) });
    console.log("====================");

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

        console.log("准备插入图标:", icon.name);
        console.log("插入位置:", point);

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

        console.log("生成文件 ID:", fileId);

        // 将 SVG 转换为 data URL（用于显示）
        const base64Svg = btoa(unescape(encodeURIComponent(safeSvg)));
        const dataUrl = `data:image/svg+xml;base64,${base64Svg}`;
        console.log("Data URL 长度:", dataUrl.length);

        // 获取当前文件系统
        const files = api.getFiles();
        console.log("注册前文件数量:", Object.keys(files).length);

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

        console.log("准备注册文件:", { id: newFile.id, mimeType: newFile.mimeType, size: newFile.size });

        // 使用 Excalidraw 的 addFiles API 来注册文件
        try {
            if (typeof api.addFiles === 'function') {
                console.log("使用 api.addFiles 注册文件");
                api.addFiles({ [fileId]: newFile });
            } else {
                console.log("addFiles 不存在，直接修改 files 对象");
                files[fileId] = newFile;
            }
        } catch (e) {
            console.warn("api.addFiles 失败，使用直接修改:", e);
            files[fileId] = newFile;
        }

        console.log("注册后文件数量:", Object.keys(api.getFiles()).length);
        console.log("已注册文件 ID:", fileId);

        // 获取当前画布中的元素
        const existingElements = api.getSceneElements();
        console.log("插入前元素数量:", existingElements.length);

        // 生成唯一的元素 ID
        const elementId = `${fileId}_el`;

        // ============================================================================
        // 开始图标插入坐标计算
        // ============================================================================
        console.log("🔵 [INSERT] ========== 开始图标插入流程 ==========");
        console.log("🔵 [INSERT] 选中的图标:", state.selectedIcon?.name || "未找到");

        try {
            // ============================================================================
            // 步骤1: 计算图标显示大小（在画布坐标系统中）
            // ============================================================================
            const displaySize = 32; // 用户期望看到的图标大小（屏幕像素）
            const iconSize = displaySize / point.zoom; // 画布坐标中的尺寸（随缩放调整）

            console.log("🟢 [SIZE] 计算图标尺寸:");
            console.log("  - 用户期望尺寸 (屏幕):", displaySize, "px");
            console.log("  - 当前画布缩放:", point.zoom);
            console.log("  - 画布坐标尺寸:", iconSize.toFixed(2), "px");
            console.log("  - 尺寸计算公式: 32 ÷", point.zoom, "=", iconSize.toFixed(2));

            // ============================================================================
            // 步骤2: 验证输入数据的有效性
            // ============================================================================
            console.log("🔵 [VALIDATE] 验证输入数据:");
            console.log("  - containerX:", point.containerX, "类型:", typeof point.containerX);
            console.log("  - containerY:", point.containerY, "类型:", typeof point.containerY);
            console.log("  - zoom:", point.zoom, "类型:", typeof point.zoom);
            console.log("  - scrollX:", point.scrollX);
            console.log("  - scrollY:", point.scrollY);

            // 数据有效性检查
            if (typeof point.containerX !== 'number' || typeof point.containerY !== 'number') {
                throw new Error("containerX 或 containerY 不是有效数字");
            }
            if (typeof point.zoom !== 'number' || point.zoom <= 0) {
                throw new Error("zoom 不是有效数字或 <= 0");
            }
            if (!isFinite(point.containerX) || !isFinite(point.containerY)) {
                throw new Error("containerX 或 containerY 为无穷大");
            }

            // ============================================================================
            // 步骤3: 计算安全的缩放比例（防止除以0或极端值）
            // ============================================================================
            const MIN_ZOOM = 0.01;  // 最小缩放：1%
            const MAX_ZOOM = 100;   // 最大缩放：10000%
            const safeZoom = Math.max(MIN_ZOOM, Math.min(point.zoom, MAX_ZOOM));

            console.log("🟢 [ZOOM] 缩放比例处理:");
            console.log("  - 原始 zoom:", point.zoom);
            console.log("  - 安全 zoom:", safeZoom);
            if (safeZoom !== point.zoom) {
                console.log("  ⚠️  zoom 已被限制到安全范围 [" + MIN_ZOOM + ", " + MAX_ZOOM + "]");
            }

            // ============================================================================
            // 步骤4: 计算鼠标点击位置对应的画布坐标（方案1：纯画布坐标法）
            // ============================================================================
            console.log("🔵 [COORD] 坐标转换开始（方案1：纯画布坐标法）");
            console.log("  - 计算公式: canvasX = containerX / zoom");
            console.log("  - containerX:", point.containerX);
            console.log("  - 除以 safeZoom:", safeZoom);

            const mouseCanvasX = point.containerX / safeZoom;
            const mouseCanvasY = point.containerY / safeZoom;

            console.log("🟢 [COORD] 鼠标画布坐标计算结果:");
            console.log("  - mouseCanvasX:", mouseCanvasX.toFixed(2));
            console.log("  - mouseCanvasY:", mouseCanvasY.toFixed(2));
            console.log("  - 📐 说明: 这是鼠标点击位置在画布坐标系中的位置");

            // 坐标有效性检查
            if (!isFinite(mouseCanvasX) || !isFinite(mouseCanvasY)) {
                console.error("🔴 [ERROR] 计算出的坐标无效!");
                console.error("  - mouseCanvasX:", mouseCanvasX);
                console.error("  - mouseCanvasY:", mouseCanvasY);
                throw new Error("计算出的坐标无效，请检查画布状态");
            }

            // ============================================================================
            // 步骤5: 计算图标左上角位置（使图标中心对齐鼠标）
            // ============================================================================
            const halfIconSize = iconSize / 2;
            const iconX = mouseCanvasX - halfIconSize;
            const iconY = mouseCanvasY - halfIconSize;

            console.log("🟢 [POSITION] 图标位置计算:");
            console.log("  - 图标中心对齐 → 需要减去图标尺寸的一半");
            console.log("  - 图标尺寸:", iconSize.toFixed(2));
            console.log("  - 半尺寸 (halfIconSize):", halfIconSize.toFixed(2));
            console.log("  - iconX = mouseCanvasX - halfIconSize");
            console.log("  - iconX =", mouseCanvasX.toFixed(2), "-", halfIconSize.toFixed(2), "=", iconX.toFixed(2));
            console.log("  - iconY = mouseCanvasY - halfIconSize");
            console.log("  - iconY =", mouseCanvasY.toFixed(2), "-", halfIconSize.toFixed(2), "=", iconY.toFixed(2));
            console.log("  - 📍 图标左上角位置: (" + iconX.toFixed(2) + ", " + iconY.toFixed(2) + ")");

            // ============================================================================
            // 步骤6: 汇总信息
            // ============================================================================
            console.log("🔵 [SUMMARY] 图标插入信息汇总:");
            console.log("┌────────────────────────────────────────────┐");
            console.log("│  画布缩放: " + String(point.zoom.toFixed(2)).padStart(10) + " (" + (point.zoom * 100).toFixed(0) + "%)         │");
            console.log("│  图标尺寸: " + String(iconSize.toFixed(2) + "px").padStart(10) + "                   │");
            console.log("│  鼠标屏幕坐标: (" + point.containerX.toFixed(0) + ", " + point.containerY.toFixed(0) + ")              │");
            console.log("│  鼠标画布坐标: (" + mouseCanvasX.toFixed(0) + ", " + mouseCanvasY.toFixed(0) + ")             │");
            console.log("│  图标左上角: (" + iconX.toFixed(0) + ", " + iconY.toFixed(0) + ")                   │");
            console.log("│  图标右下角: (" + (iconX + iconSize).toFixed(0) + ", " + (iconY + iconSize).toFixed(0) + ")           │");
            console.log("│  图标中心点: (" + (iconX + halfIconSize).toFixed(0) + ", " + (iconY + halfIconSize).toFixed(0) + ")             │");
            console.log("└────────────────────────────────────────────┘");

            // ============================================================================
            // 步骤7: 生成最终的图片元素
            // ============================================================================
            console.log("🔵 [ELEMENT] 创建图片元素:");

            // 获取当前画布中的元素
            const existingElements = api.getSceneElements();
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

            console.log("  - 元素ID:", imageElement.id);
            console.log("  - 元素类型:", imageElement.type);
            console.log("  - 位置 (x, y):", imageElement.x.toFixed(2), imageElement.y.toFixed(2));
            console.log("  - 尺寸 (w, h):", imageElement.width.toFixed(2), imageElement.height.toFixed(2));

            // ============================================================================
            // 步骤8: 插入到画布
            // ============================================================================
            console.log("🔵 [SCENE] 更新画布场景:");
            console.log("  - 插入前元素数量:", existingElements.length);

            // 使用 Excalidraw 原生 API 的 updateScene 方法
            api.updateScene({
              elements: [...existingElements, imageElement],
              files: {
                ...files,
                [fileId]: newFile
              }
            });

            const elementsAfter = api.getSceneElements();
            console.log("  - 插入后元素数量:", elementsAfter.length);
            console.log("  - 新增元素数量:", elementsAfter.length - existingElements.length);

            // ============================================================================
            // 步骤9: 验证插入结果
            // ============================================================================
            const addedElement = elementsAfter.find(el => el.id === elementId);
            if (addedElement) {
                console.log("✅ [SUCCESS] 图标插入成功!");
                console.log("🟢 [VERIFY] 验证插入的元素:");
                console.log("  - ID:", addedElement.id);
                console.log("  - 位置:", { x: addedElement.x.toFixed(2), y: addedElement.y.toFixed(2) });
                console.log("  - 尺寸:", { width: addedElement.width.toFixed(2), height: addedElement.height.toFixed(2) });
                console.log("  - fileId:", addedElement.fileId);

                // 检查位置是否与预期一致
                const positionMatch = Math.abs(addedElement.x - iconX) < 0.01 && Math.abs(addedElement.y - iconY) < 0.01;
                if (positionMatch) {
                    console.log("  ✅ 位置验证通过：图标位置与计算值一致");
                } else {
                    console.warn("  ⚠️  位置警告：图标位置与计算值不完全一致");
                    console.warn("    预期 x:", iconX, "实际 x:", addedElement.x);
                    console.warn("    预期 y:", iconY, "实际 y:", addedElement.y);
                }
            } else {
                console.error("🔴 [ERROR] 元素插入失败：在画布中找不到新插入的元素!");
                throw new Error("元素插入验证失败");
            }

            // 选中新添加的元素
            api.selectElements([imageElement]);

            // 强制刷新画布
            api.refresh();

            console.log("🔵 [INSERT] ========== 图标插入流程完成 ==========");
            new Notice(`图标已插入 (${iconSize.toFixed(0)}x${iconSize.toFixed(0)}) 在 (${iconX.toFixed(0)}, ${iconY.toFixed(0)})`);

        } catch (error) {
            // ============================================================================
            // 错误处理
            // ============================================================================
            console.error("🔴 [ERROR] ========== 图标插入失败 ==========");
            console.error("  错误类型:", error.name);
            console.error("  错误消息:", error.message);
            console.error("  错误堆栈:", error.stack);
            console.error("🔴 [ERROR] 当前状态:");
            console.error("  - point:", JSON.stringify(point, null, 2));
            console.error("  - selectedIcon:", state.selectedIcon);
            console.error("==========================================");

            new Notice(`插入图标失败: ${error.message}`);
        }
    } catch (error) {
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
await main();

// ===== 工具按钮入口 =====
/**
 * Excalidraw 工具按钮点击处理函数
 * 当用户点击工具栏的图标按钮时调用
 */
function action() {
    openSearchPanel();
}
