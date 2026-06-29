# 过程描述文件 6：大小屏适配与自由流转

## 1. 版本概述

本次迭代聚焦应用的多设备体验升级。主要完成两项核心能力：(1) **大小屏适配** — 使计算器在手机和平板两种形态上均能提供合理的交互体验，通过百分比布局、ConstraintSize 约束和药丸形按键设计，消除平板端的按钮溢出问题；(2) **自由流转** — 基于 HarmonyOS Continuation 机制，实现计算器状态在手机与平板间的无缝接续，用户可在设备间自由切换而不丢失计算上下文。

## 2. 大小屏适配

### 2.1 问题分析

原始计算器按键使用 `.width('21%').aspectRatio(1)` 的正圆形设计。在 360vp 宽的手机上，每个按钮约为 75.6vp，视觉比例合理。但在 900vp 宽的平板上，按钮膨胀至 189vp——接近手机端 2.5 倍——不仅溢出屏幕边界，也因触控目标过大而降低了操作效率。

| 屏幕 | 宽度 | 按钮宽 (21%) | 问题 |
|------|------|-------------|------|
| 手机 | 360vp | 75.6vp | ✅ 适中 |
| 平板竖屏 | 600vp | 126vp | ⚠️ 偏大 |
| 平板横屏 | 900vp | 189vp | ❌ 溢出 |

### 2.2 解决方案：ConstraintSize 上限截断

```typescript
.width('23%')
.height(50)
.borderRadius(25)
.constraintSize({ maxWidth: 180 })
```

**三层防护机制**：

| 层级 | 机制 | 手机端 (360vp) | 平板端 (900vp) |
|------|------|---------------|----------------|
| 宽度 | `23%` | 82.8vp | 207vp（未截断前） |
| 截断 | `maxWidth: 180` | 不触发（82.8 < 180） | 触发 → 180vp |
| 高度 | 固定 `50` vp | 50vp | 50vp |
| 圆角 | `25`（半高） | 胶囊形 | 胶囊形 |

手机端百分比计算值在限幅以下，行为不变；平板端触发截断，按钮停止在 180vp，四个按钮 + 间距 = 180×4 + 间隙 ≈ 720+vp，在 900vp 屏幕内完全可见且留有余白。

### 2.3 药丸形按键设计

从正圆形改为横向药丸形，宽高比从 1:1 提升为约 3.6:1（平板端）或 1.66:1（手机端）：

```
改前：    [  O  ]  [  O  ]  [  O  ]  [  O  ]    ← aspectRatio(1) 正圆
改后：    [  ○───  ]  [  ○───  ]  [  ○───  ]  [  ○───  ]    ← 胶囊形
```

设计意图：
- 横向拉伸增加触控面积宽度，降低误触率
- 纵向保持紧凑（50vp），多行按键之间间距舒适
- `borderRadius(25)` 恰好为半高，产生标准胶囊轮廓

### 2.4 百分比布局的天然自适应

按键区域使用 `FlexAlign.SpaceBetween` + 百分比宽度的组合：

```typescript
Row() {
  ForEach(row, (item, colIndex) => {
    Button() { ... }
    .width('23%').height(50)
  })
}
.width('100%')
.justifyContent(FlexAlign.SpaceBetween)
```

`SpaceBetween` 将剩余空间均分于按钮之间，无论屏幕宽度如何变化，间距总是一致且视觉均匀。显示区使用 `layoutWeight(1)` 弹性占据按键区之上的所有剩余高度，平板上的显示区自然更高，展示更多历史表达式内容。

### 2.5 所有页面统一适配

| 页面 | 按键数/行 | 约束 | 适配方式 |
|------|-----------|------|----------|
| 主计算器 | 5行 × 4键 | maxWidth: 180 | constraintSize 截断 |
| 函数绘图 | 8行 × 4键 | maxWidth: 180 | 同上 |
| 进制换算 | 5行 × 4键 | maxWidth: 180 | 同上（Btn Builder） |
| 3D 骰子 | 1按钮 + Canvas | 无 | 自适应 Canvas |
| 历史记录 | 列表项 | 无 | 百分比宽度 |

所有按键均通过 `replace_all` 统一替换为同一套适配参数，保证了全局 UI 一致性。

## 3. 自由流转（Cross-Device Continuation）

### 3.1 功能概述

自由流转是 HarmonyOS 的核心分布式能力之一。用户可在手机上开始一次计算任务（包括复杂的函数绘图和历史操作），然后通过系统流转控键将应用无缝迁移至平板，继续之前的计算会话——表达式、计算结果、历史记录乃至函数曲线状态全部保留。

### 3.2 实现架构

```
┌─────────────┐         ┌──────────────────┐         ┌─────────────┐
│   手机端      │  onContinue  │   Continuation    │  onCreate    │   平板端      │
│  Index.ets  │ ───────→  │    Framework      │ ──────→ │  Index.ets  │
│             │   wantParam │                  │  want    │             │
│ saveContinu-│             │                  │          │ restoreCont- │
│ ationState()│             └──────────────────┘          │ inuationState│
└─────────────┘                                          └─────────────┘
```

### 3.3 模块配置 (`module.json5`)

