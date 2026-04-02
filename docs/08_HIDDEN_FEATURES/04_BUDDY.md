# 08.4 - BUDDY 宠物系统

> 基于用户 ID 的确定性宠物生成：物种、稀有度、属性与 ASCII 精灵动画

## 1. 学习目标

学完本章后，你将理解：
- BUDDY 宠物的核心数据结构：Bones 与 Soul 分离设计
- 物种生成算法（哈希 + PRNG）与外观决定机制
- 20 种物种、稀有度权重、五维属性系统
- Sprite 动画系统与交互渲染
- AI 上下文注入机制

## 2. 核心内容

### 2.1 核心概念：骨骼（Bones）与灵魂（Soul）

| 概念 | 说明 | 持久化 |
|------|------|--------|
| **Bones** | 物种、稀有度、眼睛、帽子、闪光、属性 | ❌ 每次从 hash(userId) 重新生成 |
| **Soul** | 名字、性格 | ✅ 首次 hatch 时生成，存储在配置中 |

这种设计确保：
- 用户无法通过编辑配置伪造稀有度
- 物种重命名不会破坏已有的宠物

### 2.2 物种表（20种）

| 物种 | 特征 |
|------|------|
| duck | 经典鸭子 |
| goose | 鹅 |
| blob | 果冻状生物 |
| cat | 猫咪 |
| dragon | 小龙 |
| octopus | 章鱼 |
| owl | 猫头鹰 |
| penguin | 企鹅 |
| turtle | 乌龟 |
| snail | 蜗牛 |
| ghost | 幽灵 |
| axolotl | 蝾螈 |
| capybara | 水豚 |
| cactus | 仙人掌 |
| robot | 机器人 |
| rabbit | 兔子 |
| mushroom | 蘑菇 |
| chonk | 圆胖生物 |

**字符编码保护**：物种名使用 `String.fromCharCode` 动态构建，避免与 canary 检测冲突：

```typescript
const c = String.fromCharCode
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
```

### 2.3 稀有度系统

| 稀有度 | 权重 | 属性下限 | 显示符号 |
|--------|------|----------|----------|
| common | 60% | 5 | ★ |
| uncommon | 25% | 15 | ★★ |
| rare | 10% | 25 | ★★★ |
| epic | 4% | 35 | ★★★★ |
| legendary | 1% | 50 | ★★★★★ |

**属性分配规则**：
- 1 个峰值属性：稀有度下限 + 50~80
- 1 个 dump 属性：稀有度下限 - 10~15
- 其余 3 个：稀有度下限 + 0~40

### 2.4 五维属性

| 属性名 | 含义 |
|--------|------|
| DEBUGGING | 调试能力 |
| PATIENCE | 耐心程度 |
| CHAOS | 混乱倾向 |
| WISDOM | 智慧 |
| SNARK | 吐槽指数 |

### 2.5 外观定制

**眼睛样式**（6种）：
```
·  ✦  ×  ◉  @  °
```

**帽子类型**（8种）：
```
none | crown | tophat | propeller | halo | wizard | beanie | tinyduck
```
> 注意：common 稀有度必定无帽（hat: 'none'）

**闪光（Shiny）**：1% 概率触发

### 2.6 生成算法

**Mulberry32 PRNG**：轻量级确定性随机数生成器

```typescript
function mulberry32(seed: number): () => number {
  let a = seed >>> 0
  return function () {
    a = (a + 0x6d2b79f5) | 0
    let t = Math.imul(a ^ (a >>> 15), 1 | a)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
```

**生成流程**：
1. `hashString(userId + SALT)` → 种子
2. `mulberry32(种子)` → PRNG 函数
3. 依次 roll：稀有度 → 物种 → 眼睛 → 帽子 → 闪光 → 属性

**缓存机制**：500ms sprite tick、每次按键、每次 observer 检查都调用 `roll(userId)`，结果会被缓存。

### 2.7 Sprite 渲染系统

#### ASCII 精灵图

每个物种有 3 帧动画（5行 × 12字符）：

```
# duck 帧0
            '
    __      '
  <(· )___  '
   (  ._>   '
    `--´    '
```

`{E}` 占位符会在渲染时替换为实际眼睛样式。

#### 动画序列

```typescript
// 空闲序列：大部分休息(帧0)，偶尔 fidget(帧1-2)，罕见眨眼
const IDLE_SEQUENCE = [0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]
// -1 表示在帧0上眨眼（眼睛替换为 '-'）
```

#### 宠物互动（/buddy pet）

触发后 2.5 秒内显示爱心动画：

```
   ♥    ♥
  ♥  ♥   ♥
 ♥   ♥  ♥
♥  ♥      ♥
·    ·   ·
```

### 2.8 发布时间与功能开关

| 日期 | 规则 |
|------|------|
| 2026年4月1-7日 | 预告窗口（Teaser）：彩虹色 `/buddy` 提示显示在启动通知 |
| 2026年4月起 | `isBuddyLive()` 返回 `true`，命令永久可用 |

**功能检测**：
```typescript
export function isBuddyLive(): boolean {
  if ("external" === 'ant') return true; // 内部版本始终启用
  const d = new Date()
  return d.getFullYear() > 2026 ||
         (d.getFullYear() === 2026 && d.getMonth() >= 3)
}
```

### 2.9 输入框高亮

当用户在输入框输入 `/buddy` 时，会触发特殊高亮效果：

```typescript
export function findBuddyTriggerPositions(text: string): Array<{ start: number; end: number }> {
  if (!feature('BUDDY')) return []
  const re = /\ /buddy\b/g
  // 返回匹配位置，供 PromptInput 渲染高亮
}
```

### 2.10 AI 上下文注入

宠物介绍会作为 attachment 注入到 AI 上下文中：

```typescript
export function companionIntroText(name: string, species: string): string {
  return `# Companion

A small ${species} named ${name} sits beside the user's input box and occasionally
comments in a speech bubble. You're not ${name} — it's a separate watcher.

When the user addresses ${name} directly (by name), its bubble will answer.
Your job in that moment is to stay out of the way: respond in ONE line or less...`
}
```

## 3. 关键源文件

| 功能 | 文件 |
|------|------|
| 宠物类型定义 | `src/buddy/types.ts` |
| 核心生成逻辑 | `src/buddy/companion.ts` |
| ASCII 精灵图 | `src/buddy/sprites.ts` |
| React 渲染组件 | `src/buddy/CompanionSprite.tsx` |
| AI 上下文注入 | `src/buddy/prompt.ts` |
| 启动通知与高亮 | `src/buddy/useBuddyNotification.tsx` |
| 命令注册 | `src/commands.ts` |

## 4. 本章小结

- BUDDY 系统基于用户 ID 的确定性哈希生成宠物外观和属性
- Bones（外观）从 hash 重新生成，Soul（名字性格）持久化存储
- 20 种物种、5 级稀有度、6 种眼睛、8 种帽子提供丰富定制
- Mulberry32 PRNG 确保相同 userId 总是生成相同宠物
- 3 帧 ASCII 精灵动画配合空闲序列实现自然的 idle 动画
- 2026 年 4 月起 `/buddy` 命令永久可用，输入框支持高亮

---

## 下一步

← [03_ULTRAPLAN.md](03_ULTRAPLAN.md)
→ [05_ORCHESTRATION.md](05_ORCHESTRATION.md)
