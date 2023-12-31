# reactive API

> 原始值指的是 Boolean、Number、BigInt、String、undefined、null等类型的值。

由于 Proxy的 代理目标必须是非原始值，所以我们没有任何手段拦截对原始值的操作，对于这个问题，我们可以使用一个非原始值包裹原始值，如：

```ts
const wrapper = {
    value: 'vue'
}
// 可以使用 Proxy 代理 wrapper，间接实现对原始值的拦截
const name = reactive(wrapper);
name.value // vue
// 修改值可以触发响应
name.value = "vue3"
```

我们看看vue中 ref 的代码：

```js
function ref(value?: unknown) {
  return createRef(value, false)
}

function createRef(rawValue: unknown, shallow: boolean) {
  if (isRef(rawValue)) {
    // 如果传入的就是一个 ref， 那么返回其自身即可， 处理嵌套 ref 的情况
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
}

function isRef(r: any): r is Ref {
  return !!(r && r.__v_isRef === true)
}

class RefImpl<T> {
  private _value: T
  private _rawValue: T

  public dep?: Dep = undefined
  public readonly __v_isRef = true

  constructor(value: T, public readonly __v_isShallow: boolean) {
    this._rawValue = __v_isShallow ? value : toRaw(value)
    this._value = __v_isShallow ? value : toReactive(value)
  }

  get value() {
    trackRefValue(this)
    return this._value
  }

  set value(newVal) {
    const useDirectValue =
      this.__v_isShallow || isShallow(newVal) || isReadonly(newVal)
    newVal = useDirectValue ? newVal : toRaw(newVal)
    if (hasChanged(newVal, this._rawValue)) {
      this._rawValue = newVal
      this._value = useDirectValue ? newVal : toReactive(newVal)
      triggerRefValue(this, newVal)
    }
  }
}
```

ref 函数返回了执行 createRef 函数的返回值。 在 createRef 内部， 首先处理了 嵌套 ref 的情况, 接着返回了 RefImpl 对象的实例

RefImpl 内部的实现主要是劫持其实例 value 属性的 getter 和 setter。

当访问一个 ref 对象的 value 属性时，会触发 getter， 执行 track 函数做依赖收集让后返回它的值；当修改yige ref 对象的 value时，则会触发 setter， 设置新值并且执行 trigger函数派发通知；如果新值 newVal 是对象或者数组类型，那么把它转换成一个 reactive 对象。

接下来看看 trackRefValue 的实现：

```ts
type RefBase<T> = {
  dep?: Dep
  value: T
}

function trackRefValue(ref: RefBase<any>) {
  if (shouldTrack && activeEffect) {
    ref = toRaw(ref)
    if (__DEV__) {
      trackEffects(ref.dep || (ref.dep = createDep()), {
        target: ref,
        type: TrackOpTypes.GET,
        key: 'value'
      })
    } else {
      trackEffects(ref.dep || (ref.dep = createDep()))
    }
  }
}
```

可以看到， 这里直接把 ref 的相关依赖保存到了 dep 属性中， 而在 track 函数中，会把依赖保存到全局的 tragetMap 中

```ts
 let depsMap = targetMap.get(target)
 if (!depsMap) {
     targetMap.set(target, (depsMap = new Map()))
 }
 let dep = depsMap.get(key)
 if (!dep) {
    // 每个 key 对应一个 dep 集合
    depsMap.set(key, (dep = createDep()))
 }
```

显然， track 函数内部可能需要做多次判断和设置逻辑，而把依赖保存到 ref 对象的dep属性中则省去了这一系列判断和设置，从而优化了性能 

ref 实现的派发通知为 triggerRefValue，它的实现是：

```ts
function triggerRefValue(ref: RefBase<any>, newVal?: any) {
  ref = toRaw(ref)
  const dep = ref.dep
  if (dep) {
    if (__DEV__) {
      triggerEffects(dep, {
        target: ref,
        type: TriggerOpTypes.SET,
        key: 'value',
        newValue: newVal
      })
    } else {
      triggerEffects(dep)
    }
  }
}
function triggerEffects(
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
    if (effect.scheduler) {
      effect.scheduler()
    } else {
      effect.run()
    }
  }
}
```

由于能直接从ref属性中获取其所有的依赖且遍历执行，不需要执行 trigger 函数的一些额外逻辑，因此性能得到了提升。

## unref