```json5
{
  "deviceTypes": ["phone", "tablet", "2in1"],  // 允许安装到平板和二合一设备
  "abilities": [{
    "continuable": true                         // 开启流转能力
  }]
}
```

`deviceTypes` 扩展为三种设备类型是流转的前提——应用必须同时安装于两台设备上才能触发流转。

### 3.4 EntryAbility 流转生命周期

```typescript
// ===== 保存状态（源设备） =====
onContinue(wantParam: Record<string, Object>): AbilityConstant.OnContinueResult {
  let storage = LocalStorage.getShared();
  let savedState = storage.get<string>('continuationState');
  // 解析 JSON → 逐键写入 wantParam
  let stateObj = JSON.parse(savedState) as Record<string, Object>;
  for (let key of Object.keys(stateObj)) {
    wantParam[key] = stateObj[key];
  }
  return AbilityConstant.OnContinueResult.AGREE;  // 同意流转
}

// ===== 恢复状态（目标设备） =====
onCreate(want: Want, launchParam: AbilityConstant.LaunchParam) {
  if (launchParam.launchReason === AbilityConstant.LaunchReason.CONTINUATION) {
    let savedState = want.parameters;            // 读取 want 参数
    LocalStorage.getShared()!.setOrCreate(
      'continuationState', JSON.stringify(savedState)  // 写入 LocalStorage
    );
  }
}
```

流转数据通过 `wantParam` (Record<string, Object>) 传递，不依赖外部存储或网络，数据安全且低延迟。

### 3.5 Index 页面状态序列化

**保存 (`saveContinuationState`)** — 在每次按 `=` 或 `AC` 后自动触发：

| 状态字段 | 类型 | 说明 |
|----------|------|------|
| `expression` | string | 当前输入表达式 |
| `result` | string | 计算结果 |
| `historyExpr` | string | 上一条历史表达式 |
| `justCalculated` | string | 刚计算标志 ('1'/'0') |
| `historyData` | JSON | 最近 5 条历史记录 |
| `curvesData` | JSON | 函数曲线集合 |
| `xMin/xMax/yMin/yMax` | string | 图形视口参数 |

**恢复 (`restoreContinuationState`)** — 在 `aboutToAppear` 中自动执行，从 `LocalStorage.getShared()` 读取 `continuationState` 键值并反序列化回所有 `@State` 变量。

### 3.6 流转触发方式

用户可通过以下任一方式触发流转：
1. **控制中心流转按钮** — 从屏幕顶部下滑控制中心，点击流转图标
2. **多设备任务中心** — 从底部上滑进入任务卡片，拖拽应用到目标设备
3. **碰一碰** — NFC 触碰两台设备触发流转（需硬件支持）

### 3.7 数据安全考量

- 流转数据仅在设备间点对点传输（通过系统分布式软总线），不经过云端
- `wantParam` 大小受系统限制（约 100KB），仅传输关键计算状态而非全量数据
- 历史记录仅保留最近 5 条以控制传输体积

## 4. 设备类型注册对应用分发的影响

`deviceTypes: ["phone", "tablet", "2in1"]` 的影响：

| 阶段 | 影响 |
|------|------|
| 编译 | 无影响，所有设备类型共用同一份 HAP |
| 签名 | 需使用同时授权 phone + tablet 的证书 Profile |
| 分发 | AppGallery 会在手机和平板两个分类中同时展示 |
| 安装 | 用户可在手机和平板同时安装 |
| 运行 | 应用内 `deviceTypes` 声明与当前设备匹配时正常启动 |

## 5. 新增/修改文件清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `entry/src/main/module.json5` | 修改 | `deviceTypes` 扩展 + `continuable: true` |
| `entry/src/main/ets/entryability/EntryAbility.ets` | 重写 | 新增 `onContinue` 和流转感知 `onCreate` |
| `entry/src/main/ets/pages/Index.ets` | 修改 | 按键形状改为药丸形 + `constraintSize` + 状态序列化/反序列化 |

## 6. 按键 UI 演进

| 版本 | 形状 | 宽×高 (手机) | 宽×高 (平板) | 圆角 |
|------|------|-------------|-------------|------|
| V5 (此前) | 正圆形 | 75.6×75.6vp | 189×189vp | 50% |
| **V6 (本次)** | **药丸形** | **83×50vp** | **180×50vp** | **25vp** |

## 7. 功能演进总览

| 能力 | V1 | V2 | V3 | V4 | V5 | **V6** |
|------|----|----|----|----|----|--------|
| 计算器基本功能 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 函数绘图 + 进制换算 | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| 3D 骰子 | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **平板适配 (ConstraintSize)** | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |
| **药丸形按键** | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |
| **自由流转 (Continuation)** | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |
| **多设备类型声明** | ❌ | ❌ | ❌ | ❌ | ❌ | **✅** |

## 8. 后续优化建议

1. **流转内容预览**：在流转确认对话框中显示当前计算表达式的缩略内容
2. **流转恢复动画**：目标设备打开时播放从流转图标展开的过渡动画
3. **自适应列数**：在超宽屏（> 1200vp）上将按键从 4 列改为 5 列，利用更多横向空间
4. **分屏模式**：平板端在分屏/悬浮窗下自动缩小按键以适应更小的窗口
5. **流转统计日志**：记录流转次数和用户设备配对环境，优化后续体验
