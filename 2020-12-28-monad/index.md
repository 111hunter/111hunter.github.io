# IO 和 Monad


学习函数式编程总是给人一种学习哲学的错觉，Haskell 不仅定义了程序运算需要的类型，还给输入输出(IO)这样动作也定义了类型，并且把运行程序时的外部状态(RealWorld)当成函数的参数。

## IO

haskell 中 IO 的类型定义：

```haskell
newtype IO a
  = GHC.Types.IO (GHC.Prim.State# GHC.Prim.RealWorld
                  -> (# GHC.Prim.State# GHC.Prim.RealWorld, a #))
```

简化表示如下，

```haskell
type IO a = GHC.Types.IO (World -> (# World, a #))
```

程序运行时，输入值需要参与到程序运算中。用函数表示输入动作：

```haskell
World -> (# World, a #)
```

输出值不能再参与程序运算。用函数表示输出动作：

```haskell
World -> (# World, () #)
```

在 `GHCi` 中用 `:l` 加载下面的程序文件：

```haskell
{-# LANGUAGE UnboxedTuples         #-}
import GHC.Types

noExec (GHC.Types.IO a) = a

inStr = noExec (getLine)
outStr = noExec (putStrLn "hello world!")

exec = GHC.Types.IO
```

然后用 `:t inStr`，`:t outStr` 查看类型，确实是我们说的那样。

## Monad

考虑函数与单个数值或字符之间的运算：

- `(i -> o) -> i -> o`

```haskell
ghci> ((+4) 2) + ((*2) 3)
12
ghci> (\x ->(\y -> if x > y then x else y)) 8 6
8
```

调用函数就能计算结果，不过这样做计算对人来说太麻烦了。

简化计算的办法是将值放在上下文环境(e)中，例如 `[1,2,3]`，`(1,'a',3.14)` 等。怎么在不破坏值的上下文前提下，应用函数完成计算呢？

- `f -> e(i) -> e(o)`

换句话说，考虑 f 可能的情况，定义通用计算模型？

```haskell
f = (i -> o)
f = e(i -> o)
f = e(i -> e(o))
f = (e(i) -> o)
```

### Maybe

后面的讨论会用到 Maybe 类型：

```haskell
data Maybe a = Nothing | Just a
```

```haskell
ghci> :t Nothing
Nothing :: Maybe a
ghci> :t Just 2
Just 2 :: Num a => Maybe a
```

### Functor

处理函数暴露，值处于上下文的情况：

- `(i -> o) -> e(i) -> e(o)`

```haskell
ghci> (+3) (Just 2)
error
ghci> fmap (+3) (Just 2)
Just 5
```

fmap 怎么实现的？模式匹配：

```haskell
fmap f Nothing  = Nothing
fmap f (Just x) = Just (f x)
```

fmap 的动作是将上下文中的值取出，用函数处理后，重新放回上下文：

```haskell
class Functor f where
    fmap :: (a -> b) -> f a -> f b

instance Functor Maybe where
    fmap f Nothing  = Nothing
    fmap f (Just x) = Just (f x)
```

只要能被 fmap 处理，这种上下文就是 `Functor`，Maybe 是 `Functor`

### Applicative

处理函数与值处于同一种上下文的情况：

- `e(i -> o) -> e(i) -> e(o)`

```haskell
ghci> (Just (+3)) (Just 2)
error
ghci> (Just (+3)) <*> (Just 2)
Just 5
```

实现 `<*>` 同样用模式匹配：

```haskell
fmap f Nothing  = Nothing
fmap f (Just x) = Just (f x)

Nothing <*> _ = Nothing
(Just f) <*> x = fmap f x
```

`<*>` 的动作是直接从上下文中取出函数，之后就是 Functor 的情况了，用 fmap 处理

```haskell
class Functor f => Applicative f where
    pure :: a -> f a
    (<*>) :: f (a -> b) -> f a -> f b

instance Applicative Maybe where
    pure = Just
    Nothing <*> _ = Nothing
    (Just f) <*> x = fmap f x
```

只要能被 `<*>` 处理，这种上下文就是 `Applicative`，Maybe 是 `Applicative`

### Monad

处理值与函数的返回值处于同一种上下文的情况：

- `e(i) -> (i -> e(o)) -> e(o)`

这里交换了函数和值的顺序，数据流向是从左往右的。

```haskell
ghci> (Just (+3)) <*> (Just 2)
Just 5
ghci> (\x -> Just (x + 3)) (Just 2)
error
ghci> (Just 2) >>= (\x -> Just (x + 3))
Just 5
```

实现 `>>=` (发音为 bind) 同样用模式匹配：

```haskell
Nothing >>= f = Nothing
Just x  >>= f = f x
```

`>>=` 的动作是直接从上下文中取出值，应用到函数上

```haskell
class Applicative m => Monad m where
  (>>=) :: m a -> (a -> m b) -> m b
  (>>) :: m a -> m b -> m b
  return :: a -> m a

instance Monad Maybe where
    return x = Just x
    Nothing >>= _ = Nothing
    Just x >>= f = f x
```

能被 `>>=` 处理，并且实现 `Applicative` 的行为就是 `Monad`。每种 `Monad` 至少能处理三种函数：

```haskell
f = (i -> o)
f = e(i -> o)
f = e(i -> e(o))
```

我们可以利用 `>>=` 的性质实现链式操作：

```haskell
ghci> let add3 = (\x -> Just (x + 3))
ghci> (Just 2) >>= add3 >>= add3 >>= add3 >>= add3
Just 14
```

IO 类型也被 Haskell 定义为一种 Monad：

```haskell
ghci> getLine >>= readFile >>= putStrLn
```

Haskell 为我们提供了一些用于 Monad 的语法糖，称为 do 语法：

```haskell
foo = do
    filename <- getLine
    contents <- readFile filename
    putStrLn contents
```

在[这篇](https://111hunter.github.io/2020-11-26-%CE%BB-calculus/)文章中，我们用函数模拟了一阶逻辑，
而 Monad 是函数对结构化程序的模拟。

### 补充

处理值与函数的参数处于同一种上下文的情况：

- `(e(i) -> o) -> e(i) -> e(o)`

```haskell
class Func f where
    comp :: (f a -> b) -> f a -> f b

instance Func Maybe where
    comp f Nothing  = Nothing
    comp f x = Just (f x)
```

```haskell
ghci> comp (\x -> if x == (Just "pwd") then "true" else "false") (Just "pwd")
Just "true"
```

[这里](http://hackage.haskell.org/package/base-4.8.2.0/docs/Prelude.html#t:Maybe) 查阅 Maybe 的相关实现, 有时间再补充。

**推荐阅读**

- [Your easy guide to Monads, Applicatives, & Functors](https://medium.com/@lettier/your-easy-guide-to-monads-applicatives-functors-862048d61610)
- [关于 Monad 的学习笔记](https://juejin.cn/post/6844903438434697224)
- [Primitive Haskell](https://www.fpcomplete.com/haskell/tutorial/primitive-haskell/)
- [Haskell Tutorial](https://openhome.cc/Gossip/CodeData/HaskellTutorial/)
- [hackage.haskell.org](http://hackage.haskell.org/package/base-4.8.2.0/docs/Prelude.html#t:Maybe)

