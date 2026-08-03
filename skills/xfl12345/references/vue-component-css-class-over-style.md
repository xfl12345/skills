---
name: vue-component-css-class-over-style
description: 在编写或重构 Vue 3 组件时使用——用 CSS class prop 替代 `propsCssStyle` JS 对象，用 CSS 模块（CSS modules）+ `v-bind()` 取代内联样式。在组件通过 prop 传递 `CSSProperties` 或模板中存在内联 `style` 属性时触发。
---

# Vue 组件优先使用 CSS Class 而非 Style

## 概述

**禁止通过 prop 把 CSS 作为 JS 对象传递。内部元素定制用 CSS class prop，响应式动态值用 CSS 模块 + `v-bind()`。**

## 核心规则

### 1. 禁止 `propsCssStyle`

`propsCssStyle?: CSSProperties` 被禁用。替换方式取决于它原来作用于哪个元素：

**根元素** → 直接移除该 prop。使用方可以在组件标签上直接写 `class` 或 `style`。

**内部元素** → 增加一个具名 class prop（`xxxClass?: string`），并在目标元素上用 `:class` 绑定。

```vue
<!-- ❌ BAD：通过 prop 用 JS 对象传递 CSS -->
<script setup lang="ts">
import type { CSSProperties } from 'vue'

interface Props {
  propsCssStyle?: CSSProperties
}
const props = withDefaults(defineProps<Props>(), {
  propsCssStyle: () => ({ backgroundColor: 'white' }),
})
</script>
<template>
  <div :style="props.propsCssStyle">...</div>
</template>

<!-- ✅ GOOD：CSS class prop（内部元素场景）-->
<script setup lang="ts">
defineProps<{ borderClass?: string }>()
</script>
<template>
  <div :class="[$style.circleBorder, borderClass]">...</div>
</template>
<style module>
.circleBorder {
  background-color: white;
}
</style>
```

### 2. Class Prop 命名

prop 名称按它绑定的元素命名，而不是按 CSS 属性命名：

```typescript
// ❌ BAD：按 CSS 属性命名
backgroundColorClass?: string
shadowStyleClass?: string

// ✅ GOOD：按绑定元素命名
buttonBodyClass?: string      // 绑定到按钮主体元素
buttonBorderClass?: string    // 绑定到边框/阴影元素
centerPictureClass?: string   // 绑定到中间图片区域
```

### 3. 禁止 `:style` 与内联样式

所有 `style="..."`、`:style="..."` 和 `:style="someComputed"` 都必须替换为 `<style module>` 类 + `v-bind()`。静态值直接写死，动态值用 `v-bind()`。

**注意：** `v-bind()` 的值为 `undefined` 或 `null` 时会产生空的 CSS 自定义属性，浏览器会忽略它——相当于该 CSS 声明被移除。因此 computed 对象不需要给每个字段都赋值，未设置字段保持 `undefined` 即可。

**单字段** → 用简单 `v-bind(name)`：

```vue
<!-- ❌ BAD -->
<script setup lang="ts">
import { computed, type CSSProperties } from 'vue'

const props = defineProps<{ boxSize: number }>()
const boxStyle = computed<CSSProperties>(() => ({ borderRadius: props.boxSize + 'px' }))
</script>
<template>
  <div :style="[boxStyle]" />
</template>

<!-- ✅ GOOD -->
<script setup lang="ts">
const props = defineProps<{ boxSize: number }>()
</script>
<template>
  <div :class="$style.box" />
</template>
<style module>
.box { border-radius: calc(v-bind(boxSize) * 1px); }
</style>
```

**多字段** → 保留 computed，用 `v-bind('obj.field')` 逐个字段绑定：

```vue
<!-- ❌ BAD -->
<script setup lang="ts">
import { computed, reactive, type CSSProperties } from 'vue'

const state = reactive({ strokeColor: 'black', strokeWidth: 2, duration: 0.3 })
const circleStyle = computed<CSSProperties>(() => ({
  stroke: state.strokeColor,
  strokeWidth: state.strokeWidth,
  transitionDuration: state.duration + 's',
}))
</script>
<template>
  <circle :style="circleStyle" />
</template>

<!-- ✅ GOOD -->
<script setup lang="ts">
import { computed, reactive, type CSSProperties } from 'vue'

const state = reactive({ strokeColor: 'black', strokeWidth: 2, duration: 0.3 })
const circleStyle = computed<CSSProperties>(() => ({
  stroke: state.strokeColor,
  strokeWidth: state.strokeWidth,
  transitionDuration: state.duration + 's',
}))
</script>
<template>
  <circle :class="$style.filling" />
</template>
<style module>
.filling {
  stroke: v-bind('circleStyle.stroke');
  stroke-width: v-bind('circleStyle.strokeWidth');
  transition-duration: v-bind('circleStyle.transitionDuration');
}
</style>
```

### 4. `calc(v-bind() * 1px)` 与 JS 字符串 computed 的取舍

单值单位转换优先用 CSS `calc(v-bind(numericRef) * 1px)`，而不是 JS 字符串拼接 computed。这样能让 JS 里保持数值类型，单位交给 CSS 处理。

