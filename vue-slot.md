今天我们将分析我们经常使用的 vue 功能 slot 是如何设计和实现的，本文将围绕 普通插槽 和 作用域插槽 以及 vue 2.6.x 版本的 v-slot 展开对该话题的讨论。当然还不懂用法的同学建议官网先看看相关 API 先。接下来，我们直接进入正文吧

一、普通插槽
首先我们看一个我们对于 slot 最常用的例子
```
<template>
  <div class="slot-demo">
    <slot>this is slot default content text.</slot>
  </div>
</template>
```
然后我们直接使用，页面则正常显示一下内容



然后，这个时候我们使用的时候，对 slot 内容进行覆盖

```<slot-demo>this is slot custom content.</slot-demo>```
内容则变成下图所示



对于此，大家可能都能清楚的知道会是这种情况。今天我就将带领大家直接看看 vue 底层对 slot 插槽的具体实现。

1、```vm.$slots```
我们开始前，先看看 vue 的 Component 接口上对 $slots 属性的定义

```$slots: { [key: string]: Array<VNode> }; ```
多的咱不说，咱直接 console 一下上面例子中的 $slots



剩下的篇幅将讲解 slot 内容如何进行渲染以及如何转换成上图内容

2、```renderSlot```
看完了具体实例中 slot 渲染后的 vm.$slots 对象，这一小篇我们直接看看 renderSlot 这块的逻辑，首先我们先看看 renderSlot 函数的几个参数都有哪些
```
renderSlot()
export function renderSlot (
  name: string, // 插槽名 slotName
  fallback: ?Array<VNode>, // 插槽默认内容生成的 vnode 数组
  props: ?Object, // props 对象
  bindObject: ?Object // v-bind 绑定对象
): ?Array<VNode> {}
```
这里我们先不看 scoped-slot 的逻辑，我们只看普通 slot 的逻辑。
```
const slotNodes = this.$slots[name]
nodes = slotNodes || fallback
return nodes
```
这里直接先取值``` this.$slots[name] ```，若存在则直接返回其对其的 vnode 数组，否则返回 fallback。看到这，很多人可能不知道 this.$slots 在哪定义的。解释这个之前我们直接往后看另外一个方法
```
renderslots()
export function resolveSlots (
  children: ?Array<VNode>, // 父节点的 children
  context: ?Component // 父节点的上下文，即父组件的 vm 实例
): { [key: string]: Array<VNode> } {}
```
看完 resolveSlots 的参数后我们接着往后过其中具体的逻辑。如果 children 参数不存在，直接返回一个空对象
```
const slots = {}
if (!children) {
  return slots
}
```
如果存在，则直接对 children 进行遍历操作
```
for (let i = 0, l = children.length; i < l; i++) {
  const child = children[i]
  const data = child.data
  // 如果 data.slot 存在，将插槽名称当做 key，child 当做值直接添加到 slots 中去
  if ((child.context === context || child.fnContext === context) &&
    data && data.slot != null
  ) {
    const name = data.slot
    const slot = (slots[name] || (slots[name] = []))
    // child 的 tag 为 template 标签的情况
    if (child.tag === 'template') {
      slot.push.apply(slot, child.children || [])
    } else {
      slot.push(child)
    }
  // 如果 data.slot 不存在，则直接将 child 丢到 slots.default 中去
  } else {
    (slots.default || (slots.default = [])).push(child)
  }
}
```
slots 获取到值后，则进行一些过滤操作，然后直接返回有用的 slots
```
// ignore slots that contains only whitespace
for (const name in slots) {
  if (slots[name].every(isWhitespace)) {
    delete slots[name]
  }
}
return slots
```
```
// isWhitespace 相关逻辑
function isWhitespace (node: VNode): boolean {
  return (node.isComment && !node.asyncFactory) || node.text === ' '
}
```

