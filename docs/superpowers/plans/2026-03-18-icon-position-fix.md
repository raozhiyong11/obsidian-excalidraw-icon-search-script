# 图标插入位置修复实施计划 v2

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标:** 修复 Excalidraw 图标插入位置偏移问题，使图标中心精确对准鼠标点击位置，在所有缩放级别（25%-400%）下保持准确。

**架构:** 采用纯画布坐标法，移除错误的 scrollX 加法运算。将屏幕坐标（containerX）转换为画布坐标（除以 zoom），然后计算图标左上角位置（中心对齐）。

**技术栈:** JavaScript, Excalidraw Automate API, Obsidian Plugin API

---

## 📋 快速参考

**修改文件:** `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`
**修改行数:** 第 890-940 行（共 51 行）
**关键修改:** 移除 `scrollX + (containerX / zoom)` 改为 `containerX / zoom`

---

## 🚨 回滚程序（重要！）

**如果任何步骤失败，立即执行：**

```bash
cd "y:\develop_space\claude_code_workspace\obsidian-excalidraw-icon-search-plugin"
# 方法1：恢复备份
cp "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md.backup" "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md"

# 方法2：git 回滚
git checkout -- "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md"
```

---

## 阶段1：实施（Tasks 1-4）

### Task 1: 预检查和备份

**目的:** 确保环境安全，创建备份

- [ ] **Step 1: 验证文件存在**

```bash
test -f "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md" && echo "✅ OK" || echo "❌ FAIL"
```

Expected: ✅ OK

- [ ] **Step 2: 创建备份**

```bash
cp "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md" "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md.backup"
```

Expected: 无输出

- [ ] **Step 3: 验证备份**

```bash
diff "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md" "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md.backup"
```

Expected: 无差异（文件相同）

- [ ] **Step 4: 提交备份**

```bash
git add "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md.backup"
git commit -m "chore: 备份脚本文件，准备修复图标位置问题"
```

Expected: Commit 成功

---

### Task 2: 读取并确认当前代码

**目的:** 确认需要修改的代码位置

- [ ] **Step 1: 读取第 890-940 行**

使用 Read 工具读取 `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md:890-940`

Expected: 应该看到方法A/B/C代码和调试日志

- [ ] **Step 2: 确认关键代码**

检查以下内容是否存在：
- `// 计算图标位置和大小` (第 890 行)
- `const methodA = {` (第 900 行)
- `const mouseCanvasX = methodA.x;` (第 921 行)
- `console.log("========================");` (第 940 行)

Expected: 所有代码都存在

---

### Task 3: 替换代码（核心修改）

**目的:** 使用 Edit 工具替换有问题的坐标计算代码

**old_string（第 890-940 行，共 51 行）：**
```javascript
        // 计算图标位置和大小
        // 核心逻辑：图标永远出现在鼠标点击位置，大小随缩放调整

        // 1. 计算图标显示大小（随画布缩放）
        // 用户要求32px，这是画布100%时的尺寸
        const displaySize = 32; // 用户看到的图标大小
        const iconSize = displaySize * point.zoom; // 实际存储的尺寸（随缩放）

        // 2. 尝试多种坐标计算方法，找出正确的
        // 方法A: 当前使用的公式
        const methodA = {
            x: point.scrollX + (point.containerX / point.zoom),
            y: point.scrollY + (point.containerY / point.zoom),
            name: "方法A: scroll + container/zoom"
        };

        // 方法B: 简单使用容器坐标（不考虑滚动）
        const methodB = {
            x: point.containerX / point.zoom,
            y: point.containerY / point.zoom,
            name: "方法B: container/zoom (忽略scroll)"
        };

        // 方法C: 减法
        const methodC = {
            x: point.scrollX - (point.containerX / point.zoom),
            y: point.scrollY - (point.containerY / point.zoom),
            name: "方法C: scroll - container/zoom"
        };

        // 暂时使用方法A
        const mouseCanvasX = methodA.x;
        const mouseCanvasY = methodA.y;

        // 图标左上角（使图标中心对齐鼠标）
        const iconX = mouseCanvasX - (iconSize / 2);
        const iconY = mouseCanvasY - (iconSize / 2);

        console.log("=== 图标插入信息（调试） ===");
        console.log("画布缩放:", point.zoom);
        console.log("图标大小:", displaySize, "px → 实际", iconSize.toFixed(2), "px");
        console.log("滚动偏移:", { x: point.scrollX.toFixed(2), y: point.scrollY.toFixed(2) });
        console.log("鼠标容器位置:", { x: point.containerX.toFixed(2), y: point.containerY.toFixed(2) });
        console.log("🔍 坐标方法对比:");
        console.log("  " + methodA.name, { x: methodA.x.toFixed(2), y: methodA.y.toFixed(2) });
        console.log("  " + methodB.name, { x: methodB.x.toFixed(2), y: methodB.y.toFixed(2) });
        console.log("  " + methodC.name, { x: methodC.x.toFixed(2), y: methodC.y.toFixed(2) });
        console.log("当前使用方法A，图标位置:", { x: iconX.toFixed(2), y: iconY.toFixed(2) });
        console.log("⚠️ 请告诉我：图标相对鼠标位置在哪个方向？（左上/右上/左下/右下）");
        console.log("========================");
```

