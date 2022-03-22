---
title: 泛型在Go中的应用
date: 2022-03-22 20:08
category:
- Go
tags:
- Generics
- Error Handling
---

Go1.18正式发布，从此Go算是正式有了泛型，本文来探讨Go泛型的几种应用场景。

## 类型安全的函数式接口

在Go语言里，函数是一等公民，这意味着你可以进行一定程度的函数式编程，但是因为Go拉胯的类型系统以及Lambda表达式的缺位，Go做起函数式编程来总是不太舒服，现在有了参数化类型以后`map`，`fold`，`filter`等函数总算是能舒服些了。接下来看看如何利用泛型实现出类型安全的`Map`， `Filter`， `Foldl`，`Foldr`。

### 实现
从这几个函数的类型签名入手，然后按语义进行翻译就是了：

> 这里类型签名采用我偏爱的Haskell的类型签名（同时混合使用了Go的类型），个人认为这种记法的好处在于分离了identifier和type signature，便于一眼看出函数的类型以及进行类型的运算；这种记法里的`->`可以理解为一个映射
{: .prompt-info }

- `Map :: (A -> B) -> []A -> []B`
    - 对应的Go实现:

```go
func Map[A, B any](f func(A) B, l []A) []B {
        res := make([]B, len(l))
        for i, e := range l {
                res[i] = f(e)
        }
        return res
}
```

- `Filter :: (A -> bool) -> []A -> []A`
    - 对应的Go实现:

```go
func Filter[A any](f func(A) bool, l []A) []A {
        res := make([]A, 0, len(l))
        for _, e := range l {
            if f(e) {
                res = append(res, e)
            }
        }
        return res
}
```

- `Foldl :: (B -> A -> B) -> B -> []A -> B`
    - 对应的Go实现

```go
func Foldl[A, B any](f func(B, A) B, b B, l []A) B {
    res := b
    for _, e := range l {
        res = f(res, e)
    }
    return res
}
```


- `Foldr :: (A -> B -> B) -> B -> []A -> B`
    - 对应的Go实现

```go
func Foldr[A, B any](f func(A, B) B, b B, l []A) B {
	res := b
	for i := len(l) - 1; i >= 0; i-- {
		res = f(l[i], res)
	}
	return res
}
```

- `ForEach :: (A -> Unit) -> []A -> Unit`
    - 这里Unit用来表示无返回
    - 对应的Go实现

```go
func ForEach[A any](f func(A), l []A) {
	for _, e := range l {
		f(e)
	}
}
```

借助上面这些基本函数可以实现出:`concat`，`concatMap`

```go
func Concat[A any](l [][]A) []A {
	return Foldl(func(a []A, b []A) []A { return append(a, b...) }, []A{}, l)
}
func ConcatMap[A, B any](f func(A) []B, l []A) []B {
        return Concat(Map(f, l))
}
```

### 应用

比较常见的简单应用就不说了，例如Web开发中可以利用Map方便地进行Vo,Dto,Entity之间的转换。这里给几个比较fancy的例子：

- 全排列函数(这里假设列表里没有重复元素了，为了`Remove`实现方便一点)

```go
func Remove(x int, l []int) []int {
	return Filter(func(y int) bool { return x != y }, l)
}

func Permutations(s []int) [][]int {
// 如果s是个空集合，那么其全排列为一个包含空集合的集合，即{Ф}
	if len(s) == 0 {
		return [][]int{[]int{}}
	}
/* 这个ConcatMap是精髓，先看ConcatMap的对象，就是s，所以对每个元素都会执行第一个函数参数
 * 第一个函数的意思是：接受元素x，然后把集合s中的x去除以后对剩下的部分进行全排列，
 * 然后向得到的全排列集合里的每个切片的最后上插一个x
 * 如此一来就得到了以x为尾其余元素的一个全排列为头的集合的集合；
 * 对s中的每个元素都应用这个函数然后再拼接不就得到了所有元素的全排列
 * 当然这个实现是满出翔并且有bug的
 */
	return ConcatMap(func(x int) [][]int {
		return Map(func(p []int) []int { return append(p, x) }, Permutations(Remove(x, s)))
	}, s)
}
```

- 求解八皇后

```go
func abs(x int) int {
	if x < 0 {
		return -x
	}
	return x
}

func safe(k int, positions []int) bool {
	for i, p := range positions {
		if p == k || abs(p-k) == i+1 {
			return false
		}
	}
	return true
}
func sequence(a, b int) []int {
	if a > b {
		return []int{}
	}
	result := make([]int, b-a+1)
	for i := range result {
		result[i] = a + i
	}
	return result
}

func positions(k, n int) [][]int {
//positions: 尝试在大小为n的棋盘的第k行放棋子
	if k == 0 {
		return [][]int{[]int{}}
	}
	conf := positions(k-1, n)

        /*
         * 稍微解释一下这坨怪物，其实懂了以后就明白这是个非常简洁的回溯
         * 首先看ConcatMap的对象，sequence(1, n)，其实就是对于从1..n这n个位置，循环使用第一个函数
         * 可以展开成对应的for循环
         * 然后Map的效果是在对头上插上p
         * Filter过滤出那些在p上放置棋子仍然安全的情况(这样就会边搜索边剪枝了)
         */
	return ConcatMap(func(p int) [][]int {
		return Map(func(ps []int) []int { return append([]int{p}, ps...) },
			Filter(func(ps []int) bool { return safe(p, ps) }, conf))
	}, sequence(1, n))
}

func Queen(boardSize int) [][]int {
	return positions(boardSize, boardSize)
}
```

与Go泛型相关内容，计划下一篇更新迭代器模式
