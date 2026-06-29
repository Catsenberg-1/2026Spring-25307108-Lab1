# 过程描述文件 5：3D 骰子渲染引擎

## 1. 功能概述

本次迭代在应用中集成了**3D 骰子**模块，实现了一个纯 Canvas 驱动的三维立方体渲染引擎。骰子可以绕 X/Y/Z 三轴旋转，掷骰动画包含 5 圈自旋 + 缓出效果，骰面采用标准点数布局（1~6），通过画家算法（Z 排序）和背面剔除保证正确的遮挡关系。

这是整个项目中技术密度最高的功能——不依赖任何 3D 库，从旋转矩阵到投影变换全部手写，展示了在 ArkTS Canvas 2D 上下文中实现伪 3D 渲染的完整方案。

## 2. 功能入口

主页面顶栏新增 🎲 按钮，导航体系升级为 4 按钮：

```
[🎲] [🔄] [🖌️] [📚]
 骰子  进制  绘图  历史
```

点击 🎲 → `showDice = true` → 3D 骰子页面。

## 3. 3D 渲染管线

### 3.1 顶点定义

立方体 8 个顶点，以中心为原点，边长 1（半长 0.5）：

```typescript
const origVertices: number[][] = [
  [-0.5, -0.5, -0.5], [0.5, -0.5, -0.5], [0.5, 0.5, -0.5], [-0.5, 0.5, -0.5],  // 背面
  [-0.5, -0.5,  0.5], [0.5, -0.5,  0.5], [0.5, 0.5,  0.5], [-0.5, 0.5,  0.5]   // 前面
];
```

### 3.2 旋转系统

使用欧拉角旋转（X → Y → Z 顺序），每个轴独立的旋转矩阵：

| 旋转轴 | 变换函数 | 矩阵效果 |
|--------|----------|----------|
| X 轴 | `rotateX(p, angle)` | Y-Z 平面旋转，cos/sin 作用于 y, z |
| Y 轴 | `rotateY(p, angle)` | X-Z 平面旋转，cos/sin 作用于 x, z |
| Z 轴 | `rotateZ(p, angle)` | X-Y 平面旋转，cos/sin 作用于 x, y |

```typescript
// 顶点变换流水线
rotatedVertices = origVertices.map(v => {
  let p = rotateX(v, this.diceAngleX);
  p = rotateY(p, this.diceAngleY);
  p = rotateZ(p, this.diceAngleZ);
  return p;
});
```

每次重绘（约 60fps）都重新计算全部 8 个顶点的旋转后坐标，然后重建 6 个面的屏幕投影。

### 3.3 背面剔除（Backface Culling）

不渲染背对相机的面，通过法线点积判断：

```typescript
let n = face.normal;                    // 面的法向量（模型空间）
let rn = rotateZ(rotateY(rotateX(n,    // 旋转法向量到当前姿态
  this.diceAngleX), this.diceAngleY), this.diceAngleZ);
let dot = rn[0]*viewDir[0] +           // 与视线方向 [0,0,-1] 点积
          rn[1]*viewDir[1] + rn[2]*viewDir[2];
if (dot > 0.1) { ... }                  // 正面 → 渲染
```

| dot 值 | 含义 | 行为 |
|--------|------|------|
| > 0.1 | 面朝向相机 | ✅ 渲染 |
| ≤ 0.1 | 面背对相机 | ❌ 剔除 |

### 3.4 画家算法（Painter's Algorithm）

对通过背面剔除的面按 Z 深度排序（远→近），实现正确遮挡：

```typescript
sortedFaces.sort((a, b) => b.avgZ - a.avgZ);
```

`avgZ` 是面 4 个顶点旋转后 Z 坐标的平均值。Z 值越大（越靠近相机），渲染顺序越靠后（后绘制的覆盖先绘制的）。

### 3.5 正投影（Orthographic Projection）

```typescript
function project(p: number[], canvasW: number, canvasH: number,
                 d: number, scale: number): ProjectedPoint {
  let sx = p[0] * scale + canvasW / 2;    // X → 屏幕 X（居中）
  let sy = canvasH / 2 - p[1] * scale;    // Y → 屏幕 Y（翻转 + 居中）
  return { x: sx, y: sy, factor: 1 };
}
```

使用正交投影（非透视），缩放因子 `scale = 150`，骰子居中绘制。

### 3.6 点数渲染

每个面上的骰子点数通过**双线性插值**定位：

