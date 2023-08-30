## reactive

```ts
function reactive(target: object) {
  // 如果target是只读，直接返回
  if (isReadonly(target)) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap
  )
}

/**
* target 待变成响应式对象的目标对象
* isReadonly 是否创建只读的响应式对象
* baseHandlers 普通对象和数组类型数据的响应式处理器
* collectionHandlers 集合类型数据的响应式处理器
* proxyMap 原始对象和响应式对象的缓存映射图
*/
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {
  // target必须是对象或数组类型
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // target 已经是一个 Proxy对象，直接返回，
  // 有个例外，如果在一个响应式对象上调用 readonly()方法，则继续
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }
  // 如果已经有对应的Proxy，直接返回
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }
  // 只有在白名单的类型数据才能变成响应式的
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) {
    return target
  }
  // 利用 Proxy 创建响应式对象
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  // 缓存已经代理的对象
  proxyMap.set(target, proxy)
  return proxy
}
```

reactive 内部通过 createReactiveObject 函数把target变成了一个响应式对象。

在这个过程中，createReactiveObject 函数主要做了以下几件事：

1. 首先判断 target 是不是数组或函数对象类型，如果不是直接返回，所以原始数据 target 必须是对象或者数组。

2. 如果对一个已经是响应式的对象再次执行reactive，还应该返回这个响应式对象
   
   ```vue
   <script>
     import { reactive } from 'vue';
     const target = {foo: 1}
     const obj1 = reactive(target)
     const obj2 = reactive(obj1)
     console.log(obj1 === obj2) // true
   </script>
   ```
   
   可以看到， obj1 与 obj2还是同一个对象引用
   
   因为这里 reactive 函数会通过 target.__v_raw 属性来判断 target 是否已经是一个响应式对象（因为响应式对象的 __v_raw 属性会指向它自身，后面会提到），如果是的话则直接返回响应式对象。

3. 如果对同一个原始数据多次执行 reactive ，那么会返回相同的响应式对象，举个例子：
   
   ```js
   <script>
    import { reactive } from 'vue';
    const target = {foo: 1}
    const obj1 = reactive(target)
    const obj2 = reactive(target)
    console.log(obj1 === obj2) // true
   </script>
   ```
   
   可以看到， 原始对象 target 被反正执行 reactive，但 obj1和 obj2 是同一个对象

4. 对原始对象的类型做进一步限制
   
   在 createReactiveObject 内部， 会通过执行 getTargetType 函数判断数据类型
   
   ```js
   function getTargetType(value: Target) {
     return value[ReactiveFlags.SKIP] || !Object.isExtensible(value)
       ? TargetType.INVALID
       : targetTypeMap(toRawType(value))
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
   ```
   
   getTargetType 函数首先会检测对象是否有 __v_skip 属性，已经对象是否不可扩展：满足其中之一则返回0， 表示该对象不合法；都不满足则通过 targetTypeMap 来判断— 对于普通对象或数组返回1，集合类型的对象返回2，其他返回0

5. 通过 Proxy 劫持 target 对象，把它变成响应式。我们把 Proxy 函数返回的结果称作响应式对象，这里 Proxy 对应的处理器对象会根据数据类型的不同而不同，我们稍后会重点分析基本数据类型的 Proxy 处理器对象，reactive 函数传入的 baseHandlers 值是 mutableHandlers。

6. 通过 proxyMap 存储 原始对象 和 响应对象， 这就是同一个原始对象多次执行 reactive 函数却返回同一个响应式对象的原因。

## mutableHandlers

```js
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys
}
```

它其实就是劫持了我们对 observed 对象的一些操作，比如：

* 访问对象属性会触发 get 函数；

* 设置对象属性会触发 set 函数；

* 删除对象属性会触发 deleteProperty 函数；

* in 操作符会触发 has 函数；

* 通过 Object.getOwnPropertyNames 访问对象属性名会触发 ownKeys 函数。

无论命中哪个处理器函数，它都会做收集依赖和派发通知这2件事的其中之一，而依赖收集和派发通知就是响应式的精髓。

## 依赖收集：get 函数

