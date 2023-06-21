# Narrow Type

在[介于 TypeScript 与 JavaScript 之间](./What%20type%3F%20-%20TypeScript%20boundary.md)中我们提到了一些关于 TypeScript 的类型边界中的 `as const` 的作用，并了解到了它的一些特性，但是在这里我们可以根据一些 TS 中的特性来自己实现 `as const` 的功能。

对于我们常见的 primitive type 来说，当我们在函数的 Generic 位置进行声明，并在参数位置进行使用的时候，TypeScript 会自动推断出这些类型。
```typescript
declare function f0<G>(g: G): G

let t0 = f0(1)
//  ^? let t0: 1
let t1 = f0('1')
//  ^? let t1: "1"
let t2 = f0(true)
//  ^? let t2: true
```

但是我们在使用 `object literal` 的时候，TypeScript 会自动推断出这个类型的最宽泛的类型，这个时候我们就需要使用 `as const` 来进行类型的缩小。
```typescript
declare function f0<G>(g: G): G

let t0 = f0({ a: 'test' })
//  ^? let t0: { a: string; }
let t1 = f0({ a: 'test' } as const)
//  ^? let t1: { readonly a: "test"; }
```

虽然我们可以通过显式的声明来达到这个效果，但是这样的话就会显得很麻烦。那么我们有没有别的办法呢？
首先我们可以发现 TypeScript 其实在面对 `object literal` 的时候，会根据实际参数的类型信息尽可能将类型进行缩小，也就是说实际的类型并没有在传递过程中丢失成宽泛的类型。
```typescript
declare function foo(x: { foo: string, bar: 1 }): typeof x

const x = foo({ foo: 'foo', bar: 1 })
//    ^? const x: { foo: string; bar: 1; }
```

我们可以看到，`x` 的类型并没有丢失，那么我们尝试通过这种方式来使类型来进行缩小看看。
```typescript
declare function foo(x: {
  [K in string]: (typeof x)[K]
}): typeof x

const x = foo({ foo: 'foo', bar: 1 })
//    ^? const x: { [x: string]: unknown; }
// 在上面我们尝试去使用 `typeof x` 来对我们需要的类型进行推断
// 然而我们发现这样的话并没有起到我们想要的效果（全变成 unknown 了）
// 我们可以回忆一下我们最开始的函数为什么能出现自动推断 literal 的效果
// 然后我们便可以发现，我们缺少了一个 generic 类型，那么我们便可以尝试加上这个 generic 类型

declare function foo0<T>(x: {
  [K in keyof T]: T[K]
}): [typeof x, T]

const x0 = foo0({ foo: 'foo', bar: 1 })
//     ^? const x0: [{ foo: string; bar: number; }, { foo: string; bar: number; }]
// 我们可以看到，我们的类型已经被缩小了，但是我们的类型还是有一些问题
// 那么在哪里出现了问题呢？
// 在这里我猜测 TypeScript 在发现类型并不复杂的时候并不会去进行类型的缩小（可能是一个优化）
// 所以我认为 TYpeScript 在我们需要进行一个类型运算的时候，它会将一个具体的类型塞进去进行运算
// 那么我们便可以去尝试诱导一下它

declare function foo1<T>(x: {
  [K in keyof T]: T[K] extends (string) ? T[K] : never
}): [typeof x, T]

const x1 = foo1({ foo: 'foo', bar: 1 })
//    ^? const x1: [{ foo: "foo"; bar: never; }, { foo: "foo"; bar: number; }]
```
计划通！接下来我们通过工程化的手段来对他进行封装与优化，首先我们将类型扩充到所有的基础类型。

```typescript
type Primitive = string | number | boolean | bigint | symbol | undefined | null

declare function foo<T>(t: {
  [K in keyof T]: T[K] extends Primitive ? T[K] : never
}): [typeof t, T]

const x0 = foo({ foo: 'foo', bar: 1, baz: true, qux: 20n, qor: null, bor: undefined })
//    ^? const x0: [{ foo: "foo"; bar: 1; baz: true; qux: 20n; qor: null; bor: undefined; }, { foo: string; bar: number; baz: boolean; qux: bigint; qor: null; bor: undefined; }]
```

但是我们可以显然发现他不支持嵌套的定义方式，那么我们可以通过递归的方式来进行处理。
```typescript
type Primitive = string | number | boolean | bigint | symbol | undefined | null

type Narrow<T> = {
  [K in keyof T]: T[K] extends Primitive ? T[K] : Narrow<T[K]>
}

declare function foo<T>(t: Narrow<T>): T

const x0 = foo({ foo: 'foo', bar: { baz: true } })
//    ^? const x0: { foo: "foo"; bar: { baz: true; }; }
// 我们再来尝试一点特殊的类型
const x1 = foo([1, 2, true])
//    ^? const x1: (true | 1 | 2)[]
```
我们可以看到类型已经被缩小了，但是没有维持住 tuple 的类型，而是丢失了元素顺序的一个 array。那么我们有没有什么办法来维持住这个行为呢？在这里我们回忆一下我们上面所做的，实际上就是去诱导 TypeScript 的隐式推断，所以我们能不能同样的诱导出 tuple 的类型呢？
```typescript
// 我们可以先看一下这段代码
declare function t<T extends [] | number[]>(t: T): T

const t0 = t([1, 2])
//    ^? const t0: [number, number]
// 我们可以发现，当我们企图让类型可能（union）是一个 empty tuple 的时候
// TypeScript 会尝试将它理解（隐式推断）为一个 tuple 类型，从而保持了一个更小的类型
```
数组其实也算是一种特殊的 literal object，所以我们可以尝试去将它转换为一个 object，然后再去进行类型的缩小。
```typescript
type A0 = { 0: 1, 1: 2, length: 2 }
type A1 = [1, 2]
```
我们可以看到，数组其实就是一个带有数字索引的 literal object ，并且再附带了一些特殊的属性（这里的描述并不是特别的准确，但是可以按照这个思路理解这块的内容）。

接下来我们的事情便好办了，结合上面俩部分的内容，我们便可以很容易能得到一个这样的类型构造器。
```typescript
type Primitive = string | number | boolean | bigint | symbol | undefined | null

type Narrow<T> = T extends [] ? [] : {
  [K in keyof T]: T[K] extends Primitive ? T[K] : Narrow<T[K]>
}

declare function foo<T>(t: Narrow<T>): T

const x0 = foo({ foo: 'foo', bar: { baz: true } })
//    ^? const x0: { foo: 'foo'; bar: { baz: true; }; }
const x1 = foo([1, 2, true])
//    ^? const x1: [1, 2, true]
```

看到这里我们便对 Narrow 类型构造器的机制与实现原理有了一定的了解了，我们可以简单总结一下：
* 通过一个 generic 类型来诱导 TypeScript 的隐式推断
* 通过在 literal object 的 ValueType 上进行类型运算来诱导 TypeScript 的类型缩小
* 通过在 empty tuple 上进行类型运算来诱导 TypeScript 的类型缩小

> 拓展阅读：
> * [ts-toolbelt - Function Narrow](https://github.com/millsp/ts-toolbelt/blob/319e55123b9571d49f34eca3e5926e41ca73e0f3/sources/Function/Narrow.ts#L32)
> * [Suggestion: Const contexts for generic type inference](https://github.com/microsoft/TypeScript/issues/30680)
> * [Excess Property Checks](https://www.typescriptlang.org/docs/handbook/2/objects.html#excess-property-checks)