**new_string（新的代码，共 218 行）：**
```javascript
        // ============================================================================
        // 开始图标插入坐标计算
        // ============================================================================
        console.log("🔵 [INSERT] ========== 开始图标插入流程 ==========");
        console.log("🔵 [INSERT] 选中的图标:", state.selectedIcon?.name || "未找到");

        try {
            // ============================================================================
            // 步骤1: 计算图标显示大小（随画布缩放）
            // ============================================================================
            const displaySize = 32; // 用户看到的图标大小（100%缩放时）
            const iconSize = displaySize * point.zoom; // 实际存储的尺寸（随缩放）

            console.log("🟢 [SIZE] 计算图标尺寸:");
            console.log("  - 用户期望尺寸 (100%):", displaySize, "px");
            console.log("  - 当前画布缩放:", point.zoom);
            console.log("  - 实际存储尺寸:", iconSize.toFixed(2), "px");
            console.log("  - 尺寸计算公式: 32 ×", point.zoom, "=", iconSize.toFixed(2));

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
```

- [ ] **Step 1: 执行 Edit 替换**

使用 Edit 工具执行上述替换（old_string → new_string）

Expected: Edit 成功，无报错

- [ ] **Step 2: 验证替换结果**

读取第 890-950 行，确认新代码已插入

Expected: 应该看到 🔵 🟢 🔴 emoji 日志标记

- [ ] **Step 3: 提交代码**

```bash
git add "Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md"
git commit -m "fix: 修复图标插入位置偏移问题

- 采用纯画布坐标法，移除错误的 scrollX 加法运算
- 修复 100% 缩放下向右下偏移的问题
- 修复 50% 缩放下向左上偏移的问题
- 修复 150% 缩放下找不到图标的问题
- 添加详细的调试日志（使用 emoji 标记）
- 添加输入验证和错误处理
- 添加插入后验证逻辑"
```

Expected: Commit 成功

---

## 阶段2：测试（Tasks 4-6）

### Task 4: 基础功能测试（100%缩放）

**目的:** 验证基本功能正常

| 步骤 | 操作 | 预期结果 | 状态 |
|------|------|----------|------|
| 1 | 打开 Obsidian | 正常启动 | [ ] |
| 2 | 打开 Excalidraw 画布 | 画布显示正常 | [ ] |
| 3 | 按 F12 打开控制台 | 控制台可见 | [ ] |
| 4 | 点击图标搜索按钮 | 搜索面板打开 | [ ] |
| 5 | 搜索"微信"并选择图标 | 放置模式激活 | [ ] |
| 6 | 在画布中心点击（100%缩放） | 图标插入 | [ ] |
| 7 | 检查控制台日志 | 有 🔵🟢✅ 标记 | [ ] |
| 8 | 目视检查图标位置 | 中心对准鼠标 | [ ] |

**验收标准:**
- ✅ 图标中心对准鼠标点击位置（偏差 < 10 像素）
- ✅ 控制台输出完整日志，无 🔴 ERROR
- ✅ Notice 显示成功消息

---

### Task 5: 缩放级别测试矩阵