**依赖收集发生在数据访问的阶段**，由于我们用 Proxy API 劫持了数据对象，所以当这个响应式对象属性被访问的时候就会执行 get 函数。

```js
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (key === ReactiveFlags.IS_SHALLOW) {
      return shallow
    } else if (
      key === ReactiveFlags.RAW &&
      receiver ===
        (isReadonly
          ? shallow
            ? shallowReadonlyMap
            : readonlyMap
          : shallow
          ? shallowReactiveMap
          : reactiveMap
        ).get(target)
    ) {
      return target
    }

    const targetIsArray = isArray(target)

    if (!isReadonly) {
        // arrayInstrumentations 包含对数组一些方法修改的函数
      if (targetIsArray && hasOwn(arrayInstrumentations, key)) {
        return Reflect.get(arrayInstrumentations, key, receiver)
      }
      if (key === 'hasOwnProperty') {
        return hasOwnProperty
      }
    }

    // 求值
    const res = Reflect.get(target, key, receiver)

     // 内置 Symbol key 不需要依赖收集
    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }

    // 依赖收集
    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }
    // 如果时浅响应，则直接返回原始值
    if (shallow) {
      return res
    }

    if (isRef(res)) {
      // ref unwrapping - skip unwrap for Array + integer key.
      return targetIsArray && isIntegerKey(key) ? res : res.value
    }
     // 如果res是数组或对象， 调用 reactive将结果包装成响应式数据并返回
    if (isObject(res)) {

      // 如果数据为只读，则调用 readonly 对值进行包装
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```

结合上述代码来看，get 函数主要做了四件事情，

* **首先对特殊的 key 做了代理**，这就是为什么我们在 createReactiveObject 函数中判断响应式对象是否存在 __v_raw 属性，如果存在就返回这个响应式对象本身。

* 如果 如果 target 是数组且 key 命中了 arrayInstrumentations，则执行对应的函数，我们可以大概看一下 arrayInstrumentations 的实现：
  
  ```ts
  const arrayInstrumentations = /*#__PURE__*/ createArrayInstrumentations()
  
  function createArrayInstrumentations() {
    const instrumentations: Record<string, Function> = {}
    ;(['includes', 'indexOf', 'lastIndexOf'] as const).forEach(key => {
      instrumentations[key] = function (this: unknown[], ...args: unknown[]) {
        // toRaw 可以把响应式对象转成原始数据
        const arr = toRaw(this) as any
        for (let i = 0, l = this.length; i < l; i++) {
          // 依赖收集
          track(arr, TrackOpTypes.GET, i + '')
        }
        // 先尝试用参数本身，可能是响应式数据
        const res = arr[key](...args)
        if (res === -1 || res === false) {
          // 如果失败，再尝试把参数转成原始数据
          return arr[key](...args.map(toRaw))
        } else {
          return res
        }
      }
    })
  
    // 避免某些情况下长度改变导致无限循环问题
    ;(['push', 'pop', 'shift', 'unshift', 'splice'] as const).forEach(key => {
      instrumentations[key] = function (this: unknown[], ...args: unknown[]) {
        pauseTracking()
        const res = (toRaw(this) as any)[key].apply(this, args)
        resetTracking()
        return res
      }
    })
    return instrumentations
  }
  ```
  
  也就是说，当 target 是一个数组的时候，我们去访问 target.includes、target.indexOf 或者 target.lastIndexOf 就会执行 arrayInstrumentations 代理的函数，除了调用数组本身的方法求值外，还对数组每个元素做了依赖收集。因为一旦数组的元素被修改，数组的这几个 API 的返回结果都可能发生变化，所以我们需要跟踪数组每个元素的变化。

* 第三步就是通过 **Reflect.get 求值，然后会执行 track 函数收集依赖**，我们稍后重点分析这个过程。

* 函数最后会对**计算的值 res 进行判断**，如果它也是数组或对象，则递归执行 reactive 把 res 变成响应式对象。这么做是因为 Proxy 劫持的是对象本身，并不能劫持子对象的变化，这点和 Object.defineProperty API 一致。但是 Object.defineProperty 是在初始化阶段，即定义劫持对象的时候就已经递归执行了，而 Proxy 是在对象属性被访问的时候才递归执行下一步 reactive，这其实是一种延时定义子对象响应式的实现，在性能上会有较大的提升。

