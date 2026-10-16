# Agent 写 Vue 3 后台代码规范（精简版）

## 0. 使用说明

- **适用范围**：Agent 生成、修改、审查 Vue 3 后台管理系统、业务中后台、组件库、可复用 composable 与前端基础库。
- **自动化边界**：排版、import 排序、可自动修复风格交给 ESLint、Prettier、Biome、Oxlint、Stylelint 或项目既有工具；本文只约束命名、类型、模块边界、Vue 写法、后台契约、CSS、测试、构建、安全和风险。
- **冲突优先级**：项目已有 lint / format / tsconfig / UI 框架约定 > Vue 官方 Style Guide 与 API 文档 > ESLint / typescript-eslint / StandardJS / Stylelint 官方规则 > 本文 > 当前文件局部写法。
- **设计优先级**：正确 > 类型安全 > 清晰 > 简单 > 可维护 > 一致 > 简洁。先用 Vue 官方生态、项目既有抽象和标准 Web API，再新增依赖。
- **规则强度**：`MUST` 强制；`MUST NOT` 禁止；`SHOULD` 默认遵守，有充分理由才偏离；`MAY` 可选。

Agent 执行任务时：

1. 先按分类归属表定位规则。
2. 修改已有项目先读 `package.json`、Node / 包管理器版本、lint 配置、tsconfig、Vite 配置和相邻实现。
3. 生成代码遵守所有 `MUST` / `MUST NOT`。
4. 偏离 `SHOULD` 时，在交付说明里写明原因。
5. 审查问题按“正确性与安全 > 类型与异步 > 后台契约与权限 > Vue / CSS 体验 > 测试 > 性能发布 > 命名格式”排序。
6. 完成后执行第 9 节检查；不要只凭页面能跑就交付。

### 分类归属

| 分类 | 规则前缀 | 放什么 |
|---|---|---|
| 工程基线 | `BASE-*` | 包管理器、lint、format、tsconfig、CI、生成产物、monorepo |
| 代码边界 | `CODE-*` | TS / JS、命名、目录、import、模块边界、注释文档 |
| Vue 写法 | `VUE-*` | SFC、props、emits、v-model、slots、响应式、composable、template |
| 后台契约 | `ADMIN-*` | layout、路由菜单权限、CRUD、请求层、Mock、文件、主题、图表、代码生成 |
| 样式体验 | `UI-*` | CSS、token、选择器、布局、动画、可访问性 |
| 状态副作用 | `STATE-*` | async、Router、Pinia、SSR、插件、directive、生命周期清理 |
| 质量交付 | `QA-*` | 测试、性能、构建、发布、依赖 |
| 安全风险 | `SAFE-*` | XSS、权限租户、i18n、合规、风险治理 |

### 开源项目借鉴口径

Agent 不照搬任一项目目录或审美，只吸收稳定工程经验。

| 参考对象 | 借鉴点 |
|---|---|
| Vue Vben Admin | 工程基线、monorepo、权限路由、质量门禁、构建发布 |
| Soybean Admin、vue-naive-admin、naive-ui-admin | Naive UI、UnoCSS、主题、动态菜单、Composition API |
| vue-pure-admin、Geeker Admin、v3-admin-vite、vue3-element-admin | Element Plus 表格表单、布局、标签页、二次开发约定 |
| vue3-antdv-admin、Arco Design Pro Vue | 企业后台组件 API、页面契约、表单表格模式 |
| fast-crud、cool-admin、gin-vue-admin、RuoYi-Vue3 | CRUD 配置化、代码生成、按钮权限、租户和数据权限 |

---

## 1. 工程基线（BASE）

- `BASE-01 MUST` 修改代码前识别包管理器和 Node 版本：优先按 lockfile、`packageManager`、`.nvmrc` / `.node-version` / `engines` 选择 `pnpm`、`yarn`、`npm`、`bun`，不得混用。
- `BASE-02 MUST` 使用项目已有格式化、lint、typecheck、test、build 脚本。没有统一脚本时，至少寻找等价能力：ESLint、Stylelint、`vue-tsc --noEmit`、Vitest、production build。
- `BASE-03 SHOULD` Vue 3 + TS 项目开启 `strict`、ESLint、Stylelint、typecheck、unit test；组件库、复杂 UI、SSR 或布局改动补浏览器组件测试、E2E 或视觉回归。
- `BASE-04 MUST` CI 至少执行 lint、typecheck、unit test 和 production build；任一失败阻止合并。
- `BASE-05 MUST NOT` 使用无原因的 `@ts-ignore`、整文件 `eslint-disable`、裸 `eslint-disable-next-line`。压制必须写规则名和原因，TypeScript 优先 `@ts-expect-error`。
- `BASE-06 SHOULD` 无格式配置时默认 StandardJS 风格：2 空格、单引号、无分号、`===`、无未使用变量、无隐式全局、camelCase。已有 Prettier / Biome 配置优先。
- `BASE-07 MUST` 自动生成类型、路由、组件索引、接口 SDK 有明确归属：受版本控制的由脚本更新；不受版本控制的进入 `.gitignore` 并在开发或 CI 中生成。
- `BASE-08 SHOULD` monorepo 使用 workspace 脚本和受控任务编排；跨包修改先跑受影响包快速检查，合并前跑根级检查。
- `BASE-09 MUST NOT` 因本地环境差异修改 lockfile、format 配置、tsconfig 或生成产物。只有依赖、构建边界或工具版本确实变化时才提交。

