# Narrow Type

在[]()中我们提到了一些关于 TypeScript 的类型边界中的 `as const` 的作用，并了解到了它的一些特性，但是在这里我们可以根据一些 TS 中的特性来自己实现 `as const` 的功能。

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
计划通！
接下来我们通过工程化的手段来对他进行封装与优化。
