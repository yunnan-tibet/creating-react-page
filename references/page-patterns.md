# 页面模式、状态与交互

## 目录

- [拆分原则](#拆分原则)
- [选择一个主模式](#选择一个主模式)
- [状态与生命周期](#状态与生命周期)
- [简单页面完整示例](#简单页面完整示例)

## 拆分原则

简单状态页/一次查询的 state、请求协调、事件和 effect 默认留在 index.tsx；不要为有 useEffect 就拆 hooks/useDomain.ts。复杂编排或行为复用才提取 hook，独立详情/编辑面板优先持有自己的状态与请求，避免全页万能 hook。轻量上下文控件和主控列表可平铺，不为形式建 ContextBar/List 包装层。

文件组织可按类型/常量、状态/refs、派生数据、事件、effect、渲染；优先理解成本与正确依赖，不机械规定行数。props 用明确数据/状态与具名事件，通信传必要标识，避免全对象透传、any 和子组件修改父状态。组件默认不加 memo；有成本/稳定引用需求再使用 useMemo/useCallback。

## 选择一个主模式

| 模式 | 组织与关键行为 |
| --- | --- |
| 简单状态/只读页 | 页面持查询、loading/error、刷新与取消；共享展示可拆组件，不默认 hook、表单、store |
| 查询表格页 | 草稿与已提交条件分开；条件提交重置分页，重复查询可执行；查询/表格是本页私有功能，无真实分页契约不杜撰 |
| 主从编辑页 | 容器持上下文/焦点 ID 与主控请求；独立编辑面板持详情/表单/保存，成功回调通知容器刷新或缓存失效 |
| 详情页 | 明确 ID 来源和进入/切换/无效 ID；区分不存在、无权限和故障；禁止新 ID 显示旧详情 |
| 选择表单页 | 选项来源/联动依赖明确，切换上游时重验下游值；校验 → DTO 转换 → 保存 → 成功反馈/刷新；失败保留输入 |
| 弹窗区块 | 就近定义新建/编辑模式、初始值、保存/关闭/取消；按需要重置表单，焦点与键盘沿用现有组件能力 |
| 复合工作台 | 上下文 → 主选择 → 次级区块；类型不同确需不同编辑形态时使用语义化面板与明确分派协议，按复杂路径分步实施 |

主从/工作台只存焦点 ID，对象从当前数据推导，不用双份 state 维护。数据更新后检查焦点有效性；保留、清空或选首条须有产品规则，不能默认强制第一条。编辑区上报 onSaved(id) 等实际所需标识；保存完成也要确认仍属于当前上下文，避免旧任务回调影响新选择。

同父层已有主请求时直接使用刷新/缓存失效入口，不为通信造 refreshKey。区块独立闭环时，用明确事件/现有查询机制协调，不把它所有内部状态挪回父层。readonly 模式不附赠编辑/写 API。

## 状态与生命周期

- 明确首次加载/刷新时的旧数据展示策略，按需要区分反馈；空结果和失败必须分开，主控失败与次级失败归属各自区块。失败不能装成“无数据”，取消通常不报为新故障，保存按钮防重复提交。
- 表单只在进入/切换编辑对象时按策略初始化，不能每次 render/请求返回都重置抢走输入。关闭或切换可能丢弃未保存内容时先明确保护策略；保存失败保留输入，成功后的关闭/重置/刷新顺序明确。
- effect 与外部系统同步，不反复复制派生数据；依赖完整且稳定。不默认把不稳定的整个请求实例塞进依赖，也不禁用 lint；先核对库提供的稳定 API/依赖模式，无库则用 React 的正确闭包与清理。
- 参数/上下文变更、刷新替换和卸载清理所属请求；保留请求身份，忽略迟到响应。计时器/事件/observer/rAF 成对清理，开发重复挂载安全。具体取消规则见主文件 data-access 引用。
- 稳定 key、label、button type、可读反馈、焦点/键盘和窄屏；沿用现有 UI/样式方案，不固定组件库、最小桌面宽度或拖拽依赖。

## 简单页面完整示例

下面是**测试/讲解用的注入 loader**，无 HTTP 路径，不代表已确认的业务 DTO。真实页面通过领域 service 取得数据，业务接口类型放 typings。调用方提供模块级稳定 loader；测试可以注入 fixture，正式业务 mock/demo 仍需用户授权。仅需要 React/TypeScript 和已支持的 CSS Modules。

### index.tsx

```tsx
import { useCallback, useEffect, useId, useRef, useState } from 'react'
import styles from './index.module.css'

// 仅为这个中性展示示例的 UI 类型，不是网络 DTO。
interface Item { id: string; name: string }
type LoadItems = (signal: AbortSignal) => Promise<Item[]>
type Result =
  | { state: 'loading' }
  | { state: 'ready'; items: Item[] }
  | { state: 'error'; message: string }

export default function ItemsPage({ loadItems }: { loadItems: LoadItems }) {
  const [result, setResult] = useState<Result>({ state: 'loading' })
  const active = useRef<AbortController | null>(null)
  const headingId = useId()

  const load = useCallback(async () => {
    active.current?.abort()
    const controller = new AbortController()
    active.current = controller
    try {
      const items = await loadItems(controller.signal)
      if (!controller.signal.aborted && active.current === controller) {
        setResult({ state: 'ready', items })
      }
    } catch {
      if (!controller.signal.aborted && active.current === controller) {
        setResult({ state: 'error', message: '加载失败，请重试' })
      }
    } finally {
      if (active.current === controller) active.current = null
    }
  }, [loadItems])

  const refresh = () => {
    setResult({ state: 'loading' })
    void load()
  }

  useEffect(() => {
    void load()
    return () => {
      active.current?.abort()
      active.current = null
    }
  }, [load])

  return (
    <main className={styles.page} aria-labelledby={headingId}>
      <h1 id={headingId}>项目列表</h1>
      <button type="button" onClick={refresh} disabled={result.state === 'loading'}>刷新</button>
      <section aria-label="查询结果" aria-busy={result.state === 'loading'}>
        {result.state === 'loading' && <p role="status">加载中…</p>}
        {result.state === 'error' && <div role="alert">
          <p>{result.message}</p>
          <button type="button" onClick={refresh}>重试</button>
        </div>}
        {result.state === 'ready' && (result.items.length === 0
          ? <p>暂无数据</p>
          : <ul>{result.items.map(item => <li key={item.id}>{item.name}</li>)}</ul>)}
      </section>
    </main>
  )
}
```

### index.module.css

```css
.page { max-width: 60rem; margin: 0 auto; padding: 1rem; }
```

loadItems 即使忽略 signal，旧结果也不能回写；loadItems 必须稳定，正式页面 service 引用通常来自模块导出。例子展示首次加载/刷新/错误/空数据和卸载，不含过滤、保存或分页。验收时还应运行失败重试、迟到响应、卸载和重复挂载；示例能编译不等于这些运行行为已验证。

这个例子不通过变换 loader 表达 recordId/查询条件切换。参数化详情页须将结果与当前 ID 绑定，切换时隐藏不匹配的旧结果并显示相应加载反馈，不能只依靠请求身份保护而仍展示上一条详情。
