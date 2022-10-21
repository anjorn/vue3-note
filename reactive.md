### ref

```tsx
export {
  ref, 
  shallowRef, // 只代理浅层
  isRef, // 是否是Ref接口
  toRef, 
  toRefs,
  unref,
  proxyRefs,
  customRef,
  triggerRef,
  Ref,
  ToRef,
  ToRefs,
  UnwrapRef,
  ShallowUnwrapRef,
  RefUnwrapBailTypes
} from './ref'

// 定义Ref interface
export interface Ref<T = any> {
  value: T
  /**
   * Type differentiator only.
   * We need this to be in public d.ts but don't want it to show up in IDE
   * autocomplete, so we use a private Symbol instead.
   */
  [RefSymbol]: true
  /**
   * @internal
   */
  _shallow?: boolean
}
  
// 定义函数的参数
// 给ref添加类型变量T 泛型函数 尖括号来明确传入类型
// 参数类型<T extends object>与返回值类型
// 1 参数类型 T extends object 返回值类型 [T] extends [Ref] ? T : Ref<UnwrapRef<T>>
// 2 参数类型 T 返回值类型 Ref<UnwrapRef<T>>
// 3 参数类型 T any 返回值类型 Ref<T | undefined>
export function ref<T extends object>(
  value: T
): [T] extends [Ref] ? T : Ref<UnwrapRef<T>>
// 定义
export function ref<T>(value: T): Ref<UnwrapRef<T>>
// 
export function ref<T = any>(): Ref<T | undefined>
export function ref(value?: unknown) {
  return createRef(value, false)
}
// 最重要的是createRef 生成
function createRef(rawValue: unknown, shallow: boolean) {
  if (isRef(rawValue)) {
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
}

class RefImpl<T> {
  private _value: T
  private _rawValue: T

  public dep?: Dep = undefined
  public readonly __v_isRef = true // 加标记位__v_isRef

  constructor(value: T, public readonly _shallow: boolean) {
    this._rawValue = _shallow ? value : toRaw(value)
    this._value = _shallow ? value : toReactive(value)
   	this._shallow = _shallow
  }

  get value() {
    trackRefValue(this)
    return this._value
  }

  set value(newVal) {
    newVal = this._shallow ? newVal : toRaw(newVal)
    if (hasChanged(newVal, this._rawValue)) { // hasChanged 是通过Object.is(a, b)
      this._rawValue = newVal
      this._value = this._shallow ? newVal : toReactive(newVal)
      triggerRefValue(this, newVal)
    }
  }
}

export function trackRefValue(ref: RefBase<any>) {
  if (isTracking()) {
    // 变成原始数据 就不用ref.value.dep
    ref = toRaw(ref)
    if (!ref.dep) {
      ref.dep = createDep()
    }
    if (__DEV__) {
      trackEffects(ref.dep, {
        target: ref,
        type: TrackOpTypes.GET,
        key: 'value'
      })
    } else {
      // 关键在于依赖收集 dep管理
      trackEffects(ref.dep)
    }
  }
}

export function triggerRefValue(ref: RefBase<any>, newVal?: any) {
  ref = toRaw(ref)
  if (ref.dep) {
    if (__DEV__) {
      triggerEffects(ref.dep, {
        target: ref,
        type: TriggerOpTypes.SET,
        key: 'value',
        newValue: newVal
      })
    } else {
      // 触发视图更新
      triggerEffects(ref.dep)
    }
  }
}
```

### reactive

```js
export {
  reactive,
  readonly, // 只读 第二个参数为false
  isReactive,
  isReadonly,
  isProxy,
  shallowReactive,
  shallowReadonly,
  markRaw,
  toRaw,
  ReactiveFlags,
  DeepReadonly,
  ShallowReactive,
  UnwrapNestedRefs
} from './reactive'

// 定义可枚举值 加标记位 值得学习
export const enum ReactiveFlags {
  SKIP = '__v_skip', // 跳过 ？？ 什么情况下
  IS_REACTIVE = '__v_isReactive', // 代理后的数据 响应式
  IS_READONLY = '__v_isReadonly', // 只读不进行代理
  RAW = '__v_raw' // 原始数据
} 
// 四种类型的代理 四种类型的处理 代理Map  浅层代理  只读  浅层只读Map 对应有四个map,4种handles
// targte -> reactive
// target -> shallowReactive
// target -> reactive -> readonly
// target -> reactive -> shallowReadonly
export const reactiveMap = new WeakMap<Target, any>() // 让数据变成响应式的
export const shallowReactiveMap = new WeakMap<Target, any>() // 让数据第一层变成响应式的
export const readonlyMap = new WeakMap<Target, any>() // 让一个响应式数据变成只读的
export const shallowReadonlyMap = new WeakMap<Target, any>() // 让一个响应式数据 只有第一层是可读的

// 传入target类型 支持(Obejct, Array), (Map, Set, WeakMap, WeakSet) 普通数据 和 集合数据 分别有4种handlers 总共有8种
const enum TargetType {
  INVALID = 0,
  COMMON = 1,
  COLLECTION = 2
}

function targetTypeMap(rawType: string) {
  switch (rawType) {
    case 'Object':
    case 'Array':
      return TargetType.COMMON
    case 'Map':
    case 'Set':
    case 'WeakMap':
    case 'WeakSet':
      return TargetType.COLLECTION
    default:
      return TargetType.INVALID
  }
}

// 总共有8种处理类型
// reactive对应mutable
// readonly对应readonly
// 四种类型

function reactive(target) {
  // 判断是否是readonly 标记位判断 直接返回 因为区分readonly
  if (target[ReactiveFlags.IS_READONLY]) {
    return target
  }
  // target 是否可读 普通处理 集合处理 存储对象
  return createReactiveObject(target, false, mutableHandlers, mutableCollectionHandlers, reactiveMap)
}

function readOnly(target) {
  return createReactiveObject(
    target,
    true,
    readonlyHandlers,
    readonlyCollectionHandlers,
    readonlyMap
  )
}

function shallowReactive(target) {
  return createReactiveObject(
    target,
    false,
    shallowReactiveHandlers,
    shallowCollectionHandlers,
    shallowReactiveMap
  )
}

function shallowReadonly(target) {
  return createReactiveObject(
    target,
    true,
    shallowReadonlyHandlers,
    shallowReadonlyCollectionHandlers,
    shallowReadonlyMap
  )
}


function createReactiveObject(target, isReadOnly, baseHandler, collectionHandlers, proxyMap) {
  // 不符合要求的target 直接返回 isObejct
  // target直接是proxy（通过标记位判断） 直接返回它 
  // 只读 且 当前是reactive过了（通过标记位判断） 直接返回它
  // 已经转换过了 有缓存  要求双向存储  返回转换过的
  // 判断类型是无效的 不在白名单内 返回target
  // 真正代理
  // 在proxyMap存储
  // 返回代理后的proxy
}



```