**目的:** 测试不同缩放级别

| 缩放级别 | 预期图标尺寸 | 预期位置 | 状态 |
|---------|------------|---------|------|
| 50% | 16px | 对准鼠标，无左上偏移 | [ ] |
| 100% | 32px | 对准鼠标，无偏移 | [ ] |
| 150% | 48px | 对准鼠标，可找到 | [ ] |
| 200% | 64px | 对准鼠标 | [ ] |

**测试步骤（每个缩放级别重复）：**
1. 调整画布缩放到指定级别
2. 点击画布插入图标
3. 目视检查图标位置
4. 检查控制台日志中的 zoom 值

---

### Task 6: 滚动场景测试

**目的:** 测试有滚动偏移的情况

| 步骤 | 操作 | 预期结果 | 状态 |
|------|------|----------|------|
| 1 | 滚动画布 | 画布有 scrollX/scrollY | [ ] |
| 2 | 100%缩放下插入 | 图标对准鼠标 | [ ] |
| 3 | 50%缩放下插入 | 图标对准鼠标 | [ ] |
| 4 | 检查日志中的 scroll 值 | 有非零值显示 | [ ] |

---

## 阶段3：文档和报告（Tasks 7-8）

### Task 7: 更新实施计划

- [ ] **Step 1: 在计划末尾添加测试结果**

```markdown
## 测试结果（实际执行后填写）

### 通过的测试
- [ ] 100% 缩放：✅ 图标中心对准鼠标
- [ ] 50% 缩放：✅ 无左上偏移
- [ ] 150% 缩放：✅ 图标可找到
- [ ] 200% 缩放：✅ 图标准确对准
- [ ] 滚动场景：✅ 位置准确

### 发现的问题
（记录任何未解决的问题）
```

- [ ] **Step 2: 提交更新**

```bash
git add "docs/superpowers/plans/2026-03-18-icon-position-fix.md"
git commit -m "docs: 更新实施计划，添加实际测试结果"
```

---

### Task 8: 创建实施报告

- [ ] **Step 1: 创建报告文件**

```bash
cat > "docs/superpowers/plans/2026-03-18-implementation-report.md" << 'EOF'
# 图标位置修复实施报告

**日期:** 2026-03-18
**状态:** ✅ 完成

## 问题描述
- 100% 缩放：图标向右下偏移 200+ 像素
- 50% 缩放：图标向左上偏移（方向反转）
- 150% 缩放：图标完全找不到

## 解决方案
采用纯画布坐标法：`canvasX = containerX / zoom`

## 代码变更
- 文件：`Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`
- 行数：第 890-940 行
- 删除：51 行（方法A/B/C对比代码）
- 新增：218 行（含调试日志）

## 测试结果
（根据实际测试结果填写）
EOF
```

- [ ] **Step 2: 提交报告**

```bash
git add "docs/superpowers/plans/2026-03-18-implementation-report.md"
git commit -m "docs: 添加图标位置修复实施报告"
```

---

## ✅ 完成检查清单

### 代码修改
- [ ] 备份已创建
- [ ] 代码已替换（890-940行）
- [ ] 代码已提交

### 功能测试
- [ ] 100% 缩放准确
- [ ] 50% 缩放无左上偏移
- [ ] 150% 缩放可找到
- [ ] 滚动场景准确

### 日志验证
- [ ] 🔵 [INSERT] 开始标记存在
- [ ] 🟢 [SIZE] 尺寸计算正确
- [ ] 🟢 [COORD] 坐标转换正确
- [ ] ✅ [SUCCESS] 成功标记存在

### 文档
- [ ] 测试结果已记录
- [ ] 实施报告已创建

---

## 📞 问题诊断

**如果图标位置仍然不准确：**

1. **复制完整控制台日志**（从 🔵 [INSERT] 到 ✅ [SUCCESS]）
2. **检查关键数值：**
   - zoom 值是否与实际缩放一致？
   - mouseCanvasX 是否 = containerX / zoom？
   - iconX 是否 = mouseCanvasX - iconSize/2？
3. **发送日志进行分析**

---

**文档版本:** 2.0
**最后更新:** 2026-03-18
**状态:** ✅ 已通过计划审查，准备执行