如果参数是 ref，则返回内部值，否则返回参数本身。这是 `val = isRef(val) ? val.value : val` 计算的一个语法糖。

```ts
function unref<T>(ref: MaybeRef<T>): T {
  return isRef(ref) ? ref.value : ref
}

function isRef(r: any): r is Ref {
  return !!(r && r.__v_isRef === true)
}
```

## shallowReactive API

顾名思义，相对于 reactive API，shallowReactive API 会对对象做一层浅的响应式处理

```ts
function shallowReactive<T extends object>(
  target: T
): ShallowReactive<T> {
  return createReactiveObject(
    target,
    false,
    shallowReactiveHandlers,
    shallowCollectionHandlers,
    shallowReactiveMap
  )
}
```

可以看到，对于普通对象和数组类型数据的Proxy处理器对象，shallowReactive 函数传入的 baseHandles 的值是 shallowReactiveHandlers

接下来，我们看看 shallowReactiveHandlers 的实现

```ts
const shallowReactiveHandlers = /*#__PURE__*/ extend(
  {},
  mutableHandlers,
  {
    get: shallowGet,
    set: shallowSet
  }
)
```

shallowReactiveHandlers 是在 mutableHandlers 基础上进行扩展， 修改了 get 和 set 函数的实现

我们重点关注 shallowGet 的实现， 它其实 也是通过 createGetter 函数创建的 getter，只不过第二个参数shallow 设置的是 true

```ts
const shallowGet = /*#__PURE__*/ createGetter(false, true)

function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    ...
    // 如果 shallow 为 true,直接返回
    if (shallow) {
      return res
    }

    if (isRef(res)) {
      // ref unwrapping - skip unwrap for Array + integer key.
      return targetIsArray && isIntegerKey(key) ? res : res.value
    }

    if (isObject(res)) {
      // Convert returned value into a proxy as well. we do the isObject check
      // here to avoid invalid value warning. Also need to lazy access readonly
      // and reactive here to avoid circular dependency.
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```

可以看到，一旦把 shallow 设置为 true， 即使 res 值 是 对象类型，也不会通过递归把它变成响应式的

## readonly API

如果用 const 声明一个对象变量，虽然不能直接对这个变量赋值，但可以修改它的属性。因此我们希望创建只读对象，既不能修改它的属性，也不能为其添加或删除属性。

显然，想实现上述需求就要劫持对象，Vue.js 在 reactive API 的基础上设计并实现了 readonly api：

```ts
function readonly<T extends object>(
  target: T
): DeepReadonly<UnwrapNestedRefs<T>> {
  return createReactiveObject(
    target,
    true,
    readonlyHandlers,
    readonlyCollectionHandlers,
    readonlyMap
  )
}
```

readonly 和 reactive 函数的主要区别：

* 执行 createReactiveObject 函数时的参数 isReadonly 不同

* baseHandlers 和 collectionHandlers 的区别，readonly 函数传入值是 readonlyHandlers 和readonlyCollectionHandlers

* 在 执行 createReactiveObject 时，如果 isReadonly 为 true， 且传递的参数已经是响应式对象，那么仍然可以把这个响应式对象变成一个只读对象
  
  ```ts
    if (
      target[ReactiveFlags.RAW] &&
      !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
    ) {
      return target
    }
  ```

接下来，看看 readonlyHandlers 的实现：

```ts
const readonlyHandlers: ProxyHandler<object> = {
  get: readonlyGet,
  set(target, key) {
    if (__DEV__) {
      warn(
        `Set operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  },
  deleteProperty(target, key) {
    if (__DEV__) {
      warn(
        `Delete operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  }
}
```

readonHandlers 和 mutableHandles 的区别 主要在于 get、set和deleteProperty 这三个函数，显然，只读响应式对象是不允许修改和删除属性的，所以dev环境下，set和deleteProperty函数的实现会发出警告，提示用户对象是只读的

下面是 readonlyGet 的实现：

```ts
const readonlyGet = /*#__PURE__*/ createGetter(true)
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
     //...
    // isReadonly 为 true， 则不需要收集依赖
    if (!isReadonly) {
      track(target, TrackOpTypes.GET, key)
    }
    // ...

    if (isObject(res)) {
      // 如果 res 是对象或者数组类型，则递归执行 readonly 函数把 res变成 只读的
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```

可以看到， readonly API 和 reactive APi 最大的区别就是前者不做依赖收集。