整个 get 函数最核心的部分其实是执行 **track 函数收集依赖**，下面我们重点分析这个过程。

我们先来看一下 track 函数的实现：

```ts
// 是否应该收集依赖
let shouldTrack = true
// 当前激活的 effect
let activeEffect
// 原始数据对象 map
const targetMap = new WeakMap<any, KeyToDepMap>()

/**
* target 原始数据
* type 表示这次依赖收集的类型
* key 访问的属性
*/
export function track(target: object, type: TrackOpTypes, key: unknown) {
  if (shouldTrack && activeEffect) {
    let depsMap = targetMap.get(target)
    if (!depsMap) {
      // 每个 target 对应一个 depsMap
      targetMap.set(target, (depsMap = new Map()))
    }
    let dep = depsMap.get(key)
    if (!dep) {
      // 每个 key 对应一个 dep 集合
      depsMap.set(key, (dep = createDep()))
    }

    const eventInfo = __DEV__
      ? { effect: activeEffect, target, type, key }
      : undefined

    trackEffects(dep, eventInfo)
  }
}

export function trackEffects(
  dep: Dep,
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {
  let shouldTrack = false
  if (effectTrackDepth <= maxMarkerBits) {
    if (!newTracked(dep)) {
      dep.n |= trackOpBit // 标记新依赖
      // 如果依赖已经被收集，就不需要再次收集
      shouldTrack = !wasTracked(dep)
    }
  } else {
    // cleanup 模式
    shouldTrack = !dep.has(activeEffect!)
  }

  if (shouldTrack) {
    // 收集当前激活的 effect 作为依赖
    dep.add(activeEffect!)
    // 当前激活的 effect 收集 dep 集合作为依赖
    activeEffect!.deps.push(dep)
    if (__DEV__ && activeEffect!.onTrack) {
      activeEffect!.onTrack(
        extend(
          {
            effect: activeEffect!
          },
          debuggerEventExtraInfo!
        )
      )
    }
  }
}
/**
* 
*/
export const createDep = (effects?: ReactiveEffect[]): Dep => {
  const dep = new Set<ReactiveEffect>(effects) as Dep
  // w 记录已经被收集的依赖
  dep.w = 0
  // n 记录新的依赖
  dep.n = 0
  return dep
}
// 判断依赖是否已经被收集
const wasTracked = (dep: Dep): boolean => (dep.w & trackOpBit) > 0
// 判断依赖是否已经是新依赖
export const newTracked = (dep: Dep): boolean => (dep.n & trackOpBit) > 0
```

在 dep 把当前激活的 effect 作为依赖收集之前，会判断 dep是否已经收集了该 effect，如果是，则不需要再次收集。此外 ，这里还会判断当前激活的 effect 有没有被标记为新的依赖，如果没有，则将其标记为新的依赖

我们的目的是实现响应式，就是当数据变化的时候可以自动做一些事情，比如执行某些函数，所以我们**收集的依赖就是数据变化后执行的副作用函数。**

再来看实现，我们把 target 作为原始的数据，key 作为访问的属性。我们创建了全局的 targetMap 作为原始数据对象的 Map，它的键是 target，值是 depsMap，作为依赖的 Map；这个 depsMap 的键是 target 的 key，值是 dep 集合，dep 集合中存储的是依赖的副作用函数。为了方便理解，可以通过下图表示它们之间的关系：

![img](/Users/yxh/Documents/vue/vue源码阅读/img/targetMap.png)

所以每次 track ，就是把当前激活的副作用函数 activeEffect 作为依赖，然后收集到 target 相关的 depsMap 对应 key 下的依赖集合 dep 中。

## 派发通知：set 函数

**派发通知发生在数据更新的阶段** ，由于我们用 Proxy API 劫持了数据对象，所以当这个响应式对象属性更新的时候就会执行 set 函数。我们来看一下 set 函数的实现，它是执行 createSetter 函数的返回值：

