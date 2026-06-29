# 基于 ArkTS 的卡片计算器 — 工程描述文档

## 1. 项目概述

本项目基于华为 HarmonyOS ArkTS 框架，实现了一款卡片式（Widget）计算器应用。应用以服务卡片形态部署于桌面，用户无需进入完整应用即可完成四则运算，充分体现鸿蒙「轻量化交互」与「原子化服务」的设计理念。

| 属性 | 值 |
|------|-----|
| Bundle Name | `com.samples.arktscalc` |
| 应用版本 | 1.0.0 (versionCode: 1000000) |
| 目标 SDK | HarmonyOS 5.0.5(17) |
| 运行环境 | HarmonyOS 5.0.5 Release 及以上 |
| 支持设备 | 华为手机（标准系统） |
| 开发工具 | DevEco Studio 5.0.5 Release+ |
| UI 框架 | ArkTS（声明式 UI） |
| 扩展形态 | FormExtensionAbility（4×4 服务卡片） |
| 开源协议 | Apache License 2.0 |

## 2. 工程目录结构

```
calculator-master/
├── AppScope/                         # 应用全局配置
│   ├── app.json5                     # 应用包名、版本号、图标等元信息
│   └── resources/                    # 全局资源
├── entry/                            # 主模块（Entry Module）
│   ├── src/main/
│   │   ├── ets/
│   │   │   ├── calc/pages/
│   │   │   │   └── CardCalc.ets      # ★ 核心：卡片计算器页面
│   │   │   ├── entryability/
│   │   │   │   └── EntryAbility.ets  # UIAbility 入口
│   │   │   ├── entryformability/
│   │   │   │   └── EntryFormAbility.ets  # 卡片生命周期管理
│   │   │   ├── model/
│   │   │   │   └── Logger.ts         # 日志工具（hilog 封装）
│   │   │   └── pages/
│   │   │       └── Index.ets         # 首页（卡片相同实现）
│   │   ├── module.json5              # 模块配置（Ability / Extension 注册）
│   │   └── resources/                # 模块资源（字符串、颜色、媒体）
│   ├── build-profile.json5           # 模块构建配置
│   └── oh-package.json5              # 模块包描述
├── hvigor/                           # hvigor 构建插件
├── build-profile.json5               # 工程级构建配置
├── hvigorfile.ts                     # hvigor 构建入口
├── oh-package.json5                  # 工程级包描述
└── screenshots/                      # 效果截图
```

## 3. 架构设计

### 3.1 整体架构

```
┌─────────────────────────────────────────┐
│              桌面 / 负一屏                │
│         ┌───────────────┐                │
│         │   卡片 (4×4)   │                │
│         │  CardCalc.ets │                │
│         └───────┬───────┘                │
│                 │ formBindingData         │
│         ┌───────▼────────┐                │
│         │ FormExtension  │                │
│         │   Ability      │                │
│         └───────┬────────┘                │
│                 │                        │
│         ┌───────▼────────┐                │
│         │   UIAbility    │                │
│         │  (EntryAbility) │               │
│         └────────────────┘                │
│              HarmonyOS                   │
└─────────────────────────────────────────┘
```

### 3.2 核心模块说明

| 模块 | 文件 | 职责 |
|------|------|------|
| 卡片 UI 层 | `CardCalc.ets` | 计算器按键布局、输入状态管理、表达式实时渲染 |
| 算式解析引擎 | `CardCalc.ets` (calc 函数族) | 中缀表达式 → 后缀表达式 → 数值计算 |
| 应用入口 | `EntryAbility.ets` | UIAbility 生命周期管理，加载 Index 页面 |
| 卡片生命周期 | `EntryFormAbility.ets` | 卡片添加/更新/删除的事件回调 |
| 日志工具 | `Logger.ts` | 基于 hilog 的统一日志输出 |

## 4. 算式解析引擎详解

计算核心采用经典的中缀转后缀（逆波兰表示法）算法流水线：

```
输入字符串 → parseInfixExpression()  →  toSuffixExpression()  →  calcSuffixExpression()  →  结果
    (中缀)       (中缀 token 数组)           (后缀 token 数组)          (数值计算)
```

### 4.1 中缀解析 `parseInfixExpression()`

- **功能**：将原始输入字符串拆分为 token 数组（操作数 / 运算符 / 括号）
- **关键处理**：识别一元正负号（表达式首字符或 `(` 后的 `+`/`-` 视为符号而非运算符）

### 4.2 中缀转后缀 `toSuffixExpression()`

- **功能**：使用操作符栈，按优先级将中缀表达式转为逆波兰表示
- **运算符优先级**：`*` / `/` > `+` / `-`（优先级值 1 > 0）
- **括号处理**：左括号直接入栈，遇右括号弹栈至匹配左括号

### 4.3 后缀求值 `calcSuffixExpression()`

- **功能**：遍历后缀表达式，操作数入栈，遇运算符弹二算一再入栈
- **浮点精度控制**：通过 `getFloatNum()` 动态计算小数位数，使用 `Number.toFixed()` 控制精度
- **大数处理**：结果超过 15 位时自动转为科学计数法显示

### 4.4 运算精度策略

| 运算 | 精度策略 |
|------|----------|
| 加法 / 减法 | 取两操作数最大小数位数 |
| 乘法 | 取两操作数小数位数之和 |
| 除法 | max(被除数小数位 + 除数位数, 3) |

## 5. UI 布局说明

卡片采用垂直列布局（Column），自上而下包含：