initRender()
我们从上面已经知道了 vue 对 slots 是如何进行赋值保存数据的。而在 src/core/instance/render.js 的 initRender 方法中则是对 vm.$slots 进行了初始化的赋值。
```
const options = vm.$options
const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
const renderContext = parentVnode && parentVnode.context
vm.$slots = resolveSlots(options._renderChildren, renderContext)
genSlot()
```
了解了是 vm.$slots 这块逻辑后，肯定有人会问：你这不就只是拿到了一个对象么，怎么把其中的内容给搞出来呢？别急，我们接着就来讲一下对于 slot 这块 vue 是如何进行编译的。这里咱就把 slot generate 相关逻辑过上一过，话不多说，咱直接上代码
```
function genSlot (el: ASTElement, state: CodegenState): string {
  const slotName = el.slotName || '"default"' // 取 slotName，若无，则直接命名为 'default'
  const children = genChildren(el, state) // 对 children 进行 generate 操作
  let res = `_t(${slotName}${children ? `,${children}` : ''}`
  const attrs = el.attrs && `{${el.attrs.map(a => `${camelize(a.name)}:${a.value}`).join(',')}}` // 将 attrs 转换成对象形式
  const bind = el.attrsMap['v-bind'] // 获取 slot 上的 v-bind 属性
  // 若 attrs 或者 bind 属性存在但是 children 却木得，直接赋值第二参数为 null
  if ((attrs || bind) && !children) {
    res += `,null`
  }
  // 若 attrs 存在，则将 attrs 作为 `_t()` 的第三个参数(普通插槽的逻辑处理)
  if (attrs) {
    res += `,${attrs}`
  }
  // 若 bind 存在，这时如果 attrs 存在，则 bind 作为第三个参数，否则 bind 作为第四个参数(scoped-slot 的逻辑处理)
  if (bind) {
    res += `${attrs ? '' : ',null'},${bind}`
  }
  return res + ')'
}
```
注：上面的 slotName 在 src/compiler/parser/index.js 的 processSlot() 函数中进行了赋值，并且 父组件编译阶段用到的 slotTarget 也在这里进行了处理
```
function processSlot (el) {
  if (el.tag === 'slot') {
    // 直接获取 attr 里面 name 的值
    el.slotName = getBindingAttr(el, 'name')
    // ...
  }
  // ...
  const slotTarget = getBindingAttr(el, 'slot')
  if (slotTarget) {
    // 如果 slotTarget 存在则直接取命名插槽的 slot 值，否则直接为 'default'
    el.slotTarget = slotTarget === '""' ? '"default"' : slotTarget
    if (el.tag !== 'template' && !el.slotScope) {
      addAttr(el, 'slot', slotTarget)
    }
  }
}
```
随即在 genData() 中使用 slotTarget 进行 data 的数据拼接
```
if (el.slotTarget && !el.slotScope) {
  data += `slot:${el.slotTarget},`
}
```
此时父组件将生成以下代码
```
with(this) {
  return _c('div', [
    _c('slot-demo'),
    {
      attrs: { slot: 'default' },
      slot: 'default'
    },
    [ _v('this is slot custom content.') ]
  ])
}
```
然后当 el.tag 为 slot 的情况，则直接执行 genSlot()
```
else if (el.tag === 'slot') {
  return genSlot(el, state)
}
```
按照我们举出的例子，则子组件最终会生成以下代码
```
with(this) {
  // _c => createElement ; _t => renderSlot ; _v => createTextVNode
  return _c(
    'div',
    {
      staticClass: 'slot-demo'
    },
    [ _t('default', [ _v('this is slot default content text.') ]) ]
  )
}
```
二、作用域插槽
上面我们已经了解到 vue 对于普通的 slot 标签是如何进行处理和转换的。接下来我们来分析下作用域插槽的实现逻辑。

1、```vm.$scopedSlots```

了解之前还是老规矩，先看看 vue 的 Component 接口上对 $scopedSlots 属性的定义

```$scopedSlots: { [key: string]: () => VNodeChildren };```

其中的 VNodeChildren 定义如下：

```declare type VNodeChildren = Array<?VNode | string | VNodeChildren> | string;```

先来个相关的例子
```
<template>
  <div class="slot-demo">
    <slot text="this is a slot demo , " :msg="msg"></slot>
  </div>
</template>

<script>
export default {
  name: 'SlotDemo',
  data () {
    return {
      msg: 'this is scoped slot content.'
    }
  }
}
</script>
```
然后进行使用
```
<template>
  <div class="parent-slot">
    <slot-demo>
      <template slot-scope="scope">
        <p>{{ scope.text }}</p>
        <p>{{ scope.msg }}</p>
      </template>
    </slot-demo>
  </div>
</template>
```
效果如下



从使用层面我们能看出来，子组件的 slot 标签上绑定了一个 text 以及 :msg 属性。然后父组件在使用插槽使用了 slot-scope 属性去读取插槽带的属性对应的值

注：提及一下 processSlot() 对于 slot-scope 的处理逻辑
```
let slotScope
if (el.tag === 'template') {
  slotScope = getAndRemoveAttr(el, 'scope')
  // 兼容 2.5 以前版本 slot scope 的用法(这块有个警告，我直接忽略掉了)
  el.slotScope = slotScope || getAndRemoveAttr(el, 'slot-scope')
} else if ((slotScope = getAndRemoveAttr(el, 'slot-scope'))) {
  el.slotScope = slotScope
}
```
从上面的代码我们能看出，vue 对于这块直接读取 slot-scope 属性并赋值给 AST 抽象语法树的 slotScope 属性上。而拥有 slotScope 属性的节点，会直接以 **插槽名称 name 为 key、本身为 value **的对象形式挂载在父节点的 scopedSlots 属性上
```
else if (element.slotScope) { 
  currentParent.plain = false
  const name = element.slotTarget || '"default"'
  ;(currentParent.scopedSlots || (currentParent.scopedSlots = {}))[name] = element
}
```
然后在 src/core/instance/render.js 的 renderMixin 方法中对 vm.$scopedSlots 则是进行了如下赋值：
```
if (_parentVnode) {
  vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject
}
```
然后 genData() 里会进行以下逻辑处理
```
if (el.scopedSlots) {
  data += `${genScopedSlots(el, el.scopedSlots, state)},`
}
```
紧接着我们来看看 genScopedSlots 中的逻辑
```
function genScopedSlots (
  slots: { [key: string]: ASTElement },
  state: CodegenState
): string {
  // 对 el.scopedSlots 对象进行遍历，执行 genScopedSlot，且将结果用逗号进行拼接
  // _u => resolveScopedSlots (具体逻辑下面一个小节进行分析)
  return `scopedSlots:_u([${
    Object.keys(slots).map(key => {
      return genScopedSlot(key, slots[key], state)
    }).join(',')
  }])`
}
```
然后我们再来看看 genScopedSlot 是如何生成 render function 字符串的
```
function genScopedSlot (
  key: string,
  el: ASTElement,
  state: CodegenState
): string {
  if (el.for && !el.forProcessed) {
    return genForScopedSlot(key, el, state)
  }
  // 函数参数为标签上 slot-scope 属性对应的值 (getAndRemoveAttr(el, 'slot-scope'))
  const fn = `function(${String(el.slotScope)}){` +
    `return ${el.tag === 'template'
      ? el.if
        ? `${el.if}?${genChildren(el, state) || 'undefined'}:undefined`
        : genChildren(el, state) || 'undefined'
      : genElement(el, state)
    }}`
  // key 为插槽名称，fn 为生成的函数代码
  return `{key:${key},fn:${fn}}`
}
```
我们把上面例子的 $scopedSlots 打印一下，结果如下



然后上面例子中父组件最终会生成如下代码
```
with(this){
  // _c => createElement ; _u => resolveScopedSlots
  // _v => createTextVNode ; _s => toString
  return _c('div',
    { staticClass: 'parent-slot' },
    [_c('slot-demo',
      { scopedSlots: _u([
        {
          key: 'default',
          fn: function(scope) {
            return [
              _c('p', [ _v(_s(scope.text)) ]),
              _c('p', [ _v(_s(scope.msg)) ])
            ]
          }
        }])
      }
    )]
  )
}
```
2、renderSlot(slot-scope)
renderSlot()
上面我们提及对于插槽 render 逻辑的时候忽略了 slot-scope 的相关逻辑，这里我们来看看这部分内容
```
export function renderSlot (
  name: string,
  fallback: ?Array<VNode>,
  props: ?Object,
  bindObject: ?Object
): ?Array<VNode> {
  const scopedSlotFn = this.$scopedSlots[name]
  let nodes
  if (scopedSlotFn) { // scoped slot
    props = props || {}
    // ...
    nodes = scopedSlotFn(props) || fallback
  }
	// ...
	return nodes
}
resolveScopedSlots()
```
这里我们看看 renderHelps 里面的 _u ，即 resolveScopedSlots，其逻辑如下
```
export function resolveScopedSlots (
  fns: ScopedSlotsData, // Array<{ key: string, fn: Function } | ScopedSlotsData>
  res?: Object
): { [key: string]: Function } {
  res = res || {}
  // 遍历 fns 数组，生成一个 `key 为插槽名称，value 为函数` 的对象
  for (let i = 0; i < fns.length; i++) {
    if (Array.isArray(fns[i])) {
      resolveScopedSlots(fns[i], res)
    } else {
      res[fns[i].key] = fns[i].fn
    }
  }
  return res
}
```
genSlot
这块会对 attrs 和 v-bind 进行，对于这块内容上面我已经提过了，要看请往上翻阅。结合我们的例子，子组件则会生成以下代码
```
with(this) {
  return _c(
    'div',
    {
      staticClass: 'slot-demo'
    },
    [
      _t('default', null, { text: 'this is a slot demo , ', msg: msg })
    ]
  )
}
```
到目前为止，对于普通插槽和作用域插槽已经谈的差不多了。接下来，我们将一起看看 vue 2.6.x 版本的 v-slot

三、v-slot
1、基本用法
vue 2.6.x 已经出来有一段时间了，其中对于插槽这块则是放弃了 slot-scope 作用域插槽推荐写法，直接改成了 v-slot 指令形式的推荐写法(当然这只是个语法糖而已)。下面我们将仔细谈谈 v-slot 这块的内容。

在看具体实现逻辑前，我们先通过一个例子来先了解下其基本用法
```
<template>
  <div class="slot-demo">
    <slot name="demo">this is demo slot.</slot>
    <slot text="this is a slot demo , " :msg="msg"></slot>
  </div>
</template>

<script>
export default {
  name: 'SlotDemo',
  data () {
    return {
      msg: 'this is scoped slot content.'
    }
  }
}
</script>
```
然后进行使用
```
<template>
  <slot-demo>
    <template v-slot:demo>this is custom slot.</template>
    <template v-slot="scope">
      <p>{{ scope.text }}{{ scope.msg }}</p>
    </template>
  </slot-demo>
</template>
```
看着好 easy 。



2、相同与区别
接下来，咱来会会这个新特性

round 1.``` $slots & $scopedSlots```
$slots 这块逻辑没变，还是沿用的以前的代码
```
// $slots
const options = vm.$options
const parentVnode = vm.$vnode = options._parentVnode
const renderContext = parentVnode && parentVnode.context
vm.$slots = resolveSlots(options._renderChildren, renderContext)
$scopedSlots 这块则进行了改造，执行了 normalizeScopedSlots() 并接收其返回值为 $scopedSlots 的值

if (_parentVnode) {
  vm.$scopedSlots = normalizeScopedSlots(
    _parentVnode.data.scopedSlots,
    vm.$slots,
    vm.$scopedSlots
  )
}
```
接着，我们来会一会 normalizeScopedSlots ，首先我们先看看它的几个参数
```
export function normalizeScopedSlots (
  slots: { [key: string]: Function } | void,  // 某节点 data 属性上 scopedSlots
  normalSlots: { [key: string]: Array<VNode> }, // 当前节点下的普通插槽
  prevSlots?: { [key: string]: Function } | void // 当前节点下的特殊插槽
): any {}
```
首先，如果 slots 参数不存在，则直接返回一个空对象 {}
```
if (!slots) {
  res = {}
}
```
若 prevSlots 存在，且满足系列条件的情况，则直接返回 prevSlots
```
const hasNormalSlots = Object.keys(normalSlots).length > 0 // 是否拥有普通插槽
const isStable = slots ? !!slots.$stable : !hasNormalSlots // slots 上的 $stable 值
const key = slots && slots.$key // slots 上的 $key 值
else if (
  isStable &&
  prevSlots &&
  prevSlots !== emptyObject &&
  key === prevSlots.$key && // slots $key 值与 prevSlots $key 相等
  !hasNormalSlots && // slots 中没有普通插槽
  !prevSlots.$hasNormal // prevSlots 中没有普通插槽
) {
  return prevSlots
}
```
注：这里的 $key , $hasNormal , $stable 是直接使用 vue 内部对 Object.defineProperty 封装好的 def() 方法进行赋值的
```
def(res, '$stable', isStable)
def(res, '$key', key)
def(res, '$hasNormal', hasNormalSlots)
否则，则对 slots 对象进行遍历，操作 normalSlots ，赋值给 key 为 key，value 为 normalizeScopedSlot 返回的函数 的对象 res
let res
else {
  res = {}
  for (const key in slots) {
    if (slots[key] && key[0] !== '$') {
      res[key] = normalizeScopedSlot(normalSlots, key, slots[key])
    }
  }
}
```
随后再次对 normalSlots 进行遍历，若 normalSlots 中的 key 在 res 找不到对应的 key，则直接进行 proxyNormalSlot 代理操作，将 normalSlots 中的 slot 挂载到 res 对象上
```
for (const key in normalSlots) {
  if (!(key in res)) {
    res[key] = proxyNormalSlot(normalSlots, key)
  }
}

function proxyNormalSlot(slots, key) {
  return () => slots[key]
}
```
接着，我们看看 normalizeScopedSlot() 都做了些什么事情。该方法接收三个参数，第一个参数为 normalSlots，第二个参数为 key，第三个参数为 fn
```
function normalizeScopedSlot(normalSlots, key, fn) {
  const normalized = function () {
    // 若参数为多个，则直接使用 arguments 作为 fn 的参数，否则直接传空对象作为 fn 的参数
    let res = arguments.length ? fn.apply(null, arguments) : fn({})
    // fn 执行返回的 res 不是数组，则是单 vnode 的情况，赋值为 [res] 即可
    // 否则执行 normalizeChildren 操作，这块主要对针对 slot 中存在 v-for 操作
    res = res && typeof res === 'object' && !Array.isArray(res)
      ? [res] // single vnode
      : normalizeChildren(res)
    return res && (
      res.length === 0 ||
      (res.length === 1 && res[0].isComment) // slot 上 v-if 相关处理
    ) ? undefined
      : res
  }
  // v-slot 语法糖处理
  if (fn.proxy) {
    Object.defineProperty(normalSlots, key, {
      get: normalized,
      enumerable: true,
      configurable: true
    })
  }
  return normalized
}
```
round 2. renderSlot
这块逻辑处理其实和之前是一样的，只是删除了一些警告的代码而已。这点这里就不展开叙述了

round 3. processSlot
首先，这里解析 slot 的方法名从 processSlot 变成了 processSlotContent，但其实前面的逻辑和以前是一样的。只是新增了一些对于 v-slot 的逻辑处理，下面我们就来捋捋这块。过具体逻辑前，我们先看一些相关的正则和方法

1、相关正则 & functions
```
dynamicArgRE 动态参数匹配
const dynamicArgRE = /^\[.*\]$/ // 匹配到 '[]' 则为 true，如 '[ item ]'
slotRE 匹配 v-slot 语法相关正则
const slotRE = /^v-slot(:|$)|^#/ // 匹配到 'v-slot' 或 'v-slot:' 则为 true
getAndRemoveAttrByRegex 通过正则匹配绑定的 attr 值
export function getAndRemoveAttrByRegex (
  el: ASTElement,
  name: RegExp // 
) {
  const list = el.attrsList // attrsList 类型为 Array<ASTAttr>
  // 对 attrsList 进行遍历，若有满足 RegExp 的则直接返回当前对应的 attr
  // 若参数 name 传进来的是 slotRE = /^v-slot(:|$)|^#/
  // 那么匹配到 'v-slot' 或者 'v-slot:xxx' 则会返回其对应的 attr
  for (let i = 0, l = list.length; i < l; i++) {
    const attr = list[i]
    if (name.test(attr.name)) {
      list.splice(i, 1)
      return attr
    }
  }
}
ASTAttr 接口定义
declare type ASTAttr = {
  name: string;
  value: any;
  dynamic?: boolean;
  start?: number;
  end?: number
};
createASTElement 创建 ASTElement
export function createASTElement (
  tag: string, // 标签名
  attrs: Array<ASTAttr>, // attrs 数组
  parent: ASTElement | void // 父节点
): ASTElement {
  return {
    type: 1,
    tag,
    attrsList: attrs,
    attrsMap: makeAttrsMap(attrs),
    rawAttrsMap: {},
    parent,
    children: []
  }
}
getSlotName 获取 slotName
function getSlotName (binding) {
  // 'v-slot:item' 匹配获取到 'item'
  let name = binding.name.replace(slotRE, '')
  if (!name) {
    if (binding.name[0] !== '#') {
      name = 'default'
    } else if (process.env.NODE_ENV !== 'production') {
      warn(
        `v-slot shorthand syntax requires a slot name.`,
        binding
      )
    }
  }
  // 返回一个 key 包含 name，dynamic 的对象
  // 'v-slot:[item]' 匹配然后 replace 后获取到 name = '[item]'
  // 进而进行动态参数进行匹配 dynamicArgRE.test(name) 结果为 true
  return dynamicArgRE.test(name)
    ? { name: name.slice(1, -1), dynamic: true } // 截取变量，如 '[item]' 截取后变成 'item'
    : { name: `"${name}"`, dynamic: false }
}
```
2、processSlotContent
这里我们先看看 slot 对于 template 是如何处理的
```
if (el.tag === 'template') {
  // 匹配绑定在 template 上的 v-slot 指令，这里会匹配到对应 v-slot 的 attr(类型为 ASTAttr)
  const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
  // 若 slotBinding 存在，则继续进行 slotName 的正则匹配
  // 随即将匹配出来的 name 赋值给 slotTarget，dynamic 赋值给 slotTargetDynamic
  // slotScope 赋值为 slotBinding.value 或者 '_empty_'
  if (slotBinding) {
    const { name, dynamic } = getSlotName(slotBinding)
    el.slotTarget = name
    el.slotTargetDynamic = dynamic
    el.slotScope = slotBinding.value || emptySlotScopeToken
  }
}
```
如果不是 template，而是绑定在 component 上的话，对于 v-slot 指令和 slotName 的匹配操作是一样的，不同点在于由于这里需要将组件的 children 添加到其默认插槽中去
```
else {
  // v-slot on component 表示默认插槽
  const slotBinding = getAndRemoveAttrByRegex(el, slotRE)
  // 将组件的 children 添加到其默认插槽中去
  if (slotBinding) {
    // 获取当前组件的 scopedSlots
    const slots = el.scopedSlots || (el.scopedSlots = {})
    // 匹配拿到 slotBinding 中 name，dynamic 的值
    const { name, dynamic } = getSlotName(slotBinding)
    // 获取 slots 中 key 对应匹配出来 name 的 slot
    // 然后再其下面创建一个标签名为 template 的 ASTElement，attrs 为空数组，parent 为当前节点
    const slotContainer = slots[name] = createASTElement('template', [], el)
    // 这里 name、dynamic 统一赋值给 slotContainer 的 slotTarget、slotTargetDynamic，而不是 el
    slotContainer.slotTarget = name
    slotContainer.slotTargetDynamic = dynamic
    // 将当前节点的 children 添加到 slotContainer 的 children 属性中
    slotContainer.children = el.children.filter((c: any) => {
      if (!c.slotScope) {
        c.parent = slotContainer
        return true
      }
    })
    slotContainer.slotScope = slotBinding.value || emptySlotScopeToken
    // 清空当前节点的 children
    el.children = []
    el.plain = false
  }
}
```
这样处理后我们就可以直接在父组件上面直接使用 v-slot 指令去获取 slot 绑定的值。举个官方例子来表现一下
```
Default slot with text
<!-- old -->
<foo>
  <template slot-scope="{ msg }">
    {{ msg }}
  </template>
</foo>

<!-- new -->
<foo v-slot="{ msg }">
  {{ msg }}
</foo>
Default slot with element
<!-- old -->
<foo>
  <div slot-scope="{ msg }">
    {{ msg }}
  </div>
</foo>

<!-- new -->
<foo v-slot="{ msg }">
  <div>
    {{ msg }}
  </div>
</foo>
```

round 4. generate
genSlot() 在这块逻辑也没发生本质性的改变，唯一一个改变就是为了支持 v-slot 动态参数做了些改变，具体如下
```
// old
const attrs = el.attrs && `{${el.attrs.map(a => `${camelize(a.name)}:${a.value}`).join(',')}}`

// new
// attrs、dynamicAttrs 进行 concat 操作，并执行 genProps 将其转换成对应的 generate 字符串
const attrs = el.attrs || el.dynamicAttrs
    ? genProps(
        (el.attrs || []).concat(el.dynamicAttrs || []).map(attr => ({
          // slot props are camelized
          name: camelize(attr.name),
          value: attr.value,
          dynamic: attr.dynamic
        }))
    	)
    : null
```

## 最后