```typescript
import { computed, type CSSProperties } from 'vue'

const props = defineProps<{ theFontSizeInPixel: number }>()

// ❌ BAD：单值用 JS 字符串拼接
const theFontSize = computed<CSSProperties['fontSize']>(() => props.theFontSizeInPixel + 'px')
```

```css
/* ✅ GOOD：JS 保持数值，单位交给 CSS */
font-size: calc(v-bind(theFontSizeInPixel) * 1px);
```

**阈值取决于表达式复杂度：**

**简单单位转换**（`* 1px`，无额外运算）—— CSS 中重复 **2 次** 可以接受；达到 **3 次及以上** 时提取为 computed：

```css
/* ✅ OK：简单 * 1px 重复两次——无需提取 */
.icon { width: calc(v-bind(size) * 1px); }
.thumb { height: calc(v-bind(size) * 1px); }
```

```typescript
import { computed, ref, type CSSProperties } from 'vue'

const qrCodeHeightInPixel = ref(100)
// ✅ GOOD：简单转换出现 3+ 次，值得提取为 computed
const qrCodeHeight = computed<CSSProperties['height']>(() => qrCodeHeightInPixel.value + 'px')
```

```css
.qrCode {
  width: v-bind(qrCodeHeight);
  height: v-bind(qrCodeHeight);
}
.middleBox { height: v-bind(qrCodeHeight); }
```

**非简单计算**（额外乘数如 `* 1.25`、Math 函数、除法等）—— 重复 **2+ 次** 就提取。重复复杂的 calc 表达式会隐藏语义关系，并且容易漂移：

```css
/* ❌ BAD：同一非简单 calc 重复——难以看出是同一个值 */
.titleBar { font-size: calc(v-bind(theFontSizeInPixel) * 1.25 * 1px); }
.sloganBar { font-size: calc(v-bind(theFontSizeInPixel) * 1.25 * 1px); }
```

```typescript
import { computed, type CSSProperties } from 'vue'

const props = defineProps<{ theFontSizeInPixel: number }>()

// ✅ GOOD：已提取——关系显式、单一真相源
const titleFontSize = computed<CSSProperties['fontSize']>(() => props.theFontSizeInPixel * 1.25 + 'px')
```

```css
.titleBar { font-size: v-bind(titleFontSize); }
.sloganBar { font-size: v-bind(titleFontSize); }
```

**例外：Math 函数保留在 JS computed 中**（即使只用一次）。`Math.floor`、`Math.ceil` 等没有 CSS 等价写法——数值中间结果必须留在 JS 里：

```typescript
import { computed } from 'vue'

const props = defineProps<{ theFontSizeInPixel: number }>()

// ✅ GOOD：Math.floor 留在 JS 里，供 CSS 和其他 JS computed 共用
const quarterFontSizeInPixel = computed<number>(() => Math.floor(props.theFontSizeInPixel / 4))
```

### 5. 多值 CSS 属性

像 `margin`、`padding`、`box-shadow` 这类简写属性，如果每个值都用 `calc(v-bind * 1px)` 会非常冗长，保留 JS 字符串 computed 以提升可读性：

```typescript
import { computed, type CSSProperties } from 'vue'

const props = defineProps<{ theFontSizeInPixel: number }>()
type CssTypeFontSize = CSSProperties['fontSize']
const oneEighthFontSize = computed<CssTypeFontSize>(() => `${props.theFontSizeInPixel / 8}px`)
const quarterFontSize = computed<CssTypeFontSize>(() => `${props.theFontSizeInPixel / 4}px`)

// ✅ GOOD：多值场景——JS computed 更清爽
const rootMargin = computed<string>(() =>
  [oneEighthFontSize.value, quarterFontSize.value, quarterFontSize.value, quarterFontSize.value].join(' ')
)
const boxShadow = computed<string>(() => {
  const fSize = oneEighthFontSize.value
  return [fSize, fSize, fSize, '0', 'gray'].join(' ')
})
```

```css
.root { margin: v-bind(rootMargin); }
.card { box-shadow: v-bind(boxShadow); }
```

### 6. `v-bind()` 点号写法必须加引号

简单标识符不加引号即可。点号写法**必须**使用字符串形式：

```css
/* ✅ 简单标识符——不加引号 */
width: v-bind(boxWidth);

/* ❌ 点号写法没加引号 */
stroke: v-bind(circleStyle.stroke);

/* ✅ 字符串形式 */
stroke: v-bind('circleStyle.stroke');
```

### 7. 静态条件样式 → CSS Class 绑定

CSS 值是**静态**的、但按条件应用时，用 Vue 的条件 class 绑定即可，不需要 computed 或 v-bind()：