```typescript
// 面定义中的 dots 使用 UV 坐标 (0~1)
face2: { dots: [[0.3,0.8],[0.7,0.8],[0.3,0.5],[0.7,0.5],[0.3,0.2],[0.7,0.2]] }  // 6 点

// 双线性插值：UV → 3D 坐标
x = (1-u)*(1-v)*v0[0] + u*(1-v)*v1[0] + u*v*v2[0] + (1-u)*v*v3[0];
y = (1-u)*(1-v)*v0[1] + u*(1-v)*v1[1] + u*v*v2[1] + (1-u)*v*v3[1];
z = (1-u)*(1-v)*v0[2] + u*(1-v)*v1[2] + u*v*v2[2] + (1-u)*v*v3[2];
```

插值后的 3D 坐标再经 `project()` 转为屏幕坐标，绘制为半径 10px 的实心圆（黑色填充 + 白色描边）。

### 3.7 骰面点数定义

| 面 | 法向量 | 点数 | UV 布局 |
|----|--------|------|---------|
| 1 | [0, 0, 1] | 1 | 中心单点 (0.5, 0.5) |
| 2 | [0, 0, -1] | 6 | 两列三行 |
| 3 | [1, 0, 0] | 2 | 对角线 |
| 4 | [-1, 0, 0] | 5 | 四角 + 中心 |
| 5 | [0, 1, 0] | 3 | 对角线 |
| 6 | [0, -1, 0] | 4 | 四角 |

点数布局严格遵循真实骰子标准（对⾯和为 7：1↔6, 2↔5, 3↔4）。

## 4. 掷骰动画系统

### 4.1 动画状态机

```
IDLE (diceRunning=false)
  │
  │ 用户点击 [掷骰子]
  ▼
ROLLING (diceRunning=true)
  │ 随机选面 faceIndex (1~6)
  │ targetAngle = currentAngle + 1800° + standardOffset
  │ setInterval(16ms) 启动 60fps 循环
  │
  │ 每帧: easeOutCubic(t) → 插值角度 → drawDice()
  │
  ▼
DONE (t >= 1, diceRunning=false)
  diceResult = faceIndex
  drawDice() 最后一次绘制（停在结果面）
```

### 4.2 缓动函数

```typescript
let easedT = 1 - Math.pow(1 - t, 3);    // Cubic ease-out
```

| t | easedT | 视觉效果 |
|---|--------|----------|
| 0.0 | 0.000 | 静止 |
| 0.2 | 0.488 | 快速启动 |
| 0.5 | 0.875 | 高速旋转 |
| 0.8 | 0.992 | 缓慢减速 |
| 1.0 | 1.000 | 精确停在目标角度 |

Cubic ease-out 曲线模拟了物理骰子的惯性衰减——初期快速翻滚，末期缓慢趋近停止，视觉上比线性过渡更自然。

### 4.3 目标角度计算

```typescript
let faceIndex = Math.floor(Math.random() * 6) + 1;     // 随机 1~6
let base = this.standardAngles.get(faceIndex)!;

this.targetAngleX = this.diceAngleX + 1800 + base.ax;   // 5 圈自旋
this.targetAngleY = this.diceAngleY + 1800 + base.ay;
this.targetAngleZ = this.diceAngleZ + 1800 + base.az;
```

- `1800°` = 5 圈完整旋转，保证视觉上的充分翻滚
- `base.ax/ay/az` = 标准角度偏移，确保停止时目标面朝向相机

### 4.4 标准角度表

```typescript
private standardAngles = new Map<number, StandardAngleEntry>([
  [1, { ax: 0,   ay: 180, az: 0   }],
  [2, { ax: 0,   ay: 0,   az: 0   }],
  [3, { ax: 0,   ay: 90,  az: 0   }],
  [4, { ax: 0,   ay: -90, az: 0   }],
  [5, { ax: -90, ay: 0,   az: 0   }],
  [6, { ax: 90,  ay: 0,   az: 0   }]
]);
```

每个面的标准角度是该面法向量 [0,0,1] 旋转到指定方向所需的欧拉角。由于始终从当前角度增量（而非固定角度），连续投掷的角度不断累加，视觉上不会出现重复的旋转姿态。

### 4.5 动画参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 帧间隔 | 16ms (~60fps) | `setInterval` 定频 |
| 动画时长 | 2000ms | 2 秒 |
| 旋转圈数 | 5 圈 (1800°) | 每轴独立 |
| 缓动函数 | Cubic ease-out | `1-(1-t)³` |
| 骰子缩放 | 150 | 像素缩放因子 |

## 5. 骰子页面 UI

```
┌──────────────────────────────┐
│  [←]   3D 骰子               │
├──────────────────────────────┤
│                              │
│        ┌─────────┐           │
│        │  ◻ ◻   │           │  ← 实时 3D 渲染
│        │    ◻    │           │
│        │  ◻ ◻   │           │
│        └─────────┘           │
│       Canvas (layoutWeight:1)│
│                              │
│     ┌────────────────┐       │
│     │    掷骰子       │       │  ← 橙金渐变按钮（旋转时置灰）
│     └────────────────┘       │
└──────────────────────────────┘
```