```ts
// OK: 压制点小，有规则和原因。
// @ts-expect-error -- 旧 SDK 类型缺失，运行时已由 isUserPayload 校验。
const payload = legacySdk.getPayload()

// BAD: 不知道压制什么，也不知道何时删除。
// @ts-ignore
const payload = legacySdk.getPayload()
```

---

## 2. 代码边界（CODE）

### TypeScript / JavaScript

- `CODE-01 MUST` Vue 3 新代码默认 TypeScript；公共函数、导出 composable、store、service、组件 props / emits / slots 类型必须可推导或显式导出。
- `CODE-02 MUST NOT` 用 `any` 当默认逃生口。外部数据用 `unknown`，在边界通过类型守卫、schema 校验或显式转换收窄。
- `CODE-03 MUST` 外部输入、URL 参数、localStorage、postMessage、SDK 返回、接口响应都视为不可信，进入领域层前完成校验和转换。
- `CODE-04 SHOULD` `interface` 用于可扩展对象形状和公共 props；`type` 用于联合、交叉、映射、函数签名和字面量状态。
- `CODE-05 SHOULD` UI 状态使用字符串字面量联合、判别联合或 `as const` 对象；避免多个布尔值表达同一状态机。
- `CODE-06 SHOULD` 对象配置用 `satisfies` 保留字面量类型并校验结构；不要用 `as Xxx` 强行绕过校验。
- `CODE-07 MUST` `null` / `undefined` 语义明确。未加载和缺省默认 `undefined`；业务明确空值用 `null` 并写进类型。
- `CODE-08 MUST NOT` 用非空断言 `!` 掩盖真实空值；只允许在生命周期或 DOM ref 已由控制流保证时最小化使用。
- `CODE-09 MUST` 默认 `const`，需重新赋值才用 `let`；不得使用 `var`。
- `CODE-10 MUST` 使用 `===` / `!==`；唯一可接受宽松比较是有意判断 `null` 与 `undefined` 的 `value == null`。
- `CODE-11 MUST` 每个 Promise 必须 `await`、`return`、`.catch()`、`.then(_, onRejected)` 或用 `void` 明确丢弃并说明错误处理位置。
- `CODE-12 MUST` `catch` 错误按 `unknown` 处理，进入 UI 或日志前用 `instanceof Error`、领域错误守卫或 `normalizeError` 收窄。
- `CODE-13 MUST NOT` 使用 `eval`、`new Function`、字符串形式 `setTimeout` / `setInterval`，不得扩展原生对象原型。
- `CODE-14 MUST NOT` `throw` 字符串、普通对象或数字；抛出 `Error` 或领域错误类型。

### 命名

- `CODE-15 MUST` 变量、函数、参数、props 使用 camelCase；类型、interface、class、组件名使用 PascalCase；事件名用 kebab-case 或 Vue 约定名。
- `CODE-16 MUST` 缩写按词处理：`userId`、`requestId`、`url`、`httpClient`、`jsonText`。业务字段统一写 `Id`，不用 `ID`。
- `CODE-17 MUST NOT` 使用空泛名字：`data`、`list`、`info`、`item`、`temp`、`obj`、`helper`、`common`。短循环内可用 `item`，跨语句必须换成领域名。
- `CODE-18 SHOULD` 布尔值使用 `is`、`has`、`can`、`should`、`allow`、`disabled`、`loading`、`open` 等可读前缀或状态词，避免双重否定。
- `CODE-19 MUST` handler 按动作命名：`handleSubmit`、`handleRowClick`、`closeDialog`。不要写 `doSubmit`、`clickBtn`、`onClick1`。
- `CODE-20 MUST` Vue 用户组件名多词，根组件 `App` 例外；composable 命名 `useXxx`；Pinia store 命名 `useXxxStore`。
- `CODE-21 SHOULD` SFC 文件名全仓统一 PascalCase 或 kebab-case；默认 PascalCase。路由目录和普通 TS 模块默认 kebab-case。
- `CODE-22 MUST NOT` 在 props 中用 `onXxx` 表达普通事件回调；应使用 emits。render function / JSX listener object 例外。

### 模块、目录与文档

- `CODE-23 MUST` import 分组稳定：Vue / 平台 API → 第三方依赖 → 项目别名 → 相对路径 → 类型导入。类型导入使用 `import type`。
- `CODE-24 MUST NOT` 同一模块重复 import；普通 TS 模块默认具名导出，Vue SFC 可由工具链默认导出。
- `CODE-25 SHOULD` `index.ts` 只作为明确公共边界，不做“万能 barrel”隐藏依赖方向、放大循环依赖或误引入包体积。
- `CODE-26 MUST NOT` 模块 import 时执行业务副作用。例外只允许在应用入口、polyfill、样式入口或明确命名的注册模块。
- `CODE-27 SHOULD` 中大型后台按业务域组织；同一 feature 内并置 view、components、composables、services、schemas、types、tests，跨域复用再上移共享层。
- `CODE-28 MUST` 循环依赖必须修复，通过抽取类型、反转依赖、事件、依赖注入或 feature 边界解决。
- `CODE-29 MUST` workspace 包之间只通过公开入口互相引用，不跨包 import `src` 内部文件；测试或显式 internal API 例外。
- `CODE-30 SHOULD` 自动导入的 `components.d.ts`、`auto-imports.d.ts` 等必须被 tsconfig 覆盖，并明确是否提交。公共库、测试、示例代码优先显式 import。
- `CODE-31 MUST` 注释解释原因、不变量、兼容性和风险，不复述代码。普通业务注释用中文；标准 API、协议、selector、token 保持原文。
- `CODE-32 MUST` 修改公共 API、默认值、事件 payload、样式 token、slot props、可访问性行为时，同步更新文档、demo、测试和 changelog。
- `CODE-33 MUST` `TODO(risk)` 之外的 TODO 写明 owner、条件和计划；没有后续价值的 TODO 删除或落成 issue。
- `CODE-34 MUST` 方案在实现中途变化后，按当前方案回顾整条前端链路，再改代码。不得只改最后一句提到的组件，留下旧方案的类型、路由、接口字段、枚举、文案和测试。
  - 重走页面入口、路由、请求参数、响应类型、store、缓存、错误码、文案和调用方。
  - 删掉旧方案遗留：无引用类型、无路径返回的错误码、已不再展示的字段、被接口下发取代的写死文案和地址、描述旧流程的注释。
  - 方法名、组件名和目录跟同模块已经对齐的实现保持一致。

