---
name: typescript
description: TypeScript 编码规范。编写任何 TypeScript 源码时都应触发。
---

# TypeScript

## Coding Practices

### 类型系统

- 对象类型、多类型的变量，尽量在定义时就指定类型。
- 因为 `as` 断言会让编译器检查降级，所以除非 `as const` 尽量少用 `as` 断言：

  ```typescript
  import { z } from 'zod'

  // schema 即类型源头：运行时校验 + 静态类型一次搞定，单一真源
  const UserSchema = z.object({
    id: z.number(),
    name: z.string(),
  })
  type User = z.infer<typeof UserSchema>

  const raw = '{"id":1,"name":"Ann"}'
  const data: unknown = JSON.parse(raw)

  // ❌ BAD: as 直接断言，编译器闭嘴——形状错了运行时才炸
  const badUser = data as User
  const badInput = document.querySelector('#app') as HTMLInputElement

  // ✅ GOOD：让类型「从源头正确」，编译器全程参与校验
  const user2: User = { id: 2, name: 'Ben' }
  const input = document.querySelector<HTMLInputElement>('#app') // 泛型从源头带类型

  // ❌ BAD：在顶层 as，产物会外泄到后续代码，骗编译器的影响面扩大
  const badRecord = data as Record<string, unknown>

  // ✅ GOOD：不信任的外部数据（JSON / 接口返回）首选 zod：safeParse 校验通过才拿到 User
  const parsed = UserSchema.safeParse(data)
  if (parsed.success) {
    parsed.data // User
  }

  //   手写类型守卫也行（无 zod 依赖时），但值类型要查全——只查键的话 x is User 就是半句谎话，等于换个地方骗编译器
  function isUser(x: unknown): x is User {
    if (typeof x !== 'object' || x === null) return false
    const o = x as Record<string, unknown> // 守卫内部受控的 as，不外泄污染到外部世界
    return typeof o.id === 'number' && typeof o.name === 'string'
  }
  const maybeUser = isUser(data) ? data : null

  // ✅ 例外：as const 把字面量「收紧」成字面量类型，是收紧不是放松，放心用（配合 satisfies 见下条）
  const config = { role: 'admin', level: 1 } as const
  ```

- 类型注解（Type Annotation）不允许直接指定对象类型时，用 `satisfies` 增强类型检查。

  ```typescript
  import type { RouteNamedMap } from 'vue-router/auto-routes'
  export const ROUTER_NAMES = {
    INDEX: '/',
    ADMIN: '/admin',
    LOGIN: '/login',
    PLAYGROUND_INDEX: '/playground/',
    ROUTES: '/routes',
  } as const satisfies Record<string, keyof RouteNamedMap>

  export interface Foo { a: number, b: boolean, c: string }
  export const DEFAULT_FOO = { a: 1, b: true, c: 'Hello' } as const satisfies Foo
  ```

- 除非万不得已，严禁 `any`。尽量避免 `unknown`，优先提供确切的类型。`unknown` 仅用于不信任数据的边界，接住即收窄，勿偷懒当作万能类型。
- 善用 `enum` 做枚举，并从中取常量数组（`Object.values()` / `Object.keys()`）。

  ```typescript
  // TheConst.ts
  export enum NginxNjsDirective {
    JS_IMPORT = 'js_import',
    JS_PRELOAD_OBJECT = 'js_preload_object',
    JS_ACCESS = 'js_access',
  }
  export const NGINX_NJS_DIRECTIVES = [...Object.values(NginxNjsDirective)] as const
  export const NGINX_NJS_DIRECTIVE_SET = new Set(NGINX_NJS_DIRECTIVES) as ReadonlySet<NginxNjsDirective>
  ```

- 尽量复用已有类型，善用 `Omit` / `Pick` / `Exclude` / `Extract` / `Parameters` / `ReturnType` / `InstanceType` / `Required` / `Partial` / `Readonly` / `Record` 等的工具类型做类型体操。
- object key 有范围约束的时候，尽量使用 `keyof` 一致性推导，避免宽松匹配
- 如遇深层结构、过于复杂的类型时，尽量使用 `type-fest` 完成类型体操
- 除非表示 `undefined` 类型，只要是作为值用途，一律用 `void 0` 代替 `undefined`：

  ```typescript
  // ❌ BAD: 作「值」用 undefined——可被局部变量遮蔽（`let undefined = 1` 就废了）
  const badFallback = undefined
  if (value === undefined) {
  }

  // ✅ GOOD: void 0 永远是安全的 undefined，不受遮蔽影响
  const fallback = void 0
  if (value === void 0) {
  }

  // ✅ 例外：作「类型」用时照写 undefined
  type Status = 'idle' | 'done' | undefined
  ```

