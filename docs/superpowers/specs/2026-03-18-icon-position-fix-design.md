# 图标插入位置修复设计文档

**项目:** Obsidian Excalidraw 图标搜索插件
**文档类型:** 设计规范
**创建日期:** 2026-03-18
**状态:** 待审核
**优先级:** 高

---

## 📋 文档概述

### 问题描述

当前图标插入功能存在严重的坐标计算问题：
- **100%缩放时**：图标向鼠标点击位置的**右下偏移200+像素**
- **50%缩放时**：图标向鼠标点击位置的**左上偏移**（方向反转）
- **150%缩放时**：图标**完全找不到**（偏移到画布外）

### 根本原因

当前坐标计算公式（方法A）存在逻辑错误：
```javascript
// ❌ 错误的公式
x: point.scrollX + (point.containerX / point.zoom)
```

**问题分析：**
- `scrollX` 已经是**画布坐标系统**中的偏移值
- `containerX / zoom` 转换后也是**画布坐标**
- 两者直接相加导致**双重计算**，产生错误偏移

当缩放比例变化时，错误的程度不同，导致偏移方向和程度都发生变化。

### 解决方案

采用**方案1：纯画布坐标法**，移除错误的 `scrollX` 加法运算。

```javascript
// ✅ 正确的公式
x: point.containerX / point.zoom
y: point.containerY / point.zoom
```

---

## 🎯 设计目标

### 功能目标

1. ✅ 图标中心对准鼠标点击位置
2. ✅ 在所有缩放级别（25%-400%）下保持准确
3. ✅ 在画布滚动后位置仍然准确
4. ✅ 图标大小随缩放等比缩放

### 非功能目标

1. ✅ 提供详细的调试日志，便于问题定位
2. ✅ 添加输入验证和错误处理
3. ✅ 保持代码简洁易维护
4. ✅ 不引入性能开销

---

## 🏗️ 技术设计

### 架构设计

```
┌─────────────────────────────────────────────────────┐
│                   用户点击画布                        │
│                   (鼠标事件)                          │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              getCanvasCoordinates()                   │
│  • 获取容器相对位置 (containerX, containerY)          │
│  • 获取画布缩放比例 (zoom)                            │
│  • 获取滚动偏移 (scrollX, scrollY)                    │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│             insertIconToCanvas()                     │
│                                                       │
│  ┌─────────────────────────────────────────────┐   │
│  │  步骤1: 计算图标尺寸                         │   │
│  │  iconSize = 32 × zoom                        │   │
│  └─────────────────────────────────────────────┘   │
│                       │                              │
│                       ▼                              │
│  ┌─────────────────────────────────────────────┐   │
│  │  步骤2: 验证输入数据                         │   │
│  │  • 检查数值有效性                            │   │
│  │  • 检查是否为有限值                          │   │
│  └─────────────────────────────────────────────┘   │
│                       │                              │
│                       ▼                              │
│  ┌─────────────────────────────────────────────┐   │
│  │  步骤3: 计算安全缩放                         │   │
│  │  safeZoom = clamp(zoom, 0.01, 100)          │   │
│  └─────────────────────────────────────────────┘   │
│                       │                              │
│                       ▼                              │
│  ┌─────────────────────────────────────────────┐   │
│  │  步骤4: 坐标转换 (核心修复)                  │   │
│  │  mouseCanvasX = containerX / safeZoom        │   │
│  │  mouseCanvasY = containerY / safeZoom        │   │
│  └─────────────────────────────────────────────┘   │
│                       │                              │
│                       ▼                              │
│  ┌─────────────────────────────────────────────┐   │
│  │  步骤5: 计算图标左上角位置                   │   │
│  │  iconX = mouseCanvasX - iconSize/2          │   │
│  │  iconY = mouseCanvasY - iconSize/2          │   │
│  └─────────────────────────────────────────────┘   │
│                       │                              │
│                       ▼                              │
│  ┌─────────────────────────────────────────────┐   │
│  │  步骤6: 创建并插入图片元素                   │   │
│  │  api.updateScene({ elements, files })       │   │
│  └─────────────────────────────────────────────┘   │
│                       │                              │
│                       ▼                              │
│  ┌─────────────────────────────────────────────┐   │
│  │  步骤7: 验证插入结果                         │   │
│  │  • 检查元素是否存在                          │   │
│  │  • 验证位置是否正确                          │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 核心算法

#### 坐标转换公式

```javascript
// 输入：鼠标在容器中的位置（屏幕像素）
const containerX = event.clientX - rect.left;
const containerY = event.clientY - rect.top;