```vue
<!-- ❌ BAD：computed 返回静态值或 undefined，再绑定到样式 -->
<script setup lang="ts">
import { computed, reactive, type CSSProperties } from 'vue'

const settingsStore = reactive({ devMode: false })

const debugBorderAqua = computed<CSSProperties['border']>(() =>
  settingsStore.devMode ? '1px dashed aqua' : undefined
)
</script>
<template>
  <div :class="$style.cvBox" :style="{ border: debugBorderAqua }">
</template>
<!-- 更差：把静态值塞进 CSS 的 v-bind 里（而非用 class）-->
<style module>
.cvBox { border: v-bind(debugBorderAqua); }
</style>

<!-- ✅ GOOD：静态 CSS class + 条件绑定 -->
<script setup lang="ts">
import { reactive } from 'vue'

const settingsStore = reactive({ devMode: false })
</script>
<template>
  <div :class="[$style.cvBox, { [$style.cvBoxDebug]: settingsStore.devMode }]">
</template>
<style module>
.cvBoxDebug { border: 1px dashed aqua; }
</style>
```

**原因：** CSS 值本身是静态的——运行时不变，只变“是否应用”。这种情况下 CSS class 是正确抽象；v-bind() 用于动态值。

### 8. 布局状态切换 → CSS Class 名

在多个完整布局模式之间切换时（例如居中 vs 左对齐），用 `useCssModule()` + 返回 CSS 模块键名的 computed，而不是返回 CSSProperties 对象的 computed：

```vue
<!-- ❌ BAD：computed CSSProperties + 每个属性单独 v-bind('obj.field') -->
<script setup lang="ts">
import { computed, ref, type CSSProperties } from 'vue'

const shouldCenter = ref(true)

const cvBoxParentStyle = computed<CSSProperties>(() => {
  if (shouldCenter.value) {
    return { justifyContent: 'center', width: '100%', height: '100%' }
  }
  return { justifyContent: 'flex-start' }
})
</script>
<template>
  <div :class="$style.cvBoxParent">
</template>
<style module>
.cvBoxParent {
  justify-content: v-bind('cvBoxParentStyle.justifyContent');
  width: v-bind('cvBoxParentStyle.width');
  height: v-bind('cvBoxParentStyle.height');
}
</style>

<!-- ✅ GOOD：CSS class + useCssModule() 保证类型安全 -->
<script setup lang="ts">
import { computed, ref, useCssModule } from 'vue'

const shouldCenter = ref(true)

const cssModule = useCssModule()
type CssModuleKey = keyof typeof cssModule
const cvBoxParentLayoutClass = computed<CssModuleKey>(() => {
  return shouldCenter.value
    ? cssModule.cvBoxParentFoxy
    : cssModule.cvBoxParentDefault
})
</script>
<template>
  <div :class="[$style.cvBoxParent, cvBoxParentLayoutClass]">
</template>
<style module>
.cvBoxParent { display: flex; }
.cvBoxParentFoxy { justify-content: center; width: 100%; height: 100%; }
.cvBoxParentDefault { justify-content: flex-start; }
</style>
```

**原因：** 所有值都是静态的——只有“选哪一个”是动态的。CSS class 能干净表达这种选择。`useCssModule()` 为 class 名字符串提供编译期类型安全。

## 危险信号

- `propsCssStyle` 或 `propsStyle` 出现在 Props 接口中
- `xxxStyle?: CSSProperties` 出现在 Props 接口中——禁止通过 prop 传递 CSS
- 模板中存在 `:style="..."`
- 非根元素上出现 `style="..."` 静态内联属性
- `v-bind(obj.field)` 没有加引号
- 简单 `calc(v-bind(x) * 1px)` 重复 3+ 次——应提取为 JS 字符串 computed
- 非简单 `calc(v-bind(x) * N * 1px)` 重复 2+ 次——应提取为 JS 字符串 computed
- JS 字符串 computed（`x + 'px'`）只用一次——应改用 CSS `calc(v-bind(x) * 1px)`
- computed 返回静态 CSS 值或 `undefined` 做条件样式——应改用 CSS class + `{ [$style.xxx]: condition }`
- 用 computed CSSProperties 做布局状态切换——应改用 `useCssModule()` + CSS class 名

注意：把 `computed<CSSProperties>` 当作类型标注是正确的 TypeScript 实践——它不是危险信号。只有 `CSSProperties` 作为 **prop 类型** 时才被禁止。

## 常见错误

| 错误 | 修正 |
|---------|-----|
| 对静态值使用 v-bind | 在 `<style module>` 中写死静态值 |
| class prop 按 CSS 属性命名 | 按绑定元素命名 |
| `v-bind(obj.field)` 没有引号 | 使用字符串形式：`v-bind('obj.field')` |
| 单用途 JS `x + 'px'` computed | 改用 CSS `calc(v-bind(x) * 1px)` |
| 简单 `calc(v-bind(x) * 1px)` 重复 3+ 次 | 提取为 JS 字符串 computed |
| 非简单 `calc(v-bind(x) * N * 1px)` 重复 2+ | 提取为 JS 字符串 computed |
| 把 `Math.floor` / `Math.ceil` 丢给 CSS calc | 保留在 JS computed 中 |
| 静态条件样式用 computed（`x ? '1px dashed' : undefined`） | CSS class + `{ [$style.xxx]: condition }` |
| 布局状态切换用 computed CSSProperties | `useCssModule()` + 每个状态一个 CSS class 名 |