- 空值兜底用 `??`，禁用 `||`（`||` 会把 `0` / `''` / `false` 当假值，经典坑 `count || 10` 当 count=0）：

  ```typescript
  const count = 0
  const label = ''
  const enabled = false

  // ❌ BAD: || 把 0 / '' / false 都当假值，真实值被兜底吞掉
  const badCount = count || 10       // 10，丢了真实的 0
  const badLabel = label || 'anon'   // 'anon'，丢了真实的 ''
  const badFlag = enabled || true    // true，丢了真实的 false

  // ✅ GOOD: ?? 只在 null / undefined 时兜底，0 / '' / false 原样保留
  const safeCount = count ?? 10      // 0
  const safeLabel = label ?? 'anon'  // ''
  const safeFlag = enabled ?? true   // false
  ```

### 模块导入

**通用原则**（适用任何使用路径别名的项目；为避免循环引用，内部资源导入必须尽量具体）：

- TS 上下文导入 TS 文件应当省略 `.ts` 后缀：

  ```typescript
  // ❌ BAD: TS 上下文里导入 TS 文件带 `.ts` 后缀
  import { foo } from './utils/helper.ts'
  import type { User } from './types/user.ts'

  // ✅ GOOD: 省略 `.ts`——导入的是模块说明符，不是文件路径
  import { foo } from './utils/helper'
  import type { User } from './types/user'
  ```

- 导入 `index.ts` 只引用到目录名（如 `./api/common`、`./api`），除非有目录引用约定。
- 对 常量 / 通用模型 的引用准确到文件（如 `./api/common/TheConst`、`./model/BroadcastContext`）。
- 组件引用：跨树引用准确到文件（如 `./components/views/chat/Markdown2HTML`）；邻里（同级）引用准确到目录（如 `import { NewChatButton } from '../NewChatButton'`）。
- 文件引用 ≤2 级用相对（`./types`、`../common`）；>2 级用绝对（如 `@/api/common/TheConst`）。级数 = 路径里 `../` 的层数（`./` 不计层级，只是「当前目录内」）：

  ```typescript
  // src/components/views/chat/Chat.vue

  // ✅ GOOD: 0 级（同目录 ./）——相对
  import { foo } from './ChatInput'
  import type { Msg } from './types'

  // ✅ GOOD: 1 级（一个 ../）——相对
  import { bar } from '../shared/format'      // ../shared = 1 级
  import { baz } from '../../model/User'      // ../../model = 2 级 → 仍在阈值内

  // ❌ BAD: 超过 2 级还用相对，层级深、易写错、移动文件就断
  import { qux } from '../../../api/common/TheConst'   // 3 级 → 超阈值

  // ✅ GOOD: >2 级用绝对别名（@ 指向 src）
  import { qux } from '@/api/common/TheConst'
  ```

- 目录级资源用 `index.ts` 做 barrel 逐级导出；该目录有类型定义时配套 `types.ts`。
- 同一文件的值导入 + 类型导入写成一条语句，用内联 `type` 修饰符；单条因 value / type 混杂或数量多而变难读时，才拆成「值导入」+「类型导入」两条：

  ```typescript
  // ❌ BAD: 能合成一条却拆成两条
  import { foo } from './module'
  import type { Bar } from './module'

  // ✅ GOOD: 一条搞定，内联 type 修饰符（TS 4.5+）
  import { foo, type Bar } from './module'

  // ❌ BAD: value / type 混杂、数量又多，硬塞一条，读起来跳
  import {
    connect,
    disconnect,
    type ChannelConfig,
    ChannelContext,
    type ChannelEvent,
    type ChannelStatus,
    TheConst,
  } from './features/realtime/broadcast/channel'

  // ✅ GOOD: 拆成「值导入」+「类型导入」两条，各归各
  import {
    connect,
    disconnect,
    ChannelContext,
    TheConst,
  } from './features/realtime/broadcast/channel'
  import type {
    ChannelConfig,
    ChannelEvent,
    ChannelStatus,
  } from './features/realtime/broadcast/channel'
  ```
