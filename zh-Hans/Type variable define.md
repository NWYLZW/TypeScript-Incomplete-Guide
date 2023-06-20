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
//   ^? type Q = string
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
在这里的 type 或者 interface 中的泛型声明，可以在后续的类型运算中使用。但是如果未给定默认值，那么在调用的时候必须传入，并不是很方便。所以这里如果需要让 TypeScript 对其进行计算，那么最好给定一个与 extends 约束相同的默认值（或者 extends 定义一个相对宽泛的类型，在默认值的位置定义更确切需要的类型）。

通过上面的描述，我们可以感觉这种定义的方式似乎并不是很方便，那么我们有没有更符合的场景呢？
```typescript
// 在 TypeScript 的函数中，通过编译器对参数的类型推断，我们可以不去定义参数类型，而是由编译器推断出来
declare function foo<
    T extends readonly any[],
    N extends T[0]
>(t: T): N

const a = foo([1, 2, 3] as const)
//    ^? const a: 1
```
这样看起来就方便了许多了，但是对于这个同时也存在一些已知的问题。当你想传入一个类型，但是不想传入一个类型的时候，则会导致你必须像使用 type 一样去设置一个默认值。

```typescript
const b = foo<(1 | 2)[]>([1, 2, 3])
//            ^^^^^^^^^ TS2558: Expected 2 type arguments, but got 1.
```

这个问题在 TypeScript 中是暂时无法解决的，因为它是一个[已知的问题](https://github.com/microsoft/TypeScript/issues/20122)，并被准备[在 5.2 版本](https://github.com/microsoft/TypeScript/issues/54298#:~:text=Investigate%20Type%20Argument%20Placeholders)中得到[支持](https://github.com/microsoft/TypeScript/pull/26349)。

> 我们可以利用这个做一些常用类型的初始化，但是请注意，尽量减少在这里的类型定义。
>
> 在这里过多过大的类型可能会导致类型不方便被观测检察。

## 类型中的 PropertyKey

其次便是一个相对来说十分常用的类型定义位置了，在 `{}` 的 PropertyKey 的位置中我们能够使用 `in` 运算符定义一个 `string | number | symbol` 类型的变量，并在 PropertyValue 的位置中使用它。

```typescript
// 在这里我们可以将 K 上的 number 形式的字符串通过 infer 转换为对应的 number 类型
//   _? type T0 = { 1: 1 }
type T0 = {
    [K in '1']: K extends `${infer T extends number}` ? T : never
}
```

我们还可以使用 [ts@4.1 - Key Remapping via `as`](https://www.typescriptlang.org/docs/handbook/2/mapped-types.html#key-remapping-via-as) 从而修改 PropertyKey 的名称，而不改变 PropertyValue 中对 `in` 操作符得到的类型进行变动。
```typescript
//   _? type T0 = { 2: 1 }
type T1 = {
  [K in '1' as '2']: K extends `${infer T extends number}` ? T : never
}
```

<!-- TODO 讲一讲怎么动态的使用 class infer 出一个嵌套类型 -->
<!-- 有时候我们可能会遇到一种特殊的情况，定义一个嵌套的类型 -->