按钮在旋转期间（`diceRunning = true`）切换为灰色半透明样式，防止重复点击导致的动画冲突。

## 6. 新增接口定义

```typescript
interface DiceNormalFace { n: number[]; val: number; }
interface DiceFaceData {
  indices: number[];       // 顶点索引（0~7）
  dots: number[][];        // 点数 UV 坐标
  normal: number[];        // 法向量
}
interface SortedDiceFace {
  face: DiceFaceData;
  avgZ: number;            // 平均 Z 深度
  vertices: number[][];    // 旋转后的顶点
}
interface ProjectedPoint {
  x: number; y: number;    // 屏幕坐标
  factor: number;          // 保留（透视扩展预留）
}
interface StandardAngleEntry {
  ax: number; ay: number; az: number;  // 标准角度
}
```

## 7. 新增状态字段

```typescript
@State showDice: boolean = false;              // 骰子页面可见性
@State diceRunning: boolean = false;           // 动画播放中
@State diceResult: number = 0;                 // 掷出结果 (0=未掷)
@State diceAngleX: number = 0;                 // 当前 X 旋转角
@State diceAngleY: number = 0;                 // 当前 Y 旋转角
@State diceAngleZ: number = 0;                 // 当前 Z 旋转角
private targetAngleX/Y/Z: number = 0;          // 目标角度
private diceAnimTimer: number = -1;            // setInterval ID
private diceCtx: CanvasRenderingContext2D;      // 骰子画布上下文
private standardAngles: Map<number, StandardAngleEntry>; // 标准角度表
```

## 8. 生命周期管理

离开骰子页面时调用 `clearDiceTimers()`：

```typescript
clearDiceTimers(): void {
  if (this.diceAnimTimer !== -1) {
    clearInterval(this.diceAnimTimer);
    this.diceAnimTimer = -1;
  }
  this.diceRunning = false;
  this.diceResult = 0;
}
```

避免页面切换后仍有时钟回调操作已卸载组件的 @State。

## 9. 进制模块修复

本次同步修复了进制换算中的一个 UX 问题：切换进制后面板未自动激活为编辑面板。在 `BaseOption` 的 `onClick` 中新增 `this.activeInput = 1/2`，确保用户切换进制后可直接编辑当前面板。

## 10. 组件层级树（新增骰子模块）

```
showDice = true
└── Column (全屏, #000000)
    ├── Row (顶栏)
    │   ├── Button [←]
    │   │   .onClick → showDice = false + clearDiceTimers()
    │   └── Text '3D 骰子'
    ├── Stack → Canvas (diceCtx, layoutWeight:1)
    │   .onReady → drawDice()
    └── Button [掷骰子]
        ├── 空闲: 橙金渐变
        └── 旋转中: 灰色半透明 + 禁用点击
```

## 11. 动画帧时序

```
Frame 0:   drawDice() 初始姿态
Frame 1:   t≈0.016, easedT≈0.047,  角度≈当前+85°
Frame 15:  t≈0.24,  easedT≈0.56,   角度≈当前+1008°
Frame 30:  t≈0.48,  easedT≈0.86,   角度≈当前+1548°
Frame 60:  t≈0.96,  easedT≈1.0,    角度≈目标
Frame 62:  t≥1.0,   clearInterval,  diceRunning=false, diceResult=faceIndex
```

每帧完整执行：顶点旋转 → 面排序 → 背面剔除 → 投影 → 填充 → 描边 → 点数绘制。在 60fps 下每帧允许的时间预算为 16ms，纯 Canvas 2D 计算在 ArkTS 环境下轻松达标。

## 12. 功能演进总览

| 能力 | V1 | V2 | V3 | V4 | V5 | **V6** |
|------|----|----|----|----|----|--------|
| 计算器基本功能 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 函数绘图 | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| 进制换算 | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **3D 骰子** | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |
| 3D 旋转矩阵 | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |
| 背面剔除 | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |
| 画家算法 Z 排序 | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |
| 缓出动画 | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |

## 13. 后续优化建议

1. **触控旋转**：在骰子静止时，用户可手动拖拽旋转查看各面
2. **物理模拟**：引入速度和角速度变量，使用重力+摩擦力做更真实的减速
3. **多骰子**：支持同时掷 2~5 个骰子并排显示
4. **透视投影**：替换正交投影为透视投影（增加 Z 深度缩放），增强立体感
5. **纹理贴图**：替代纯白面+黑点，支持自定义图片作为骰面纹理
6. **音效**：骰子落地时播放碰撞音效
7. **历史记录**：记录最近 10 次掷骰结果并展示统计分布
