+++
title = "Programming Language Roam: Isomorphism Lens"
date = 2022-05-04T17:00:00+08:00
tags = ["fsharp"]
draft = false
+++

isomorphism lens 是 Twan van Laarhoven 继 「[Van Laarhoven Lens]({{< relref "van_laarhoven_lens" >}})」后发明的一种新的 lens 实现。

这种实现更加简洁、更符合直觉，所以易于理解，在部分情况下比前者更加高效，但大多数场景下可能没有前者使用方便。


## 定义 {#定义}

isomorphism lens 的实现思想十分富有创意，他将 lens 抽象定义为:

> A lens from type a to b is a bijection between a and a pair of b and some residual r.

翻译过来就是: **一个 a 到 b 的 lens 是 a 到 (b, r) 的双射，其中 r 为 a 中除去 b 后的剩余部分**

简单解释下:

1.  设类型 a 为一个集合，集合内各个元素为其字段
2.  b 为 a 中的一个字段
3.  则 r = a - {b}
4.  lens 为一个双射/同构, 由两个互逆函数 f 和 b 组成

    f : a -&gt; (b, r)

    b : (b, r) -&gt; a

因此这种 lens 被称作同构 lens

作者为了便于理解，特意制作了一副示意图:
![](https://www.twanvl.nl/image/lens/isolens1.png)

结合图片，就可以十分简单明了的重新定义访问和更新了

访问
: 将元素 b 从集合 a 中拿出来，得到 b 和 余集 r

更新
: 将更新后的元素 b 加入到余集 r 中，就得到更新后的新集合 a


### 在代码中的定义 {#在代码中的定义}

```fsharp
type ISO<'a,'b> = ('a -> 'b) * ('b -> 'a)
type Lens<'a, 'b, 'r> = Lens of ISO<'a, 'b * 'r>
```


## get/set 的实现 {#get-set-的实现}


### get {#get}

```fsharp
let get (Lens iso) = fst iso >> fst
```

get 的实现比较简单，取出 ISO 结构中的 a -&gt; (b, r) 然后 和 fst 函数复合就可以了

更加详细易于理解的代码如下:

```fsharp
let get (Lens iso) =
    let f = fst iso      // 取出元组中的 a -> (b, r) 部分，绑定到变量 f 上
    fun a ->             // 返回 getter 函数, getter : a -> b
        let pair = f a   // 将 a 应用到函数 f 上
        fst pair         // 从 (b, r) 中取出 b
```


### set 的实现 {#set-的实现}

```fsharp
let first f (a, b) =
    (f a, b)

let private mkConst a b = a

let modify (Lens iso) f = fst iso >> first f >> snd iso
let put l a = modify l (mkConst a)
```

set 的实现是先取出 (b, r)，然后将更新函数 f 应用到 b 上，最后将得到的结果应用到 ISO 的 (b, r) -&gt; a 上

同上，详细的代码如下:

```fsharp
let moify (Lens iso) f =
    let f1 = fst iso  // 取出 a -> (b, r) 部分
    let f2 =          // 复合更新函数 f，将结果绑定到变量 f2 上, f2 : a -> (b, r)
        fun a ->
            let (b, r) = f1 a
            (f b, r)
    let w = snd iso   // 取出更新函数 w : (b, r) -> a
    fun a ->          // 返回 setter : b -> a
        let res = f2 a  // 将 a 作用到 f2 上，实现对 b 的更新
        w res           // 将 res : (b, r) 应用到 w 上，得到更新后的 a
```


## 示例 {#示例}

示例和上次一样，演示对深层嵌套结构的更新，在这种情况下 isomorphism lens 在自由组合上，不如 van laarhoven lens 方便

```fsharp
open Lens

// 定义需要更新的数据类型
type Skill = {
    Damage : int
    }

type Monster = {
    Name : string
    Skill : Skill
    }

// 定义 Monster -> Skill 的 a -> (b, r) 部分
let fwm m = (m.Skill, fun s -> {m with Skill = s})

// 定义 Skill -> Damage 的 a -> (b, r) 部分
let fws s = (s.Damage, fun d -> {s with Damage = d})

// 在 record 的更新这种情况下, (b, r) -> a 是一个通用的函数
let bw (l, r) = r l

// lens 的复合
// 可以看见，在处理 record 这种情况下
// isomorphism lens 的复合 比 van laarhoven lens 麻烦些
// 但是比朴素的 (getter, setter) 元组又要简便许多
// 在处理 tuple 或者数据本身就存在同构时, isomorphism lens 的复合则会十分简单
let com f1 f2 s =
    let (s2, r1) = f1 s
    let (s3, r2) = f2 s2
    (s3, r2 >> r1)

// 得到 Monster -> Damage 的 lens
let lens = Lens (com fwm fws, bw)

let monster = {Name = "A"; Skill = {Damage = 12}}

// 访问
get lens monster |> printfn "Damage is:%O"

// 更新
put lens 33 monster |> printfn "New Monster is:%O"

```

结果:

```bash
Damage is:12
New Monster is:{ Name = "A"
                 Skill = { Damage = 33 } }
```