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
declare function f0<G>(g: G): G extends A ? 1 : 2
declare function f1<G>(g: G): G extends B ? 1 : 2

let t0 = f0({ a: '' })
//  ^? let t0: 1
t0 = f1({ a: '' }) // Type '2' is not assignable to type '1'.ts(2322)
```
我们回忆下在上面的总结「当输出类型为协变时，即被替换目标的输出类型**小于或等于**替换目标时才能使得前者能替换为后者」。那么在这里假设我们需要将 f0 替换为 f1，那么我们就需要让 f0 的输出类型**小于或等于** f1 的输出类型。
在什么情况下 f0 的输出类型会**小于或等于** f1 的输出类型呢？只有一种情况下 f0 会小于 f1，那就是 `A === B` 的时候，所以我们反向思考一下，如果俩个形如上式的函数能够满足替换关系，那么 `A === B`。
再转化一下角度「能够满足替换关系」=>「F\<A> extends F\<B>」。

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
type T3 = IsEqual<A, any>
//   ^? type T3 = false
```

## 介于 TypeScript 与 JavaScript 之间

在这里我将 `as`, `is`, `satisfies` 和 `in` 也归类于此节，实际上来说它们也算是确保一个什么类型的行为，只是位于 TypeScript 与 JavaScript 的边界中，将变量的类型显式的与目标的类型按照规则进行匹配。

对于 `as` 这种常见的操作符来说，一般会在几个位置中出现。
* 作为 JavaScript 与 TypeScript 的交界处，将某一个 JavaScript 中的值作为一个类型去与 TypeScript 中的类型进行断言
* 作为 TypeScript 中的类型断言，将某一个 TypeScript 中的类型作为一个类型去与 TypeScript 中的类型进行断言