```ts
const set = /*#__PURE__*/ createSetter()

function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    // 获取旧值
    let oldValue = (target as any)[key]
    if (isReadonly(oldValue) && isRef(oldValue) && !isRef(value)) {
      return false
    }

    if (!shallow) {
      if (!isShallow(value) && !isReadonly(value)) {
        // 通过原始对象获取旧值和当前值
        oldValue = toRaw(oldValue)
        value = toRaw(value)
      }
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // 浅响应模式，对象将原样设置

    }
    // 通过hadKey 判断是添加还是修改属性
    const hadKey =
      // isIntegerKey 判断key是否是数组
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
    // 获取结果
    const result = Reflect.set(target, key, value, receiver)
    //  如果target的原型链也是一个 proxy，通过 Reflect.set 
    // 修改原型链上的属性会再次触发 setter，这种情况下就没必要触发两次 trigger 了
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
```

结合上述代码来看，set 函数的实现逻辑很简单，主要就做两件事情， 首先**通过 Reflect.set 求值** ，**然后通过 trigger 函数派发通知**  ，并依据 key 是否存在于 target 上来确定通知类型，即新增还是修改。

整个 **set 函数最核心的部分就是 执行 trigger 函数派发通知** ，下面我们将重点分析这个过程。

我们先来看一下 trigger 函数的实现，为了分析主要流程，这里省略了 trigger 函数中的一些分支逻辑：

```ts
// 原始数据对象 map
const targetMap = new WeakMap()

export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  // 通过 targetMap 拿到 target 对应的依赖集合
  const depsMap = targetMap.get(target)
  if (!depsMap) {
    // 没有依赖，直接返回
    return
  }
  // effect 集合
  let deps = []
  if (type === TriggerOpTypes.CLEAR) {
    // 待清空的effect
    deps = [...depsMap.values()]
  } else if (key === 'length' && isArray(target)) {
    // target 为数组且 key为length
    const newLength = Number(newValue)
    depsMap.forEach((dep, key) => {
      if (key === 'length' || key >= newLength) {
        deps.push(dep)
      }
    })
  } else {
    //  SET | ADD | DELETE 操作之一，添加对应的 effects
    if (key !== void 0) {
      deps.push(depsMap.get(key))
    }

    switch (type) {
      case TriggerOpTypes.ADD:
        // target 不是数组
        if (!isArray(target)) { 
          deps.push(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            deps.push(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        } else if (isIntegerKey(key)) {
          // 数组新增下标触发的数组长度变化
          deps.push(depsMap.get('length'))
        }
        break
      case TriggerOpTypes.DELETE:
        if (!isArray(target)) {
          deps.push(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            deps.push(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        }
        break
      case TriggerOpTypes.SET:
        if (isMap(target)) {
          deps.push(depsMap.get(ITERATE_KEY))
        }
        break
    }
  }

  const eventInfo = __DEV__
    ? { target, type, key, newValue, oldValue, oldTarget }
    : undefined

   // deps 只有一个元素
  if (deps.length === 1) {
    if (deps[0]) {
      if (__DEV__) {
        triggerEffects(deps[0], eventInfo)
      } else {
        triggerEffects(deps[0])
      }
    }
  } else {
    const effects: ReactiveEffect[] = []
    for (const dep of deps) {
      if (dep) {
        effects.push(...dep)
      }
    }
    if (__DEV__) {
      triggerEffects(createDep(effects), eventInfo)
    } else {
      triggerEffects(createDep(effects))
    }
  }
}


export function triggerEffects(
  dep: Dep | ReactiveEffect[],
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {
  // spread into array for stabilization
  const effects = isArray(dep) ? dep : [...dep]
  for (const effect of effects) {
    if (effect.computed) {
      triggerEffect(effect, debuggerEventExtraInfo)
    }
  }
  for (const effect of effects) {
    if (!effect.computed) {
      triggerEffect(effect, debuggerEventExtraInfo)
    }
  }
}

function triggerEffect(
  effect: ReactiveEffect,
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {
  if (effect !== activeEffect || effect.allowRecurse) {
    if (__DEV__ && effect.onTrigger) {
      effect.onTrigger(extend({ effect }, debuggerEventExtraInfo))
    }
    // 如果一个副作用函数存在调度器，则调用该调度器
    if (effect.scheduler) {
      effect.scheduler()
    } else {
      // 直接运行
      effect.run()
    }
  }
}
```