---

## 3. Vue 写法（VUE）

### SFC 与组件 API

- `VUE-01 SHOULD` SFC 默认使用 `<script setup lang="ts">`；只有 Options API 兼容、插件注册或历史约束明确时使用普通 `<script>`。
- `VUE-02 SHOULD` SFC block 顺序跟随项目；无约定时使用 `<script setup lang="ts">` → `<template>` → `<style scoped>`。
- `VUE-03 MUST` `<script setup>` 顶层代码按组件实例执行，不创建跨实例共享可变单例；跨实例常量放模块顶层且保持不可变。
- `VUE-04 MUST` `defineProps`、`defineEmits`、`defineModel`、`defineSlots`、`defineExpose`、`defineOptions` 是编译器宏，不导入。
- `VUE-05 SHOULD` props / emits 优先用泛型声明；需要运行时 validator、默认值、文档生成时可用运行时声明。
- `VUE-06 MUST` props 只读。不得修改 props，也不得修改对象 / 数组 props 的嵌套字段；紧耦合例外必须写明。
- `VUE-07 MUST` 数组和对象默认值使用工厂函数。Vue 3.5+ reactive props destructure 默认值需确认项目最低 Vue 版本。
- `VUE-08 MUST` emits 显式声明并类型化；同一事件只表达一种语义。`update:xxx` 只服务双向绑定。
- `VUE-09 SHOULD` Vue 3.4+ 使用 `defineModel()`；兼容旧版用 `modelValue` + `update:modelValue`。同一状态只允许一个写入来源。
- `VUE-10 SHOULD` slots 是结构扩展点，slot props 命名清楚且类型化，不传巨型 reactive 对象。
- `VUE-11 MUST` 多根节点组件明确处理 fallthrough attrs；输入、按钮、弹层包装组件应把 `$attrs` 绑定到真实交互元素或声明 `inheritAttrs: false`。
- `VUE-12 SHOULD` `defineExpose` 只暴露命令式能力，如 `focus`、`scrollTo`、`validate`，不暴露内部 ref 让父组件任意修改。
- `VUE-13 MUST` 公共组件的 disabled、loading、readonly、error 状态同时影响交互、样式和可访问性属性。

### 响应式与 composable

- `VUE-14 SHOULD` 默认用 `ref` 声明响应式状态；多个字段同生同灭且不直接解构时可用 `reactive`。
- `VUE-15 MUST` 解构 `reactive` 或 props 不得丢失响应式。对外暴露字段用 `toRef` / `toRefs`，传入 composable 用 getter 或 `toValue`。
- `VUE-16 SHOULD` 大型不可变结构、第三方实例、DOM API、图表、地图使用 `shallowRef` / `markRaw`。
- `VUE-17 MUST` `computed` 只做同步、纯派生计算，不请求接口、不写状态、不 emit、不依赖时间随机数。
- `VUE-18 MUST` watcher 源必须是 ref、reactive、getter 或数组；异步 watcher 必须处理过期结果和清理。
- `VUE-19 MUST` 异步创建的 watcher、interval、监听、observer 不会自动随组件卸载清理时，必须保存 stop / dispose 并在 `onUnmounted` / `onScopeDispose` 清理。
- `VUE-20 MUST` composable 命名 `useXxx`，只在被调用时产生副作用；模块 import 不启动监听、请求或读取用户环境。
- `VUE-21 SHOULD` composable 参数优先 options object，支持 `MaybeRefOrGetter` 时内部用 `toValue`。
- `VUE-22 MUST` composable 使用 `window`、`document`、`navigator`、`localStorage` 时处理 SSR、测试和 iframe；公共 composable 允许注入 global。
- `VUE-23 SHOULD` 异步 composable 返回 `data`、`error`、`isLoading`、`execute`、`abort` 等明确状态和命令，不只返回 Promise。
- `VUE-24 SHOULD` composable 返回给外部的状态默认 `readonly`，修改入口通过 action 暴露。

### Template

- `VUE-25 MUST` `v-for` 使用稳定唯一 `key`；列表可能新增、删除、排序、筛选时不得用 index。
- `VUE-26 MUST NOT` 同一元素同时使用 `v-if` 和 `v-for`。先 computed 过滤，或把 `v-if` 放到外层 `<template>`。
- `VUE-27 MUST` template 表达式保持短小；复杂计算、异步调用和副作用移到 computed、handler 或 composable。
- `VUE-28 MUST` 用户输入内容默认用文本插值；使用 `v-html` 前必须确认来源可信并已消毒。
- `VUE-29 MUST` 表单控件绑定正确类型：数字显式转换，trim 按业务决定，不让后端承担前端类型混乱。
- `VUE-30 SHOULD` 条件渲染不破坏焦点、输入和动画状态；需要保留状态时用 `v-show`、缓存或显式恢复策略。

