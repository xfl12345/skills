---
name: xfl12345
description: >
  xfl12345 的个人编码推荐规范。当且仅当系统提示或者用户指定使用本规范，
  在写、改、审任何 Java / TypeScript / JavaScript / Vue 源码时触发。
when_to_use: >
  当且仅当系统提示或者用户指定使用本规范，在写、修改、审查 Java / TypeScript / JavaScript / Vue 源码时使用本 skill
license: MIT
metadata:
  author: xfl12345
  version: "1.0.0"
---

# xfl12345 个人编码推荐规范

## When to Use

当且仅当系统提示或者用户指定使用本规范，在写、修改、审查 Java / TypeScript / JavaScript / Vue 源码时使用本 skill

## 通用约定

- 旧注释有参考价值就保留，不要养成改代码就删注释的坏习惯
- 代码可读性永远放在第一位（减少 BUG、性能优先的同时，也要保证人类可读性）
- 尽量不要让高度相似的代码重复三遍，该抽象的抽象，该通用的通用
- **闭包纯净**：lambda / 箭头函数内被捕获的外部变量只读不改写——要产出走返回值（生产归生产），要修改放回显式循环或具名方法让副作用可见（修改归修改）；纯观察型副作用（如日志）不在此限；确需例外的必须注释缘由

### 语法

- **switch vs if**：支持 switch 的语言，3 条及以上常量匹配一律 switch-case；2 条 / 范围判断 / 复合条件用 if
- **花括号**：Java/JS/TS/C# 等 if 控制块用花括号的语言，必须花括号包裹，禁止写成一行
- **链式调用拆行**：一行最多 3 次函数调用（含无参的起始调用如 `Config.builder()`，以及终止调用如 `.build()`）。≤3 次可一行；一旦超过 3 次，整条链全部拆成每行一次调用
  ```typescript
  // ✅ GOOD: 刚好 3 次调用（filter / map / slice），一行
  const activeNames = list.filter(x => x.active).map(x => x.name).slice(0, 10)

  // ❌ BAD: 4 次调用挤一行，动作太多难读
  const badSortedNames = list.filter(x => x.active).map(x => x.name).slice(0, 10).sort()

  // ✅ GOOD: 超过 3 次，整条链拆成每行一次调用（第一行 = 起点接收者 + 第一个调用）
  const sortedActiveNames = list.filter(x => x.active)
    .map(x => x.name)
    .slice(0, 10)
    .sort()

  // ✅ GOOD: builder 模式——Config.builder() / .setName() / .build() 刚好 3 次，一行
  const config = Config.builder().setName('admin').build()

  // ❌ BAD: 4 次调用挤一行（Config.builder() / .setName() / .setLevel() / .build()）
  const badConfig = Config.builder().setName('admin').setLevel(1).build()

  // ✅ GOOD: 超过 3 次，整条链拆开（Config.builder() 起点调用独占一行）
  const config2 = Config.builder()
    .setName('admin')
    .setLevel(1)
    .build()
  ```

### 包容性命名

**总则**：字段 / 标识符 / 文案命名不得出现歧视性词汇；凡带种族、压迫色彩的词一律换中性词。

- ❌ `blacklist` / `whitelist` → ✅ `denylist` / `allowlist`
- ❌ `slave` / `master`（DB / 缓存 / 集群拓扑等）→ ✅ `worker` / `controller`（视语义也可 `primary` / `secondary`、`leader` / `follower`）

## 细则

按当前文件的语言进入对应 reference（路径相对本 skill 目录）：

> 写 / 改 / 审任何 Vue 组件时，按内部的 script 标签按其 lang 属性叠加触发加载对应的语言 SKILL  
> 例如 `<script lang="ts">` / `<script lang="js">` 分别对应 TypeScript 和 JavaScript  
> tsx/jsx 同理

| 语言 | 入口  | 何时读 | 常见文件后缀 |
|---|---|---|---|
| Java | [`references/java.md`](./references/java.md) | 写 / 改 / 审任何 Java 源码 | `.java` |
| JavaScript | [`references/javascript.md`](./references/javascript.md) | 写 / 改 / 审任何 JavaScript/TypeScript 源码 | `.js` / `.jsx` |
| TypeScript | [`references/typescript.md`](./references/typescript.md) | 写 / 改 / 审任何 TypeScript 源码 | `.ts` / `.tsx` |
| Vue | [`references/vue.md`](./references/vue.md) | 写 / 改 / 审任何 Vue 组件源码 | `.vue` |