trigger 函数的实现也很简单，主要做了四件事情：

* 通过 targetMap 拿到 target 对应的依赖集合 depsMap；

* 创建运行的 effects 集合；

* 根据 key 从 depsMap 中找到对应的 effects 添加到 effects 集合；

* 遍历 effects 执行相关的副作用函数。

所以每次 trigger 函数就是根据 target 和 key ，从 targetMap 中找到相关的所有副作用函数遍历执行一遍。

在描述依赖收集和派发通知的过程中，我们都提到了一个词：副作用函数，依赖收集过程中我们把 activeEffect（当前激活副作用函数）作为依赖收集，它又是什么？接下来我们来看一下副作用函数的庐山真面目。

## 副作用函数

vue内部有一个effect副作用函数：

```ts
// 给定函数立即执行一次； 每次属性更新都会执行一次
export function effect<T = any>(
  fn: () => T,
  options?: ReactiveEffectOptions
): ReactiveEffectRunner {
  if ((fn as ReactiveEffectRunner).effect) {
      // 如果fn已经是一个effect函数，则指向原始函数
    fn = (fn as ReactiveEffectRunner).effect.fn
  }
  // 创建 _effect 实例
  const _effect = new ReactiveEffect(fn)
  if (options) {
    // 把 options 中的属性复制到 _effect 中
    extend(_effect, options)
    if (options.scope) recordEffectScope(_effect, options.scope)
  }
  if (!options || !options.lazy) {
    // 立即执行
    _effect.run()
  }
  // 绑定 run 函数，作为 effect runner
  const runner = _effect.run.bind(_effect) as ReactiveEffectRunner
  // 在 runner 中保留对 _effect 的引用
  runner.effect = _effect
  return runner
}
export class ReactiveEffect<T = any> {
  // 是否活跃
  active = true
  // dep 数组，在响应式对象收集依赖时也会将对应的依赖项添加到这个数组中
  deps: Dep[] = []
  // 上一个 ReactiveEffect 的实例
  parent: ReactiveEffect | undefined = undefined

  // 创建后可能会附加的属性，如果是 computed 则指向 ComputedRefImpl
  computed?: ComputedRefImpl<T>
  // 是否允许递归，会被外部更改，
  allowRecurse?: boolean
  // 延迟停止
  private deferStop?: boolean
  // 停止事件
  onStop?: () => void
  // dev only
  onTrack?: (event: DebuggerEvent) => void
  // dev only
  onTrigger?: (event: DebuggerEvent) => void

  constructor(
    // 参数赋值给fn
    public fn: () => T,
     // 参数赋值给 scheduler
    public scheduler: EffectScheduler | null = null,
    scope?: EffectScope
  ) {
    // effectScope 的相关处理逻辑
    recordEffectScope(this, scope)
  }

  run() {
    // 如果处于非激活状态直接执行原始函数
    if (!this.active) {
      return this.fn()
    }
    // 存储最上层 ReactiveEffect 对象
    let parent: ReactiveEffect | undefined = activeEffect
    // 缓存 是否可以跟踪依赖 上一次的结果
    let lastShouldTrack = shouldTrack
    while (parent) {
      if (parent === this) {
        return
      }
      parent = parent.parent
    }
    try {
      // 父结点指向上一个 ReactiveEffect
      this.parent = activeEffect
      // 当前活跃的 ReactiveEffect
      activeEffect = this
      // 允许追踪依赖
      shouldTrack = true
      // 定义当前的 ReactiveEffect 层级
      trackOpBit = 1 << ++effectTrackDepth
      // 当前层级没超过最大层级限制
      if (effectTrackDepth <= maxMarkerBits) {
        // 给依赖打上标记
        initDepMarkers(this)
      } else {
        // 清除副作用，一般不会触发
        cleanupEffect(this)
      }
      // 执行构造函数传入的方法
      return this.fn()
    } finally {
      // 当前层级没超过最大层级限制，清空标记
      if (effectTrackDepth <= maxMarkerBits) {
        finalizeDepMarkers(this)
      }
      // 回到上一层 ReactiveEffect
      trackOpBit = 1 << --effectTrackDepth

      activeEffect = this.parent
      shouldTrack = lastShouldTrack
      this.parent = undefined

      if (this.deferStop) {
        this.stop()
      }
    }
  }

  stop() {
    // stopped while running itself - defer the cleanup
    if (activeEffect === this) {
      this.deferStop = true
    } else if (this.active) {
      cleanupEffect(this)
      if (this.onStop) {
        this.onStop()
      }
      this.active = false
    }
  }
}
export function recordEffectScope(
  effect: ReactiveEffect,
  scope: EffectScope | undefined = activeEffectScope
) {
  if (scope && scope.active) {
    scope.effects.push(effect)
  }
}
```

