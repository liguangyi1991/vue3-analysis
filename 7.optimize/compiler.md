# Vue3 中模板编译优化

## 1.1 PatchFlags 优化

Diff 算法无法避免新旧虚拟 DOM 中无用的比较操作，通过 patchFlags 来标记动态内容，可以实现快速 diff 算法。

```vue
<div>
  <h1>Hello Jiang</h1>
  <span>{{name}}</span>
  <p><span>{{age}}</span></p>
</div>
```

> 此 template 经过模板编译会变成以下代码：

```ts
import {
  createElementVNode as _createElementVNode,
  toDisplayString as _toDisplayString,
  openBlock as _openBlock,
  createElementBlock as _createElementBlock,
} from "vue";

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (
    _openBlock(),
    _createElementBlock("div", null, [
      _createElementVNode("h1", null, "Hello Jiang"),
      _createElementVNode(
        "span",
        null,
        _toDisplayString(_ctx.name),
        1 /* TEXT */
      ),
    ])
  );
}
```

> 生成的虚拟 DOM 是：

```json
{
	type: "div",
  __v_isVNode: true,
  children:[
    {type: 'h1', props: null, key: null, …}
    {type: Symbol(), props: null, key: null, …}
  	{type: 'span', props: null, key: null, …}
  ],
	dynamicChildren:[{type: 'span', children: _ctx.name, patchFlag: 1}]
}
```

> 此时生成的虚拟节点多出一个 dynamicChildren 属性。这个就是 block 的作用，block 可以收集所有后代动态节点。这样后续更新时可以直接跳过静态节点，实现靶向更新

```ts
export const enum PatchFlags {
  TEXT = 1, // 动态文本节点
  CLASS = 1 << 1, // 动态class
  STYLE = 1 << 2, // 动态style
  PROPS = 1 << 3, // 除了class\style动态属性
  FULL_PROPS = 1 << 4, // 有key，需要完整diff
  HYDRATE_EVENTS = 1 << 5, // 挂载过事件的
  STABLE_FRAGMENT = 1 << 6, // 稳定序列，子节点顺序不会发生变化
  KEYED_FRAGMENT = 1 << 7, // 子节点有key的fragment
  UNKEYED_FRAGMENT = 1 << 8, // 子节点没有key的fragment
  NEED_PATCH = 1 << 9, // 进行非props比较, ref比较
  DYNAMIC_SLOTS = 1 << 10, // 动态插槽
  DEV_ROOT_FRAGMENT = 1 << 11,
  HOISTED = -1, // 表示静态节点，内容变化，不比较儿子
  BAIL = -2, // 表示diff算法应该结束
}
```

## 1.2 BlockTree

为什么我们还要提出 blockTree 的概念？ 只有 block 不就挺好的么？ 问题出在 block 在收集动态节点时是忽略虚拟 DOM 树层级的。

```vue
<div>
    <p v-if="flag">
        <span>{{a}}</span>
    </p>
    <div v-else>
        <span>{{a}}</span>
    </div>
</div>
```

> 这里我们知道默认根节点是一个 block 节点，如果要是按照之前的套路来搞，这时候切换 flag 的状态将无法从 p 标签切换到 div 标签。 **解决方案：就是将不稳定的结构也作为 block 来进行处理**

### 不稳定结构

所谓的不稳结构就是 DOM 树的结构可能会发生变化。不稳定结构有哪些呢？ （v-if/v-for/Fragment）

#### **v-if**

```vue
<div>
    <div v-if="flag">
        <span>{{a}}</span>
    </div>
    <div v-else>
        <p><span>{{a}}</span></p>
    </div>
</div>
```

```ts
return function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (
    _openBlock(),
    _createElementBlock("div", null, [
      _ctx.flag
        ? (_openBlock(),
          _createElementBlock("div", { key: 0 }, [
            _createElementVNode(
              "span",
              null,
              _toDisplayString(_ctx.a),
              1 /* TEXT */
            ),
          ]))
        : (_openBlock(),
          _createElementBlock("div", { key: 1 }, [
            _createElementVNode("p", null, [
              _createElementVNode(
                "span",
                null,
                _toDisplayString(_ctx.a),
                1 /* TEXT */
              ),
            ]),
          ])),
    ])
  );
};
```