```vue
<!-- OK: props 只读，更新通过 emit。 -->
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

function handleInput(event: Event) {
  emit('update:modelValue', (event.target as HTMLInputElement).value)
}
</script>

<template>
  <input :value="props.modelValue" @input="handleInput">
</template>
```

---

## 4. 后台契约（ADMIN）

### 应用骨架与目录

- `ADMIN-01 MUST` 修改后台项目前先识别 layout、router、menu、tabs、breadcrumb、keep-alive、permission、request、store、theme 的既有约定，不绕过框架能力另起一套。
- `ADMIN-02 SHOULD` 业务域目录并置页面、组件、composable、service、schema、类型和测试：

  ```text
  features/order/
    views/
      OrderList.vue
      OrderDetail.vue
    components/
    composables/
    services/
    schemas/
    types.ts
  ```

- `ADMIN-03 MUST` 后台 layout 的 sidebar、topbar、tabs、breadcrumb、content、footer、watermark、global search 有明确层级和滚动容器；表格页必须定义高度策略，避免 body 和表格双滚动。
- `ADMIN-04 MUST` 主题、暗色模式、紧凑密度、语言、标签页、菜单折叠、列配置等用户偏好集中持久化，并按用户 / 租户隔离；持久化结构有版本号和迁移兜底。

### 路由、菜单与权限

- `ADMIN-05 MUST` 路由 meta 类型化并集中维护。常用字段：`title`、`i18nKey`、`icon`、`layout`、`requiresAuth`、`permissions`、`roles`、`activeMenu`、`keepAlive`、`hideInMenu`、`order`、`breadcrumb`、`affixTab`。
- `ADMIN-06 MUST` 后端动态路由 / 动态菜单经过单一归一化管道：接口 schema → 类型校验 → route record / menu tree → 权限过滤。组件不得散落解析原始菜单结构。
- `ADMIN-07 MUST` 登录页、回调页、403、404、500 等白名单路由集中声明；守卫处理 token、用户信息、权限加载、重定向和异常，防止无限重定向。
- `ADMIN-08 MUST` 前端按钮权限、菜单权限、路由权限只做展示和交互控制；接口鉴权、租户隔离、行级 / 数据权限必须由后端执行。
- `ADMIN-09 SHOULD` tab、keep-alive、breadcrumb、active menu 使用稳定 route name / fullPath 策略。退出、租户切换、权限变化时清理不可访问缓存页和标签页。
- `ADMIN-10 MUST` 页面离开、关闭 tab、切换详情对象前，未保存表单、草稿、批量编辑、审批操作必须有 dirty guard 或明确丢弃策略。

### CRUD 页面

- `ADMIN-11 SHOULD` 列表页默认包含查询表单、工具栏、表格、分页、批量操作、列配置、刷新、导入 / 导出入口；缺少项必须符合业务场景。
- `ADMIN-12 MUST` 查询、分页、排序、筛选、列配置和 URL query 的同步策略有单一来源。点击查询重置到第一页；分页和排序保留当前查询；重置回到初始查询值。
- `ADMIN-13 MUST` 表格使用稳定 `rowKey` / `key`。数据刷新、筛选或权限变化后，失效的 selection、展开行、编辑行、批量状态必须清理或重算。
- `ADMIN-14 MUST` 列表请求实现并发策略，搜索和分页默认“取消上一次 / 取最后一次”。响应归一化为 `{ list, total, page, pageSize }` 或项目等价结构。
- `ADMIN-15 MUST` 列表、空状态、错误状态、无权限状态、加载状态、部分失败状态都有 UI 表达。
- `ADMIN-16 SHOULD` 批量删除、批量审核、导入、导出、状态迁移有权限判断、二次确认、loading、结果反馈和部分失败展示。
- `ADMIN-17 MUST` 新增、编辑、详情、审核、复制等页面模式显式建模，例如 `mode: 'create' | 'edit' | 'detail' | 'audit' | 'copy'`。模式决定标题、字段禁用、按钮、接口和校验。
- `ADMIN-18 MUST` 表单初始值使用工厂函数；接口 DTO、表单状态、提交 payload 分层转换。不得把接口响应对象直接作为长期可变表单状态。
- `ADMIN-19 MUST` 表单提交处理校验、后端字段错误映射、重复提交、幂等、成功跳转 / 留页策略和失败恢复。弹窗 / 抽屉保存成功后通过事件或 action 通知父级刷新。
- `ADMIN-20 SHOULD` 配置化 CRUD、动态表单、动态表格使用类型化 schema。schema 声明字段、权限、校验、接口、列、搜索项和操作；复杂业务逻辑不要藏在匿名 render 里。

### 请求、Mock 与文件

