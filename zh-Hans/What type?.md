# 什么“变量”？

你以为我要教你类型“变量”有哪几种？教你有哪些基本类型？那就又大错特错了！

在实际的类型运算中，我们比较常用的一个功能便是比较一个类型是否和另外一个类型逆变、协变、不变关系。通过用这些关系的是否满足，从而来触发一些我们需要的运算与校验。接下来我会介绍几种常用的方式：

在 TypeScript 中最基本的判断单元便是 `extends`，实际应用中他有许多的语义，而在这里仅需要一个将俩个类型进行判断的功能，所以我会避开那些造成歧义的用法，而单独介绍如何判断俩个类型的关系。

## 一点小小的 never 震撼

常见的我们会去判断一个类型是否为 Primitive 类型，这也是最常见的需求
```typescript
type IsString<X> = X extends string ? true : false

type T0 = IsString<'a'>
//   ^? type T0 = true
type T1 = IsString<string>
//   ^? type T1 = true
type T2 = IsString<boolean>
//   ^? type T2 = false
```

另外的，时常会有一些特殊情况我们需要去处理，比如这个类型 [`never`](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#the-never-type)。在 TypeScript 中，这个类型意味着一个不存在值的类型，从集合的角度来看，便是[空集](https://zh.wikipedia.org/wiki/%E7%A9%BA%E9%9B%86)、[单位元（幺元）](https://zh.wikipedia.org/zh-cn/%E5%96%AE%E4%BD%8D%E5%85%83)。
基本知识我们了解到了，那我们来思考一下一个问题，我如果通过 `extends` 运算去判断一个类型是不是 `never` 会发生什么呢？
```typescript
type T0 = never extends never ? true : false
//   ^? type T0 = true
// 很好，这里是符合预期的
type IsNever<T> = T extends never ? true : false

type T1 = IsNever<never>
//   ^? type T1 = never
// 但是当我们在泛型中使用他的时候却发现变成了 never
```

在这里我们需要知道一个关于 `extends` 的问题，实际上 ts 对于类型系统并没有引入过多的运算符，在这里便遇到了与之相关的一个问题，`extends` 他并不只有判断的语义。他同时还存在一个很特殊的情况，他会尝试拆解右值（在 `extends` 右侧的类型）如果为 union type 则按照规则遍历运行得到一个新的 union type。

> 具体关于拆解遍历行为，请参考[ts@2.8 - Distributive conditional types](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-8.html#distributive-conditional-types)

而我们在这里使用 `never` 便是一个特殊的 union type，它有没有任何一个成员，所以遍历它最后便只能得到一个“新的” never 回来，所以我们在这里看起来是无法直接处理这个问题。

但是，怎么可能呢？我们将思路打开，如果是个空我们就无法得到我们的预期，那么我们便可以构造一个类型，在外面包裹上一个新的类型来防止 TypeScript 的遍历 never 行为，可以是 `{}` 也可以是 `[]`（这里我们暂时不提函数的包裹方式）。简单写个例子：
```typescript
type IsNever<T> = [T] extends [never] ? true : false
type T0 = IsNever<never>
//   ^? type T0 = true
```
这样我们便能知道一个“变量”到底是不是一个 `never` 了。

> 思考一下：如何写一个 IsBoolean 类型运算?

## 奇妙的函数类型

在上面我们主要讨论的了作为非函数类型他们之间“是不是”的一个判断方式行为以及特殊情况，接下来我们便要去了解更深入一点的关于函数的判断规则。

函数作为 JavaScript 中的一等公民，在类型系统中也有举足轻重的地位，函数之间的关系判断使用的也是 `extends` 但是在这里存在一些问题我们需要注意的，在一般情况（不涉及泛型与函数重载）下满足[类型构造符→对输入类型是逆变的而对输出类型是协变的。](https://zh.wikipedia.org/wiki/%E5%8D%8F%E5%8F%98%E4%B8%8E%E9%80%86%E5%8F%98#:~:text=%E7%B1%BB%E5%9E%8B%E6%9E%84%E9%80%A0%E7%AC%A6%E2%86%92%E5%AF%B9%E8%BE%93%E5%85%A5%E7%B1%BB%E5%9E%8B%E6%98%AF%E9%80%86%E5%8F%98%E7%9A%84%E8%80%8C%E5%AF%B9%E8%BE%93%E5%87%BA%E7%B1%BB%E5%9E%8B%E6%98%AF%E5%8D%8F%E5%8F%98%E7%9A%84%E3%80%82)

简单的介绍完了，接下来我们利用一下「输入类型是逆变的而对输出类型是协变的」这一点做一些有趣的事情，在这里我假设一个需求：当俩个函数类型为何种关系的时候能对应的函数类型能被另一个函数类型所替换。
```typescript
interface A {
  a: string
}
interface B {
  a: string
  b: number
}
declare function fxo(): A
declare function fyo(): B

let a = fxo()
a = fyo()
// fyo extends fxo
// fxo 能被替换为 fyo
type A0 = typeof fyo extends typeof fxo ? true : false
//   ^? type A0 = true

let b = fyo()
b = fxo() // Property 'b' is missing in type 'A' but required in type 'B'.ts(2741)
// fxo not extends fyo
// fyo 不能被替换为 fxo
type B0 = typeof fxo extends typeof fyo ? true : false
//   ^? type B0 = false

// 总结的来说便是，当输出类型为协变时，即被替换目标(fxo)的输出类型**小于等于**替换目标(fyo)时才能使得前者能替换为后者

declare function fmo(a: A): void
declare function fno(a: B): void

fmo({ a: 'a' })
fno({ a: 'a' }) // Argument of type '{ a: string; }' is not assignable to parameter of type 'B'.
                // Property 'b' is missing in type '{ a: string; }' but required in type 'B'.ts(2345)
// fno not extends fmo
// fmo 不能被替换为 fno
type A1 = typeof fno extends typeof fmo ? true : false
//   ^? type A1 = false

// 这里的 as 是为了对齐
fno({ a: 'a', b: 1 } as B)
// 这里的 as 是为了防止 TypeScript 的 literal type 优化
fmo({ a: 'a', b: 1 } as B)
// fmo extends fno
// fno 能被替换为 fmo
type B1 = typeof fmo extends typeof fno ? true : false
//   ^? type B1 = true

// 总结的来说便是，当输入类型为逆变时，即被替换目标(fno)的输出类型**大于等于**替换目标(fmo)时才能使得前者能替换为后者
```

好了，到这里我们便对函数的逆变协变有了一定的了解，我们来整点特殊的类型来看看应该怎么做。
```typescript
interface A {
  a: string
}
interface B {
  a: '1'
}
declare function f0<G>(g: G): G extends A ? 1 : 2
declare function f1<G>(g: G): G extends B ? 1 : 2

let t0 = f0({ a: '' })
//  ^? let t0: 1
t0 = f1({ a: '' }) // Type '2' is not assignable to type '1'.ts(2322)
```
我们回忆下在上面的总结「当输出类型为协变时，即被替换目标的输出类型**小于或等于**替换目标时才能使得前者能替换为后者」。
```markdown
在这里假设我们需要将 f0 替换为 f1
    => 我们就需要让 f0 的输出类型**小于或等于** f1 的输出类型
    => f0 的输出类型由 G 与 A 共同决定，f1 也是如此
    => 俩个函数的 G 都会由用户的输入而决定，无法被保证
    => 那么这个时候如果需要能够替换，那么就需要 A 的类型**小于或等于** B 的类型
    => 而该输出类型还受 G 的影响，所以我们需要 G extends A 与 G extends B 都成立
    => 只有当 `A === B` 时，俩个函数的输出类型才能满足协变关系
    => 所以我们可以得知 `F<A> extends F<B>` 时，`A === B`
```

那么我们知道了这个有什么用呢？比如说 any 作为 top type ，使用很多的办法都没有办法判断一个类型是不是 any，但是我们通过这个就能判断你的同事是不是传了个 any 进来了！是不是很有用，那么接下来我们来写一段代码看看：
```typescript
type IsEqual<A, B> = (
    <G>() => G extends A ? 1 : 2
) extends (
    <G>() => G extends B ? 1 : 2
) ? true : false

type T0 = IsEqual<A, B>
//   ^? type T0 = false
type T1 = IsEqual<A, A>
//   ^? type T1 = true
type T2 = IsEqual<A, {
//   ^? type T2 = true
  a: string
}>
type T3 = IsEqual<{
//   ^? type T3 = false
  a: '1'
}, A>
type T4 = IsEqual<A, any>
//   ^? type T4 = false
```

> 拓展阅读：
> * [介于 TypeScript 与 JavaScript 之间](./What%20type%3F%20-%20TypeScript%20boundary.md)