```
┌─────────────────────┐
│    表达式显示区       │  ← Stack / Text (20%)
├─────────────────────┤
│  [C]  [÷]  [×] [⌫] │  ← Row 1 (16%)
├─────────────────────┤
│  [7]  [8]  [9] [-] │  ← Row 2 (16%)
├─────────────────────┤
│  [4]  [5]  [6] [+] │  ← Row 3 (16%)
├─────────────────────┤
│  [1]  [2]  [3] [0] │  ← Row 4 (16%)
├─────────────────────┤
│  [  .  ]  [  =  ]  │  ← Row 5 (16%) ★ 新增
└─────────────────────┘
```

- 按键背景色：功能键（前三列）`#33007DFF`（半透明蓝），数字键 `#F0F0F0`（浅灰）
- 数字 0 按钮宽度为 2.5 倍标准按钮（`aspectRatio: 2.5`）
- 卡片设计宽度 720px，自适应宽度

## 6. 新功能：小数点输入支持

### 6.1 变更概述

原版卡片计算器仅支持整数的四则运算。本次迭代新增**小数点输入功能**，使计算器具备完整的小数运算能力，覆盖更广泛的实际计算场景（如金额计算、科学测量、百分比换算等）。

### 6.2 UI 变更

在原 4 行按键布局基础上，新增第 5 行操作栏：

| 按钮 | 位置 | 宽度占比 | 样式 | 说明 |
|------|------|----------|------|------|
| 小数点 `.` | 左侧 | 50% | 黑字粉底 (`Color.Pink`) | 点击输入小数点 |
| 等号 `=` | 右侧 | 50% | 图标 / 灰底 | 执行计算并显示结果 |

等号按钮从原布局中独立出来，与小数点按钮并列，优化了交互动线——输入完成后直接点击同行右侧等号，操作路径更短。

### 6.3 输入逻辑设计

小数点输入处理位于 `onInputValue()` 方法中，遵循以下状态机规则：

```
输入 '.' 事件
    │
    ├─[1] 定位当前表达式中的最后一个操作数
    │     从表达式末尾向前扫描，找到最近的操作符或 '(' 位置
    │
    ├─[2] 重复输入保护
    │     若当前操作数已包含 '.'，则忽略本次输入
    │     （杜绝 "3..14" 等非法表达式的产生）
    │
    ├─[3] 自动补零（Leading-Zero Normalization）
    │     若当前操作数为空（如表达式末尾为操作符或表达式为空）
    │     则自动补 "0." → 符合数学书写规范
    │     （如：输入 "0." 而非 "."，保证表达式合法性）
    │
    └─[4] 实时求值反馈
          拼接 '.' 后立即调用 calc() 执行实时预计算
          （尽管未完成的表达式可能无法得出最终结果，但保证 UI 状态一致）
```

### 6.4 精度扩展影响

小数点的引入触发了运算精度的全面升级。在原有 `OPERATORHANDLERS` 中，各运算符均通过 `getFloatNum()` 动态计算保留小数位数，确保：

- 加法/减法：结果小数位 = 两操作数最大小数位数
- 乘法：结果小数位 = 两操作数小数位数之和
- 除法：结果小数位 ≥ 3，避免精度丢失

此设计保证小数点输入后的计算结果准确且显示简洁——不会出现浮点运算常见的 `0.1 + 0.2 = 0.30000000000000004` 问题。

### 6.5 关键代码片段

```typescript
// 小数点输入处理逻辑
} else if (value === '.') {
  let expr = this.expression;
  let lastNumberStart = 0;
  // [1] 定位当前操作数起点
  for (let i = expr.length - 1; i >= 0; i--) {
    if (isOperator(expr[i]) || expr[i] === '(' || expr[i] === ')') {
      lastNumberStart = i + 1;
      break;
    }
  }
  let lastNumber = expr.substring(lastNumberStart);
  // [2] 重复小数点保护
  if (lastNumber.indexOf('.') !== -1) {
    return;
  }
  // [3] 自动补零规范化
  if (lastNumber.length === 0) {
    this.expression += '0.';
  } else {
    this.expression += '.';
  }
  // [4] 实时求值
  this.result = calc(this.expression);
}
```

## 7. 构建与运行

```bash
# 环境要求
- DevEco Studio ≥ 5.0.5 Release
- HarmonyOS SDK ≥ 5.0.5 Release
- 签名证书（调试用自动签名即可）

# 构建步骤
1. 使用 DevEco Studio 打开工程根目录
2. File → Sync and Refresh Project
3. Build → Build Hap(s) / APP(s)
4. Run → Run 'entry' 到真机或模拟器
5. 长按应用图标 → 添加卡片到桌面
```

## 8. 附录：文件清单

| 序号 | 文件路径 | 大小 | 说明 |
|------|----------|------|------|
| 1 | `AppScope/app.json5` | 495 B | 应用全局配置 |
| 2 | `entry/src/main/module.json5` | 1.6 KB | 模块注册配置 |
| 3 | `entry/src/main/ets/calc/pages/CardCalc.ets` | 12.8 KB | **核心计算器页面** |
| 4 | `entry/src/main/ets/pages/Index.ets` | 12.8 KB | 首页（与卡片同实现） |
| 5 | `entry/src/main/ets/entryability/EntryAbility.ets` | 1.5 KB | 应用入口 |
| 6 | `entry/src/main/ets/entryformability/EntryFormAbility.ets` | 1.4 KB | 卡片扩展 Ability |
| 7 | `entry/src/main/ets/model/Logger.ts` | 927 B | 日志工具 |
| 8 | `entry/src/main/resources/base/profile/form_config.json` | 434 B | 卡片配置（4×4） |
| 9 | `build-profile.json5` | 495 B | 工程构建配置 |
| 10 | `oh-package.json5` | 230 B | 工程包描述 |
| 11 | `LICENSE` | 9.5 KB | Apache 2.0 许可证 |