- `ADMIN-21 MUST` 请求层集中封装 baseURL、凭证、401 / 403、token refresh、超时、取消、traceId / requestId、错误归一化和响应解包。组件不得依赖 axios / fetch 原始响应结构。
- `ADMIN-22 MUST` token refresh 使用 single-flight 或等价机制，避免多个并发 401 同时刷新；刷新失败必须清理登录态并跳转可控登录路径。
- `ADMIN-23 SHOULD` service 按领域命名：`fetchOrderList`、`createOrder`、`updateOrderStatus`。不要把 HTTP 动词和 URL 泄漏到组件 handler。
- `ADMIN-24 MUST` 上传、下载、预览、导出校验文件大小、MIME、扩展名、权限和取消策略。Blob URL 使用后 revoke，下载文件名需从可信来源解析并消毒。
- `ADMIN-25 MUST` Mock 只能用于 dev / test / demo 边界，不得被 production bundle 引用。Mock schema 与真实接口类型共用或自动校验。
- `ADMIN-26 SHOULD` fixture 放在靠近 service 或测试的 fixtures 中，不把大量假数据写进 SFC。

### 图表、主题与代码生成

- `ADMIN-27 SHOULD` 主题接入优先使用组件库 token、CSS 自定义属性、UnoCSS / Tailwind preset 或项目设计 token；暗色模式调整语义 token，不在页面堆一次性颜色覆盖。
- `ADMIN-28 SHOULD` dashboard、图表、地图、编辑器按路由或组件懒加载。ECharts / 地图 / 编辑器实例用 `shallowRef` / `markRaw`，存在于组件或 composable，并在卸载时 dispose。
- `ADMIN-29 MUST` 图表处理 loading、empty、error、resize、主题切换和数据单位。不得把图表实例、DOM 节点或 resize observer 放入 Pinia 领域 store。
- `ADMIN-30 MUST` 代码生成产物标明来源、生成命令和可编辑边界。可重复生成文件不得手工改；定制通过 schema、hooks、slots、局部组件或 extension point 扩展。
- `ADMIN-31 SHOULD` 生成器、配置化页面和插件系统有 schema version、迁移策略和 fixture 测试。新增字段兼容旧配置或给迁移。
- `ADMIN-32 MUST` 框架级扩展，如权限指令、全局 modal、全局 table wrapper、request plugin、layout plugin，必须有类型、测试和文档；业务页面不得复制相似实现。

```ts
// OK: route meta 字段集中，菜单和权限复用同一契约。
const orderRoutes = [
  {
    path: '/orders',
    name: 'OrderList',
    component: () => import('./views/OrderList.vue'),
    meta: {
      title: '订单管理',
      i18nKey: 'route.orders',
      icon: 'shopping-cart',
      requiresAuth: true,
      permissions: ['order:list'],
      keepAlive: true,
      order: 30,
    },
  },
] satisfies AdminRouteRecord[]
```

---

## 5. 样式体验（UI）

### CSS 与 token

- `UI-01 MUST` 样式改动遵守项目既有方案：原生 CSS、PostCSS、Sass / Less、CSS Modules、Tailwind、UnoCSS、CSS-in-JS 或组件库主题系统。不得临时混入另一套范式。
- `UI-02 SHOULD` 组件样式默认局部化：`<style scoped>`、CSS Modules、原子类约定或项目 CSS-in-JS；全局样式只放 reset、base、tokens、theme、typography、layout 入口。
- `UI-03 MUST` 公共组件使用设计 token、CSS 自定义属性或主题系统，不硬编码不可维护的颜色、字体、间距、圆角、阴影、层级和 z-index。
- `UI-04 MUST` token 命名表达语义，不表达当前颜色：用 `--color-danger-bg`、`--space-control-x`，不用 `--red-500-for-button`。
- `UI-05 SHOULD` CSS 自定义属性按作用域定义：全局 token 在 `:root` 或主题容器；组件私有变量在组件根 class；公共变量必须有默认值或主题契约。
- `UI-06 MUST` class 命名使用项目约定，默认 kebab-case + 结构 / 状态语义：`user-card`、`user-card__title`、`is-active`、`has-error`。
- `UI-07 MUST NOT` 使用 id 选择器承载组件样式；id 只用于锚点、label 关联或第三方协议。
- `UI-08 SHOULD` 选择器保持低特异性，嵌套深度默认不超过 2 层。不得把深度选择器、全局选择器、`!important` 当常规方案。
- `UI-09 MUST` 公共组件可覆盖部位必须作为明确 CSS API、slot API、CSS 变量或 `data-*` 状态，不让调用方依赖脆弱 DOM 层级。

### 布局、动画与可访问性

- `UI-10 MUST` 响应式布局优先使用 flex、grid、container query、media query、`minmax()`、`clamp()`、`aspect-ratio`、逻辑属性；不要用 JS 反复测量完成普通布局。
- `UI-11 MUST` 固定格式元素定义稳定尺寸或约束：按钮、图标按钮、表格列、头像、缩略图、canvas、图表容器必须有 `width` / `height` / `aspect-ratio` / `minmax()` 等边界。
- `UI-12 MUST` 可滚动区域明确 `min-height: 0` / `min-width: 0`、`overflow` 和滚动容器边界。移动端全高布局不要只用 `100vh`，优先考虑 `100dvh` / `100svh` 与 safe-area。
- `UI-13 MUST` 文本容器处理溢出：长英文、URL、文件名、用户输入使用 `overflow-wrap`、`word-break`、`text-overflow` 或明确换行策略。
- `UI-14 MUST NOT` 用绝对定位、负 margin、魔法数字或 transform 平移修复正常文档流能解决的布局。
- `UI-15 SHOULD` z-index 使用命名层级 token：base、sticky、dropdown、modal、toast、tooltip，不随手写 `9999`。
- `UI-16 MUST` 动画不影响关键状态表达；默认只动画 `opacity`、`transform`、颜色等低成本属性，并尊重 `prefers-reduced-motion`。
- `UI-17 MUST` 可点击元素优先语义元素：按钮用 `<button type="button">`，链接用 `<a href>`，不用 `<div @click>` 模拟。
- `UI-18 MUST` 表单控件、图标按钮、错误状态有可访问名称或文本说明；错误不能只靠颜色表达。
- `UI-19 SHOULD` 弹层、下拉、菜单、对话框处理焦点进入、焦点返回、Escape、键盘导航和滚动锁定。
- `UI-20 MUST NOT` 给原生语义元素添加冲突 role；不移除可见 focus ring，自定义焦点样式必须有足够对比度。