// 转换为画布坐标（考虑缩放）
const safeZoom = Math.max(0.01, Math.min(zoom, 100));
const mouseCanvasX = containerX / safeZoom;
const mouseCanvasY = containerY / safeZoom;

// 计算图标左上角（使中心对齐鼠标）
const iconX = mouseCanvasX - (iconSize / 2);
const iconY = mouseCanvasY - (iconSize / 2);
```

**关键点：**
1. ✅ **不再加 scrollX**：避免双重计算
2. ✅ **中心对齐**：减去图标尺寸的一半
3. ✅ **安全缩放**：限制在合理范围

#### 图标尺寸计算

```javascript
// 用户期望的显示尺寸（100%缩放时）
const displaySize = 32;

// 实际存储的尺寸（随缩放调整）
const iconSize = displaySize * zoom;
```

**原理：**
- Excalidraw 的元素尺寸是**画布坐标**中的值
- 当 zoom > 1 时，图标存储更大的尺寸，但视觉上仍然是 32px
- 当 zoom < 1 时，图标存储更小的尺寸，但视觉上仍然是 32px

---

## 📊 数据流设计

### 输入数据

| 字段 | 类型 | 说明 | 来源 |
|------|------|------|------|
| `containerX` | number | 鼠标相对于容器左边的像素距离 | `event.clientX - rect.left` |
| `containerY` | number | 鼠标相对于容器顶部的像素距离 | `event.clientY - rect.top` |
| `zoom` | number | 画布缩放比例 | `api.getAppState().zoom.value` |
| `scrollX` | number | 画布水平滚动偏移 | `api.getAppState().scrollX` |
| `scrollY` | number | 画布垂直滚动偏移 | `api.getAppState().scrollY` |

### 输出数据

| 字段 | 类型 | 说明 | 用途 |
|------|------|------|------|
| `iconX` | number | 图标左上角的画布X坐标 | 创建图片元素 |
| `iconY` | number | 图标左上角的画布Y坐标 | 创建图片元素 |
| `iconSize` | number | 图标的宽高（画布像素） | 创建图片元素 |

### 中间计算

```javascript
// 坐标转换流程
containerX (屏幕像素)
    ↓ ÷ safeZoom
mouseCanvasX (画布坐标)
    ↓ - iconSize/2
iconX (图标左上角)
```

---

## 🛡️ 错误处理设计

### 输入验证

```javascript
// 1. 类型检查
if (typeof point.containerX !== 'number' || typeof point.containerY !== 'number') {
    throw new Error("containerX 或 containerY 不是有效数字");
}

// 2. 范围检查
if (typeof point.zoom !== 'number' || point.zoom <= 0) {
    throw new Error("zoom 不是有效数字或 <= 0");
}

// 3. 有限值检查
if (!isFinite(point.containerX) || !isFinite(point.containerY)) {
    throw new Error("containerX 或 containerY 为无穷大");
}

// 4. 计算结果验证
if (!isFinite(mouseCanvasX) || !isFinite(mouseCanvasY)) {
    throw new Error("计算出的坐标无效");
}
```

### 边界情况处理

| 情况 | 处理方式 | 说明 |
|------|----------|------|
| `zoom <= 0` | 限制到最小值 0.01 | 防止除以0 |
| `zoom > 100` | 限制到最大值 100 | 防止极端值 |
| `containerX/Y` 为 `Infinity` | 抛出错误 | 不允许无穷大 |
| 计算结果为 `NaN` | 抛出错误 | 数值异常 |

### 插入后验证

```javascript
// 验证元素是否成功插入
const addedElement = elementsAfter.find(el => el.id === elementId);
if (!addedElement) {
    throw new Error("元素插入验证失败");
}