```
Block(div)
	Blcok(div,{key:0}) -> Block(div,{key:1})
```

> 父节点除了会收集动态节点之外，也会收集子 block。 更新时因 key 值不同会进行删除重新创建

#### v-for

随着`v-for`变量的变化也会导致虚拟 DOM 树变得不稳定

```html
<div>
  <div v-for="item in fruits">{{item}}</div>
</div>
```

```ts
export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (
    _openBlock(true),
    _createElementBlock(
      _Fragment,
      null,
      _renderList(_ctx.fruits, (item) => {
        return (
          _openBlock(),
          _createElementBlock("div", null, _toDisplayString(item), 1 /* TEXT */)
        );
      }),
      256 /* UNKEYED_FRAGMENT */
    )
  );
}
```

> 可以试想一下，如果不增加这个 block，前后元素不一致是无法做到靶向更新的。因为 dynamicChildren 中还有可能有其他层级的元素。同时这里还生成了一个 Fragment，因为前后元素个数不一致，所以称之为**不稳定序列**。

### 稳定 Fragment

这里是可以靶向更新的, 因为稳定则有参照物

```html
<div>
  <div v-for="item in 3">{{item}}</div>
</div>
```

```ts
export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (
    _openBlock(),
    _createElementBlock("div", null, [
      (_openBlock(),
      _createElementBlock(
        _Fragment,
        null,
        _renderList(3, (item) => {
          return _createElementVNode(
            "div",
            null,
            _toDisplayString(item),
            1 /* TEXT */
          );
        }),
        64 /* STABLE_FRAGMENT */
      )),
    ])
  );
}
```

## 1.3 静态提升

```html
<div>
  <span>hello</span>
  <span a="1" b="2">{{name}}</span>
  <a><span>{{age}}</span></a>
</div>
```

我们把模板直接转化成 render 函数是这个酱紫的，那么问题就是每次调用`render`函数都要重新创建虚拟节点。

```ts
export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (
    _openBlock(),
    _createElementBlock("div", null, [
      _createElementVNode("span", null, "hello"),
      _createElementVNode(
        "span",
        {
          a: "1",
          b: "2",
        },
        _toDisplayString(_ctx.name),
        1 /* TEXT */
      ),
      _createElementVNode("a", null, [
        _createElementVNode(
          "span",
          null,
          _toDisplayString(_ctx.age),
          1 /* TEXT */
        ),
      ]),
    ])
  );
}
```

```ts
const _hoisted_1 = /*#__PURE__*/ _createElementVNode(
  "span",
  null,
  "hello",
  -1 /* HOISTED */
);
const _hoisted_2 = {
  a: "1",
  b: "2",
};

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (
    _openBlock(),
    _createElementBlock("div", null, [
      _hoisted_1,
      _createElementVNode(
        "span",
        _hoisted_2,
        _toDisplayString(_ctx.name),
        1 /* TEXT */
      ),
      _createElementVNode("a", null, [
        _createElementVNode(
          "span",
          null,
          _toDisplayString(_ctx.age),
          1 /* TEXT */
        ),
      ]),
    ])
  );
}
```

> 静态提升则是将静态的节点或者属性提升出去。**静态提升是以树为单位**。也就是说树中节点有动态的不会进行提升。

## 1.4 预字符串化

> 静态提升的节点都是静态的，我们可以将提升出来的节点字符串化。 当连续静态节点超过 20 个时，会将静态节点序列化为字符串。

```vue
<div>
  <span></span>
       ...
       ...
  <span></span>
</div>
```

```ts
const _hoisted_1 = /*#__PURE__*/ _createStaticVNode("<span>....</span>", 20);
```

## 1.5 缓存函数

```html
<div @click="e=>v=e.target.value"></div>
```

> 每次调用 render 的时都要创建新函数

```ts
export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (
    _openBlock(),
    _createElementBlock(
      "div",
      {
        onClick: (e) => (_ctx.v = e.target.value),
      },
      null,
      8 /* PROPS */,
      ["onClick"]
    )
  );
}
```

> 开启函数缓存后,函数会被缓存起来，后续可以直接使用

```ts
export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (
    _openBlock(),
    _createElementBlock("div", {
      onClick: _cache[0] || (_cache[0] = (e) => (_ctx.v = e.target.value)),
    })
  );
}
```