```css
/* OK: token 语义化，状态绑定真实属性。 */
.user-card {
  --user-card-bg: var(--color-surface);
  --user-card-border: var(--color-border-subtle);

  background: var(--user-card-bg);
  border: 1px solid var(--user-card-border);
}

.user-card[aria-disabled="true"] {
  cursor: not-allowed;
  opacity: 0.56;
}
```

---

## 6. 状态副作用（STATE）

### 异步与副作用

- `STATE-01 MUST` 请求、定时器、事件监听、observer、worker、websocket 都有明确生命周期和清理策略。
- `STATE-02 MUST` 用户可重复触发的异步动作定义并发策略：忽略重复、取消上一次、队列化、取最后一次或允许并发。
- `STATE-03 MUST` `loading`、`disabled`、`error`、`empty` 状态与异步流程一致，`finally` 恢复状态；允许并发时用计数或请求 id。
- `STATE-04 MUST` 同一错误只处理一次：要么在 service 边界转换，要么在 UI 边界 toast / modal / fallback，不重复 toast 和重复日志。
- `STATE-05 MUST` `catch` 不能静默吞错。降级成功写清原因；真正忽略必须说明不会影响用户和可观测性。
- `STATE-06 SHOULD` 请求支持 `AbortSignal`；路由切换、搜索输入变化、组件卸载时取消过期请求。
- `STATE-07 MUST` 表单提交、支付、创建订单、审批、状态迁移等关键动作处理幂等或重复触发。
- `STATE-08 SHOULD` 重试只用于幂等或明确可重入操作，并设置退避、最大次数和取消条件。
- `STATE-09 MUST` 乐观更新有回滚策略；回滚失败、局部成功和服务端修正值必须能被 UI 表达。
- `STATE-10 MUST NOT` 提交 `console.log` 调试代码；生产诊断使用项目 logger、埋点或错误上报。

### Router、Pinia、SSR 与插件

- `STATE-11 MUST` Composition API 中使用 `useRouter()` / `useRoute()`；不要 watch 整个 `route` 对象，只监听需要的 params、query 或 meta 字段。
- `STATE-12 SHOULD` 业务跳转优先使用命名路由和 typed route helpers，避免散落硬编码路径字符串。
- `STATE-13 MUST` 路由守卫返回语义明确：允许、重定向、取消或抛错；异步守卫处理错误和竞态。
- `STATE-14 SHOULD` Pinia 管理跨页面、跨组件、需持久化或需 devtools 追踪的状态；一次性表单、弹窗开关、临时输入留在组件或 composable。
- `STATE-15 MUST` store state 是函数或 setup store 的 ref / reactive；getter 保持纯计算；action 可异步但必须声明错误、loading 和并发策略。
- `STATE-16 MUST` 解构 store state / getters 使用 `storeToRefs`；actions 可直接解构。
- `STATE-17 MUST NOT` store 保存组件实例、DOM 节点、Router 实例、AbortController、Promise、图表实例等不可序列化运行时对象，专门运行时 store 例外但要清理。
- `STATE-18 MUST` 权限、租户、用户身份等全局上下文有单一来源，组件不得从多个位置拼凑权限判断。
- `STATE-19 MUST` SSR 每个请求创建独立 app、router、Pinia、API client 和请求上下文；不得用模块级可变单例承载用户、租户、权限、语言或请求数据。
- `STATE-20 MUST` SSR 渲染路径不得在模块顶层读取 `window`、`document`、`navigator`、`localStorage`、timezone、随机数或当前时间。
- `STATE-21 MUST` 避免 hydration mismatch：服务端和客户端首屏输出确定；用户本地设置、媒体查询、随机 id 延迟到挂载后或提供一致初值。
- `STATE-22 MUST` SSR 状态序列化转义并过滤敏感信息，不把 token、cookie、内部错误堆栈、私有配置写入 HTML。
- `STATE-23 SHOULD` Vue plugin 的 `install(app, options)` 可重复调用或明确防重；directive 只封装真实 DOM 行为，并在 unmounted 清理监听和 observer。

---

## 7. 质量交付（QA）

### 测试

- `QA-01 MUST` 新增重要行为必须有测试。纯函数测输入输出；composable 测状态、副作用和清理；组件测 props、events、slots、DOM 与用户交互。
- `QA-02 SHOULD` Vite / Vue 3 默认 Vitest + Vue Test Utils；依赖真实浏览器行为、布局测量、焦点、拖拽、弹层定位时使用 Cypress Component Testing 或 Playwright。
- `QA-03 MUST` 组件测试关注公共接口，不测试内部 ref、私有函数名或大快照作为唯一断言。
- `QA-04 MUST` 断言异步更新时等待 Vue 更新和 Promise 队列：`await nextTick()`、`flushPromises()` 或等价 API。
- `QA-05 SHOULD` 公共组件覆盖默认渲染、关键 props、事件 payload、slot、disabled / loading / error、可访问名称。
- `QA-06 MUST` composable 若注册副作用，测试卸载后清理：事件监听、定时器、watcher、请求取消。
- `QA-07 MUST` store action 测成功、失败、重复触发、权限 / 租户边界和持久化边界。
- `QA-08 SHOULD` 网络请求测试 mock service 边界或使用 MSW / 项目等价方案，不在组件测试散落底层 fetch mock。
- `QA-09 SHOULD` 样式、主题、暗色模式、响应式布局和关键组件视觉状态用视觉回归或截图 smoke test 覆盖。
- `QA-10 MUST` UI 改动至少运行相关测试；无法运行时在交付中说明原因和风险。