// 验证位置是否正确
const positionMatch = Math.abs(addedElement.x - iconX) < 0.01;
if (!positionMatch) {
    console.warn("图标位置与计算值不完全一致");
}
```

---

## 📝 调试日志设计

### 日志分级

| 级别 | 标记 | 用途 | 示例 |
|------|------|------|------|
| INFO | 🔵 | 流程节点 | 函数入口/出口 |
| DATA | 🟢 | 数据值 | 坐标、尺寸 |
| WARN | 🟡 | 警告 | 位置不完全一致 |
| ERROR | 🔴 | 错误 | 计算失败 |

### 日志内容

```
🔵 [INSERT] ========== 开始图标插入流程 ==========
🔵 [INSERT] 选中的图标: weixin

🟢 [SIZE] 计算图标尺寸:
  - 用户期望尺寸 (100%): 32 px
  - 当前画布缩放: 1
  - 实际存储尺寸: 32.00 px

🔵 [VALIDATE] 验证输入数据:
  - containerX: 256 类型: number
  - containerY: 189 类型: number
  - zoom: 1 类型: number

🟢 [COORD] 鼠标画布坐标计算结果:
  - mouseCanvasX: 256.00
  - mouseCanvasY: 189.00

🟢 [POSITION] 图标位置计算:
  - 图标中心对齐 → 需要减去图标尺寸的一半
  - 图标尺寸: 32.00
  - 半尺寸: 16.00
  - iconX: 240.00
  - iconY: 173.00

┌────────────────────────────────────────────┐
│  画布缩放:       1.00 (100%)               │
│  图标尺寸:    32.00px                      │
│  鼠标屏幕坐标: (256, 189)                   │
│  鼠标画布坐标: (256, 189)                   │
│  图标左上角: (240, 173)                     │
│  图标中心点: (256, 189)                     │
└────────────────────────────────────────────┘

✅ [SUCCESS] 图标插入成功!
🔵 [INSERT] ========== 图标插入流程完成 ==========
```

### 使用日志定位问题

**步骤：**
1. 按 F12 打开开发者工具
2. 切换到 Console 标签
3. 点击画布插入图标
4. 复制从 `🔵 [INSERT]` 开始的所有日志
5. 发送日志进行分析

**日志会显示：**
- ✅ 每一步的中间计算值
- ✅ 最终的图标位置
- ✅ 任何错误或警告

---

## 🧪 测试计划

### 单元测试场景

| 测试场景 | 输入条件 | 预期输出 |
|----------|----------|----------|
| 正常坐标 | containerX=256, zoom=1 | iconX=240 |
| 放大坐标 | containerX=256, zoom=2 | iconX=224 |
| 缩小坐标 | containerX=256, zoom=0.5 | iconX=248 |
| 极限缩放 | containerX=256, zoom=100 | iconX≈255 |
| 最小缩放 | containerX=256, zoom=0.01 | iconX≈-24 |

### 集成测试场景

| 场景 | 缩放 | 滚动 | 预期结果 |
|------|------|------|----------|
| 基础插入 | 100% | 无 | 图标中心对准鼠标 |
| 缩小插入 | 50% | 无 | 图标中心对准鼠标 |
| 放大插入 | 150% | 无 | 图标中心对准鼠标 |
| 滚动后插入 | 100% | 有 | 图标中心对准鼠标 |
| 综合测试 | 75% | 有 | 图标中心对准鼠标 |

### 回归测试清单

- [ ] 100%缩放下无滚动时准确
- [ ] 100%缩放下有滚动时准确
- [ ] 50%缩放下无左上偏移
- [ ] 150%缩放下可找到图标
- [ ] 极端缩放（25%、400%）正常
- [ ] 连续多次插入保持准确

---

## 📦 代码修改清单

### 修改文件

**文件:** `Excalidraw/Scripts/Downloaded/iconfont-obsidian-search.md`

**修改范围:** 第 899-940 行

### 修改内容

#### 删除的内容

```javascript
// 删除方法A、B、C的对比代码
const methodA = {
    x: point.scrollX + (point.containerX / point.zoom),
    y: point.scrollY + (point.containerY / point.zoom),
    name: "方法A: scroll + container/zoom"
};