在这里我采用了[《TypeScript Deep Dive》](https://basarat.gitbook.io/typescript/type-system/type-assertion#type-assertion-vs.-casting)中的说法，倾向于 `as` 是作为一个「Assertion（断言）」，而不是一个「Casting（转化）」。

```typescript
interface A {
    a: string
}
// 作为 JavaScript 与 TypeScript 的交界处
const a0 = { a: '1' } as A
const a1 = { b: 1 } as A // TS2352: Conversion of type '{ b: number; }' to type 'A' may be a mistake
                         // because neither type sufficiently overlaps with the other.
                         // If this was intentional, convert the expression to 'unknown' first.
                         // Property 'a' is missing in type '{ b: number; }' but required in type 'A'.
// 在上面我们断言了 `{ b: 1 }` 是一个 A 类型，但是实际上他完全和 A 没有关系
// 于是乎 TypeScript 便也给我们抛出来了一个错误
const a2 = {  } as A
const a3 = { a: '1', b: 1 } as A

type A0 = { a: string } extends A ? true : false
//   ^? type A0 = true
type A1 = { b: number } extends A ? true : false
//   ^? type A1 = false
// 在上面的 `{}` 实际上被**隐式推断**为了 `any`，所以这里使用 `{}` 进行测试
// 这里我们暂且记住**隐式推断**，在下文中我会去解释他
type A2 = [any] extends [A] ? true : false
//   ^? type A2 = true
type A3 = { a: string, b: number } extends A ? true : false
//   ^? type A3 = true
```

上面是我们对 `as` 的一个简单的了解，他的行为很类似于使用 `extends` 对俩个类型进行推断。但是我们也从中发现了一些令人感到十分疑惑不解的部分，为什么我们使用 `{}` 会被**隐式推断**为 `any` 呢？他会不会在其他的地方被触发呢？
在我们对 `as` 进行深入探索之前我们需要回忆一下在 TypeScript Handbook 中对 `as` 行为的这句定义「[转化向一个更具体（more specific）或者更不具体（less specific）版本的类型](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#:~:text=convert%20to%20a%20more%20specific%20or%20less%20specific%20version%20of%20a%20type)」。
```typescript
type A = 1

// 在这里当我们尝试将 1 断言为 A 时，表面上看一切都是没问题的
const a0 = 1 as A
// 但是实际上我们在触发断言操作的前方出现了一个 literal primitive type
// 实际上他会触发一个隐式推断，所以实际的执行效果是 `2 as number as A`
const a1 = 2 as A
const a2 = Number() as A
```

对于上面的几个来说类型的运算看起来都能通过 `V as number as A` 进行解释，但是在下面可以发现了一个问题，哪怕我们强制 `as` 了一个 `const` 的类型，又或者是依赖 TypeScript 自己的 literal type 转化出来的 const 类型也能通过类型校验。
```typescript
const t0 = 2
const t1 = t0 as A
const t2 = 2 as const as A
const t3 = 2 as 2 as 1
```
`2 as 1` 这是一个被 TypeScript 被允许的行为，但是只被允许在 primitive type 中，当俩个 literal primitive type 的 `supertype` 一致时，则会被允许被断言所操作。
```typescript
const x0 = 1 as true // TS2352: Conversion of type 'number' to type 'true' may be a mistake because neither type sufficiently overlaps with the other.
                     // If this was intentional, convert the expression to 'unknown' first.
const x1 = true as boolean
// less specific
const x2 = Boolean() as true
// more specific

type B = { b: 1 }

const b0 = { b: 1 } as B
const b1 = { b: 2 } as B // TS2352: Conversion of type '{ b: 2; }' to type 'B' may be a mistake because neither type sufficiently overlaps with the other.
                         // If this was intentional, convert the expression to 'unknown' first.
                         //   Types of property 'b' are incompatible.
                         //     Type '2' is not comparable to type '1'.
const b2 = { b: 2 as number } as B
const b3 = { b: 2 }
const b4 = b3 as B
const b5 = { b: 2 } as const
const b6 = b5 as B // TS2352: Conversion of type '{ readonly b: 2; }' to type 'B' may be a mistake because neither type sufficiently overlaps with the other.
                   // If this was intentional, convert the expression to 'unknown' first.
                   //   Types of property 'b' are incompatible.
                   //     Type '2' is not comparable to type '1'.
type C = [1]

const c0 = [1] as C
const c1 = [2] as C // TS2352: Conversion of type '[2]' to type 'C' may be a mistake because neither type sufficiently overlaps with the other.
                    // If this was intentional, convert the expression to 'unknown' first.
                    //   Type '2' is not comparable to type '1'.
const c2 = [2 as number] as C
const c3 = [2]
const c4 = c3 as C
const c5 = [2] as const
const c6 = c5 as C // TS2352: Conversion of type 'readonly [2]' to type 'C' may be a mistake because neither type sufficiently overlaps with the other.
                   // If this was intentional, convert the expression to 'unknown' first.
                   //   The type 'readonly [2]' is 'readonly' and cannot be assigned to the mutable type 'C'.
```
在上面关于 B 和 C 类型的测试中，我们可以看到一旦 literal primitive type 进入到 `{}`、`[]` 中便会丢失该特性，那么我们再来尝试将其拆出来进行验证。

```typescript
const b00 = 2 as B['b']
const c00 = 2 as C[number]
const c01 = 2 as C[0]
```

好了，基于上述我们对 `as` 的各种行为观察，我们可以得到一些结果。
* 当 `as` 操作的左侧类型出现了 literal primitive type 时 TypeScript 会将其隐式推断为其的 `supertype`，如：`1` -> `number`、`true` -> `boolean`，在这个基础上进行是否为 supertype 或 subtype 的判断。
* 当 `as` 操作的左侧为非 primitive 的 literal value 时，将其隐式推断为对应的 const 类型，再进行是否为 supertype 或 subtype。
* 当 `as` 操作的左侧为 `{}` 类型时，隐式转化为 any 进行是否为 supertype 或 subtype 的判断。

我们再以一图解释一下 `as any as T` 是如何工作的：

<img
src="../imgs/as_any_as_T.png"
alt="as any as T"
/>

在 top type 与 bottom type 之间的任意类型，都可以通过 more specific 和 less specific 来到 top 或者 bottom 中，再从其前往任意一个类型，从而解决了不同类型之间的隔断。

所以其实除了 `as any as T` 还有 `as unknown`、`as never` 均可用于解决该问题。

> 拓展阅读：
> * [Type hierarchy tre](https://www.zhenghao.io/posts/type-hierarchy-tree)
> * [Unknown vs any](https://stackoverflow.com/a/67314534/15375383)
>
> 相比较使用 `any` 作为中间类型，使用 `unknown` 会更好，因为前者是编译器开的洞，至于为什么不是 `never` 呢，主要还是语义看起来有点奇怪。