### 性能、构建与发布

- `QA-11 MUST` 先用 profile、Vue Devtools、Performance、bundle analyzer 或真实指标定位瓶颈，再做复杂优化。
- `QA-12 SHOULD` 大列表使用分页或虚拟滚动；重型路由页、编辑器、图表、地图按需加载。
- `QA-13 SHOULD` 大型不可变数据使用 `shallowRef` / `shallowReactive` 并替换根引用。
- `QA-14 MUST NOT` 在 template、computed、render 热路径创建昂贵对象、正则、Intl formatter 或复杂排序。
- `QA-15 MUST` 第三方依赖引入前评估体积、tree-shaking、SSR 兼容、维护状态、license 和安全风险。
- `QA-16 MUST` 性能优化不得移除输入校验、错误处理、权限判断、可访问性或测试覆盖。
- `QA-17 MUST` 发布包 `package.json` 明确 `name`、`version`、`type`、`exports`、`types`、`files`、`sideEffects`、`peerDependencies`。
- `QA-18 MUST` Vue 组件库把 `vue` 放入 `peerDependencies` 并 externalize；样式副作用显式声明，支持按需加载和 tree-shaking。
- `QA-19 SHOULD` 发布前用 example / playground / fixture app 安装本地产物，验证消费路径、类型补全、样式加载、SSR 和 tree-shaking。
- `QA-20 MUST` `VITE_*`、`import.meta.env` 和 public 静态资源都视为客户端公开信息，不放私密配置。
- `QA-21 MUST` 破坏性变更、废弃 API、新增公共 API、默认行为变化同步 changelog、迁移说明和 semver / changeset。

---

## 8. 安全风险（SAFE）

### 安全与国际化

- `SAFE-01 MUST` 不渲染未经消毒的用户 HTML；`v-html` 只允许可信内容或已消毒内容。
- `SAFE-02 MUST NOT` 把用户字符串作为 Vue template、render function、`innerHTML`、事件属性、脚本、style 字符串、CSS 变量名、`url()`、`@import` 或 class 名执行 / 注入。
- `SAFE-03 MUST` 用户可控 URL 使用 `URL`、白名单和同源策略处理；新窗口链接使用 `rel="noopener noreferrer"`。
- `SAFE-04 MUST NOT` 把密钥、私有 token、内部地址写入前端代码、public 资源或 `VITE_*`。
- `SAFE-05 MUST` 鉴权、租户隔离、菜单权限、按钮权限、接口权限分开治理；前端隐藏按钮不是权限控制。
- `SAFE-06 SHOULD` 敏感数据展示、日志、埋点、错误上报做脱敏，不记录 token、cookie、身份证、手机号和完整接口响应。
- `SAFE-07 MUST` iframe、postMessage、下载、上传、剪贴板、文件预览校验 origin、MIME、大小、扩展名和用户授权。
- `SAFE-08 MUST` `URL.createObjectURL`、文件预览、下载链接和临时 Blob 使用后及时 revoke。
- `SAFE-09 SHOULD` CSP、Trusted Types、SRI 等安全头由服务端统一配置；前端新增内联脚本、第三方资源或 HTML 注入点必须评估影响。
- `SAFE-10 SHOULD` 用户可见文案通过 i18n、字典或项目文案系统管理；不要在组件逻辑散落文案。
- `SAFE-11 MUST` 日期、时间、数字、金额、百分比使用 `Intl` 或项目格式化工具，并明确 locale、timeZone、currency 和 fallback。
- `SAFE-12 SHOULD` 有 RTL 或多语言需求时使用逻辑属性，不硬编码 left / right 语义。

### 风险治理

- `SAFE-13 MUST` 命中下表风险时采用对应治理手段，落到代码、测试、配置或流程。
- `SAFE-14 MUST` 无法当场治理时，紧邻代码写中文 `TODO(risk)`，写明条件、影响、兜底和计划。
- `SAFE-15 MUST NOT` 为了更短或更快移除输入校验、鉴权 / 租户隔离、注入防护、幂等、有效错误处理、可观测性、可访问性。

| 类型 | 典型场景 | 治理 |
|---|---|---|
| 异步竞态 | 搜索、路由切换请求、重复点击 | AbortController、请求序号、禁用按钮、取最后一次 |
| 幂等重试 | 创建订单、提交审批、保存表单 | 幂等键、前端禁重、后端唯一约束 |
| 状态一致 | 多弹窗、多 store、URL 与本地状态 | 单一来源、状态机、提交前校验、回滚 |
| 输入边界 | 表单、URL 参数、上传、外部 SDK | schema 校验、类型收窄、大小 / MIME 限制 |
| 权限租户 | 菜单、按钮、路由、接口参数 | 后端鉴权、前端守卫、租户参数不可由用户任意指定 |
| 容量性能 | 大表格、无限滚动、大 JSON | 分页、虚拟滚动、增量加载、内存上界 |
| SSR 一致 | 跨请求状态、hydration mismatch | 每请求实例、确定首屏、client-only 边界 |
| 发布兼容 | 默认值、导出路径、类型破坏 | semver、changeset、deprecation、fixture app |
| 可观测 | 故障难复现、用户只看到失败 | 结构化错误、traceId、埋点、错误上报 |
| 可访问性 | 弹层、键盘、表单错误 | focus 管理、aria、语义元素、屏幕阅读器测试 |