上面代码注释对 ReactiveEffect 的变量做了一些说明，我们再介绍下其中比较关键的变量：

- trackOpBit: 一个二进制变量，标识依赖收集的状态
  - 在 ReactiveEffect 嵌套执行时产生，比如在 computed 中执行 computed，就会产生嵌套层级。
- effectTrackDepth: 表示 递归嵌套执行effect函数的深度。
  - 每产生一个嵌套层级就 +1 等到对应的嵌套层级执行完后就会回到上个层级的标记数。
- activeEffect 当前活跃的副作用对象。
  - 会在 ref、reactive、computed 收集依赖时作为依赖项添加到对应的dep集合中
* maxMarkerBits 最大标记的位数

effect内部通过 ReactiveEffect 创建一个 _effect对象，并且函数返回的 runner 指向 ReactiveEffect 类的run函数。也就是说，在执行付做好函数 effect 时，实际上执行的就是这个 run 函数

当 run 函数执行的时候，首先会记录 trackOpBit， 然后看递归深度是否超过了 maxMarkerBits, 如果超过了，仍然执行 cleanup 逻辑；如果没超过，则执行 initDepMarkers 给依赖打标记

```js
 const initDepMarkers = ({ deps }: ReactiveEffect) => {
  if (deps.length) {
    for (let i = 0; i < deps.length; i++) {
      deps[i].w |= trackOpBit // 标记当前层的依赖已经被收集
    }
  }
}
```

initDepMarkers 函数的实现很简单， 遍历 _effect  实例中的 deps 函数， 通过或运算 为每个 dep的 w 属性标记 trackOpBit， 表示当前层的依赖已经被收集

当 fn 函数执行的时候，会访问响应式数据， 触发它们的 getter， 进而执行 track 函数进行依赖收集。

然后我们看看 fn 执行之后的逻辑：

```js
finally {
      // 当前层级没超过最大层级限制，清空标记
      if (effectTrackDepth <= maxMarkerBits) {
        finalizeDepMarkers(this)
      }
      // 回到上一层 ReactiveEffect
      trackOpBit = 1 << --effectTrackDepth

      activeEffect = this.parent
      shouldTrack = lastShouldTrack
      this.parent = undefined

      if (this.deferStop) {
        this.stop()
      }
    }
```

在满足依赖标记的条件下，需要执行 finalizeDepMarkers 完成依赖标记，看看它的实现：

```js
const finalizeDepMarkers = (effect: ReactiveEffect) => {
  const { deps } = effect
  if (deps.length) {
    let ptr = 0
    for (let i = 0; i < deps.length; i++) {
      const dep = deps[i]
      // 曾经被收集但不是新的依赖， 需要删除
      if (wasTracked(dep) && !newTracked(dep)) {
        dep.delete(effect)
      } else {
        deps[ptr++] = dep
      }
      // 清空状态
      dep.w &= ~trackOpBit
      dep.n &= ~trackOpBit
    }
    deps.length = ptr
  }
}
```

finalizeDepMarkers 主要就是清空 dep 在当前层级的依赖状态，同时找到那些曾经被收集但是在新一轮（依赖收集）中没有被收集的依赖，并将其从 deps 中删除。
