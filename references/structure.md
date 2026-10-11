# src 结构与组织规范

以下相对路径从目标项目的实际 src 根解析，src 可位于仓库根或某个应用子目录。先检查现有组织；新增项目采用此默认方式，既有项目按任务范围适配。

## 目录职责

| 位置 | 放什么 | 不放什么 |
| --- | --- | --- |
| 根入口与 App | React 挂载、全局样式、Provider、路由/页面组合、必要初始化 | 完整页面业务、领域请求和大型状态逻辑 |
| pages/<PageName>/index.tsx | 页面协调、简单查询/状态/事件与展示入口；复杂私有 components/hooks/utils/config/类型/测试就近 | 其他页面依赖的共享基础工具 |
| components/<ComponentName>/index.tsx | 跨页面复用组件或有明确复用需求的公共能力；内部组件与 hook 并置 | 页面领域流程，反向 import pages |
| layouts/<LayoutName>/index.tsx | 导航、应用外壳、共享布局与内容插槽 | 每个页面自己的表单、查询与提交 |
| services/<domain>.ts | 具名领域接口、契约验证、网络到领域结果的映射 | JSX、页面中文提示、传输层基础设施 |
| services/index.ts | 领域命名空间导出，便于页面使用 domainService | 将所有接口实现堆入 index |
| store/<domain>.ts | 确需共享的领域状态、actions 与选择器 | 本地输入框、所有请求结果的重复缓存 |
| utils/ | 按职责分文件的纯工具与请求等基础设施；共享 hook 分 use 文件或 hooks 目录 | 依赖页面的万能 utils.ts，互相调用的业务流程 |
| constants/<domain>.ts | 业务枚举、状态映射、选项及稳定字段配置；页面强绑定的布局/派发表就近 | 当前登录信息、运行状态、带事件和副作用的配置 |
| typings/<domain>.ts | 领域接口参数/响应、services 与页面共用的业务类型，显式 export | JSX、运行代码、全部组件 props 的集合 |
| typings/*.d.ts | 环境/第三方补充、目标项目确需的 ambient 声明 | 为省 import 而新增全局业务 namespace/any 集合 |
| styles/ | reset、字体、主题、通用 mixin、限定范围的第三方覆盖 | 本页具体布局、泄漏到全站的裸选择器 |
| assets/ | 共享图像、字体、图标；功能私有资源也可并置 | 密钥、运行数据、无需求的占位素材 |

## 页面与组件内部组织

```text
src/
├── App.tsx
├── layouts/BasicLayout/index.tsx
├── pages/Products/
│   ├── index.tsx
│   ├── index.module.css
│   ├── components/ProductEditDialog/
│   │   ├── index.tsx
│   │   └── index.module.css
│   ├── types.ts
│   └── config.ts
├── components/EmptyState/index.tsx
├── services/
│   ├── products.ts
│   └── index.ts
├── typings/products.ts
├── constants/products.ts
└── utils/XRequest.ts
```

这是职责示例，不要求一次创建全部文件。一个简单组件可仅有 index.tsx；很短的 UI 私有常量/类型可留所属文件，网络 DTO 则按后面的 typings 规则归属。有独立职责或真实复用再拆；复杂组件可以有内部 components/hooks/utils，与页面采用相同的并置规则。

- **简单页面默认不拆 hook**：一次加载、刷新、loading/error、局部筛选和少量事件可在 index.tsx 中组织。有 useEffect 或调用 service 本身不是提取理由。
- 提取 hook 需有明确收益：复用同一行为、复杂多步骤编排可独立表达/测试，或页面理解成本显著降低；指出收益及状态所有者。页面私有 hook 放本页 hooks，共享后再按仓库惯例提升。不能把每个请求机械拆文件，也不把全页状态集中成万能 hook。
- 主从页的轻量上下文/主控列表可以平铺在页面；详情、编辑面板等独立请求和交互闭环可抽私有组件。只有某区块使用的状态与请求归属该区块，不因存在 onSaved 回调就上移所有状态。

- 页面筛选、表格、编辑弹窗为功能私有时留在本页 components；相同 UI 在其他页面有实际复用需求时再提取公共组件。
- 页面只使用的树转换函数留在页面 utils 或独立 camelCase 文件；一旦共享组件需要它，重新确定公共归属，不能 import 私有页面路径。
- 动态组件/低代码场景可采用 components/atom/<Name>，将 config、类型、运行视图并置；普通页面不用复制这种组件族或配置引擎。
- 就近测试采用仓库既有 test/spec 命名与位置，支持 utils/__tests__ 等模式，但不强制迁移已有 tests 目录。

## 命名、导出与依赖

- 页面、布局、组件目录及组件名 PascalCase；普通模块和函数 camelCase，hook 以 use 开头。index 作为组件入口，config/const/types 等按实际职责命名，避免同目录多种不一致写法。
- 组件默认导出、普通工具/领域函数具名导出作为默认习惯；尊重既有公共 API。service 入口示例：`export * as productService from './products'`。
- 同模块使用相对导入；跨模块可用仓库已支持的 alias。若新增 alias，运行时/构建、TypeScript 和测试解析都要验证，不能只改编辑器 paths。
- 页面/页面 hook/状态 action → 领域 service → 统一请求 → HTTP；布局组合页面/公共组件；公共组件和纯工具不能依赖 pages。统一请求不能回调业务 store/service 形成环，应通过配置或依赖注入处理必要策略。
- 不强制给每层建 barrel。目录入口只汇总稳定公共 API，内部文件避免经自己的 index 导入造成循环依赖。
- 默认两空格、单引号、无分号；遵守仓库 formatter/lint 的真实设置，禁止顺手格式化无关文件。

## 类型、配置和资源

- 接口请求/响应、跨 service 与页面的领域结果类型默认集中在 typings/<domain>.ts，使用 export interface/type 和 import type。不要继续新增 services/<domain>.types.ts；迁移时更新调用方，避免两份定义。已有仓库明确采用另一规范时说明冲突，迁移只限本次授权。
- 小 props 留在组件旁；页面 UI-only 类型可放本页 types.ts；store 的内部状态类型就近。UI 表单类型与 DTO 可以不同，转换明确归属；集中业务类型不等于集中所有类型。
- .d.ts 用于真实 ambient 需求。已有全局声明可按项目规范保留，但不默认新增 declare module/namespace。未知 JSON 从 unknown 校验，泛型或类型断言不是验证。
- 枚举、选项、状态映射和业务字段配置默认按域放 constants；页面专属布局、面板派发表或很小的 UI 常量可并置。静态配置与事件通过参数/工厂组合，不把动态 viewport、用户信息塞进“常量”；字段细则见主文件按需引用。
- 全局图标/字体集中，页面素材并置或按资产惯例存放。避免硬编码创建者绝对路径或业务品牌。

## 样式与框架适配

- 局部样式与组件并置，默认使用仓库现有的 CSS Modules/Less 等方案；没有选择时优先隔离样式。`import styles from './index.module.css'` 或受支持的 .module.less 是明确的局部入口。
- 来源项目的 `import styles from './index.less'` 可能依赖编译器自动模块化；先核对目标构建，不能照搬后宣称可用，也不为延续文件名自动安装 Less。
- 根入口显式引入全局样式。主题/变量和 mixin 共享；第三方覆盖限定在相应区域，组件自己的布局不放到 global。
- 布局用 children 或当前路由的内容插槽，接入现有路由配置；单页无导航需求不加路由库。Provider 的放置按消费范围，不默认全部堆根部。
- 排除框架专有自动入口、生成目录、路由导入、编译常量和旧业务认证/宿主桥接。学习初始化/加载回退的职责，按目标框架显式接线。
- 保留功能就近、分域 service、局部样式与状态选择器的组织方法；调整反向依赖、全局业务类型和大型万能工具；排除原业务配置与框架魔法。