```ts
// OK: 风险闭环清楚。
// TODO(risk): 保存接口缺少服务端幂等键。条件：用户网络抖动后重复点击。影响：重复草稿。兜底：前端禁用提交按钮。计划：下个接口版本增加 requestId。
```

---

## 9. 完成检查（CHECK）

- [ ] 已读取项目 `package.json`、Node / 包管理器版本、lint 配置、tsconfig、Vite 配置和相邻实现。
- [ ] 使用项目包管理器运行相关 ESLint、Stylelint、typecheck、测试和 build；无法运行时说明原因。
- [ ] 无无原因 lint / type 压制；无裸 `any`、`@ts-ignore`、整文件 `eslint-disable`。
- [ ] 外部输入已收窄；导出 API、props、emits、slots、store、service 类型清晰。
- [ ] 命名、import、目录、模块边界、生成产物归属符合 `CODE-*`。
- [ ] SFC、props、v-model、slots、attrs、响应式、watcher、composable、template 符合 `VUE-*`。
- [ ] 后台 route meta、动态菜单、白名单路由、按钮权限、tab / keep-alive、租户权限边界统一治理。
- [ ] CRUD 查询、分页、selection、批量操作、表单模式、提交幂等、刷新策略符合 `ADMIN-*`。
- [ ] 请求层、token refresh、文件上传下载、Mock、用户偏好、图表实例、代码生成没有绕过框架约定。
- [ ] CSS token、选择器、布局、滚动、文本溢出、动画、可访问性符合 `UI-*`。
- [ ] Promise、请求、定时器、监听、observer、worker、websocket 有生命周期和竞态治理。
- [ ] Router / Pinia / SSR / plugin / directive 无跨请求污染、响应式丢失和不可清理副作用。
- [ ] 测试覆盖新增重要行为；复杂 UI 有浏览器级或视觉 smoke test。
- [ ] 性能、依赖、发布包 `exports` / `types` / `sideEffects` / `peerDependencies` / changelog 已治理。
- [ ] 无 XSS、开放跳转、用户可控 CSS / JS、敏感信息泄露、前端伪权限。
- [ ] 命中竞态、幂等、状态一致、外部依赖等风险时已按 `SAFE-*` 治理；无法治理的有 `TODO(risk)`。
- [ ] 方案中途变化时已按 `CODE-34` 回顾整条前端链路，并清掉旧方案遗留的类型、路由、字段、错误码和写死文案。

---

## 10. 参考依据

### 官方规范与基础设施

- https://standardjs.com/rules-zhcn
- https://eslint.org/docs/latest/use/configure/configuration-files
- https://typescript-eslint.io/users/configs/
- https://www.typescriptlang.org/docs/handbook/2/basic-types.html
- https://stylelint.io/user-guide/get-started
- https://github.com/stylelint/stylelint-config-standard
- https://developer.mozilla.org/en-US/docs/Web/CSS
- https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascading_variables/Using_CSS_custom_properties
- https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_cascade/Specificity
- https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_containment/Container_queries
- https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl
- https://www.w3.org/WAI/ARIA/apg/
- https://vite.dev/guide/build
- https://vite.dev/guide/ssr

### Vue 生态

- https://vuejs.org/style-guide/
- https://vuejs.org/api/sfc-script-setup.html
- https://vuejs.org/guide/components/props.html
- https://vuejs.org/guide/components/v-model.html
- https://vuejs.org/guide/essentials/watchers.html
- https://vuejs.org/guide/scaling-up/ssr.html
- https://vuejs.org/guide/scaling-up/testing
- https://vuejs.org/guide/best-practices/performance
- https://vuejs.org/guide/best-practices/security
- https://router.vuejs.org/guide/advanced/composition-api
- https://pinia.vuejs.org/core-concepts/getters.html
- https://test-utils.vuejs.org/guide/
- https://vitest.dev/guide/
- https://vueuse.org/guidelines
- https://github.com/unplugin/unplugin-vue-components
- https://github.com/changesets/changesets

### 组件库与后台项目样本

- https://github.com/element-plus/element-plus
- https://github.com/tusen-ai/naive-ui
- https://github.com/vueComponent/ant-design-vue
- https://github.com/arco-design/arco-design-pro-vue
- https://github.com/vbenjs/vue-vben-admin
- https://github.com/pure-admin/vue-pure-admin
- https://github.com/soybeanjs/soybean-admin
- https://github.com/jekip/naive-ui-admin
- https://github.com/zclzone/vue-naive-admin
- https://github.com/HalseySpicy/Geeker-Admin
- https://github.com/un-pany/v3-admin-vite
- https://github.com/youlaitech/vue3-element-admin
- https://github.com/buqiyuan/vue3-antdv-admin
- https://github.com/fast-crud/fast-crud
- https://github.com/cool-team-official/cool-admin-vue
- https://github.com/flipped-aurora/gin-vue-admin
- https://github.com/yangzongzhuan/RuoYi-Vue3
