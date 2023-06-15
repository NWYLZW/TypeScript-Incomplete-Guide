## “变量”声明

你以为我要教你 type 和 interface 是什么？那就大错特错了！来吧，写点不一样的东西。

## `infer` 的妙用

```typescript
// 当一个类型无法通过简单的方式访问到时, 我们可以使用 infer 关键字来实现对其的快速访问
type Type0<T> = T extends (
    (...args: infer U) => any
) ? U : never

type P = Type0<(a: number, b: string) => void>
//   ^? type P = [a: number, b: string]

interface Foo<T> {
    a: { b: { c: T } }
}
type Type1<T> = T extends (
    Foo<infer U>
) ? U : never

type Q = Type1<{
//   ^? type Q = number
    a: { b: { c: string } }
}>

// 同时我们也可以将其运用于数组之中（虽然它可以通过简单的方式访问到）
type Type2<T> = T extends [infer T0, ...any[]]
    ? T0
    : never

type R = Type2<[1, 2, 3]>
//   ^? type R = 1
```

同时在较高版本 infer 提供了可以约束推断类型的行为（infer 默认推断的类型不可用）。

* [ts@4.7 - 为 `infer` 提供 `extends` 进行类型约束](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#extends-constraints-on-infer-type-variables)
* [ts@4.8 - 增强模版字符串中的 `infer` 类型约束](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-8.html#improved-inference-for-infer-types-in-template-string-types)

第一个增强没啥太大的用，主要用于简化代码，以及更好的提供对 infer 的类型支持，重点是第二个可以用来整点点小活。

```typescript
type T0 = "1" extends `${infer T extends number}` ? T : never
//   ^? type T0 = 1
type T1 = "true" extends `${infer T extends boolean}`
//   ^? type T1 = "This is true"
    ? ([T] extends [true] ? 'This is true' : 'This is false')
    : never
```

> 其实它很像一个概念「[Pattern Matching（模式匹配）](https://zh.wikipedia.org/wiki/%E6%A8%A1%E5%BC%8F%E5%8C%B9%E9%85%8D)」，或者说就是。
>
> 当然在 ECMAScript 中也有一个相关的[提案](https://github.com/tc39/proposal-pattern-matching)。
>
> 拓展阅读：
> * [Type inference in conditional types](https://github.com/Microsoft/TypeScript/pull/21496)

## 类型中的泛型

除了 infer 的形式声明一个“变量”，还有常见于在「Generic（泛型）」中的“变量”声明。

```typescript
type T0<
    A,
    // 直接定义间接类型变量（这里并没有描述为「Constraint（约束）」，因为主要想阐述的是在这里的作用，你能这么用）
    B extends A,
    // 同时支持定义 type 一样的运算
    C extends A extends [infer A0] ? A0 : never,
    // 下面俩种不会保证类型一定正确，所以在后续的类型运算过程中可能会有问题
    D = A,
    E = A extends [infer A0] ? A0 : never,
> = {}
```

> 我们可以利用这个做一些常用类型的初始化，但是请注意，尽量减少在这里的类型定义。
>
> 在这里过多过大的类型可能会导致类型不方便被观测检察。

其次便是一个相对来说十分常用的类型定义位置了，在 Property 的位置中我们能够使用 `in` 运算符定义一个 `string | number | symbol` 类型的变量，并在 ValueType 的位置中使用它。

```typescript
type T0 = {
    [K in string | number | symbol]: K
}
```

通过这个我们可以实现一些小需求
* 修改前面的 Property 的类型，但是不去修改 Value 引用位置 K 的类型
* 将某些 Property 通过一定的规则删除掉
  详情可以参考 [ts@4.1 - Key Remapping via `as`](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html#key-remapping-via-as) 。

<!-- TODO 讲一讲怎么动态的使用 class infer 出一个嵌套类型 -->
<!-- 有时候我们可能会遇到一种特殊的情况，定义一个嵌套的类型 -->