const methodB = {
    x: point.containerX / point.zoom,
    y: point.containerY / point.zoom,
    name: "方法B: container/zoom (忽略scroll)"
};

const methodC = {
    x: point.scrollX - (point.containerX / point.zoom),
    y: point.scrollY - (point.containerY / point.zoom),
    name: "方法C: scroll - container/zoom"
};

const mouseCanvasX = methodA.x;
const mouseCanvasY = methodA.y;

// 删除方向询问提示
console.log("⚠️ 请告诉我：图标相对鼠标位置在哪个方向？");
```

#### 新增的内容

```javascript
// 1. 添加输入数据验证
// 2. 添加安全缩放处理
// 3. 使用纯画布坐标法计算位置
// 4. 添加详细的调试日志
// 5. 添加插入后验证逻辑

// 详见完整代码实现...
```

### 代码统计

| 指标 | 修改前 | 修改后 | 变化 |
|------|--------|--------|------|
| 代码行数 | ~42 行 | ~180 行 | +138 行 |
| 核心逻辑行 | ~10 行 | ~20 行 | +10 行 |
| 调试日志行 | ~15 行 | ~120 行 | +105 行 |
| 注释行数 | ~5 行 | ~40 行 | +35 行 |

**说明：** 虽然总行数增加，但核心逻辑更清晰，且新增了大量调试日志便于问题定位。

---

## 🚀 部署计划

### 修改步骤

1. **备份原文件**
   ```bash
   cp iconfont-obsidian-search.md iconfont-obsidian-search.md.backup
   ```

2. **应用代码修改**
   - 使用 Edit 工具替换第 899-940 行
   - 或手动复制新代码

3. **在 Obsidian 中测试**
   - 重启 Obsidian 或重新加载脚本
   - 在各种缩放级别下测试

4. **收集反馈**
   - 查看控制台日志输出
   - 记录任何异常情况

### 回滚计划

如果修改后问题更严重：

```bash
# 恢复备份
cp iconfont-obsidian-search.md.backup iconfont-obsidian-search.md

# 或使用 git 回滚
git checkout -- iconfont-obsidian-search.md
```

---

## 📚 附录

### A. Excalidraw 坐标系统说明

Excalidraw 使用多层坐标系统：

1. **屏幕坐标**: 浏览器事件系统中的坐标
2. **容器坐标**: 相对于 Excalidraw 容器的坐标
3. **画布坐标**: Excalidraw 内部使用的坐标（考虑缩放）

**转换公式：**
```
屏幕坐标 → 容器坐标: containerX = clientX - rect.left
容器坐标 → 画布坐标: canvasX = containerX / zoom
```

### B. 相关 API 文档

- `api.getAppState()`: 获取画布状态（zoom, scrollX, scrollY）
- `api.updateScene()`: 更新画布场景
- `api.getSceneElements()`: 获取所有元素
- `api.selectElements()`: 选定元素

### C. 问题诊断检查表

如果修改后仍有问题，检查以下项目：

- [ ] 浏览器控制台是否有错误
- [ ] 日志中的坐标值是否合理
- [ ] `zoom` 值是否在有效范围内
- [ ] 容器尺寸是否正确获取
- [ ] 图标尺寸计算是否正确

---

## ✅ 审批清单

- [x] 设计方案已确认
- [x] 技术细节已明确
- [x] 测试计划已制定
- [x] 错误处理已考虑
- [ ] 用户审核
- [ ] 代码审查
- [ ] 实施批准

---

**文档版本:** 1.0
**最后更新:** 2026-03-18
**状态:** ✅ 设计完成，等待审核
