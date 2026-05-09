---
title: "Algorithm W 分步讲解"
description: "使用 Haskell 从零实现经典 Hindley-Milner 类型推断 Algorithm W 的中文讲解。"
aliases:
  - Algorithm W Step by Step
tags:
  - type-system
  - algorithm-w
  - hindley-milner
  - haskell
  - type-inference
  - translation
lang: "zh-CN"
enableToc: true
author: "Martin Grabmüller"
source: "https://raw.githubusercontent.com/wh5a/Algorithm-W-Step-By-Step/master/AlgorithmW.lhs"
---

> [!info] 来源
> 本文整理自 Martin Grabmüller 的 Algorithm W 分步讲解版本，原始来源见 [AlgorithmW.lhs](https://raw.githubusercontent.com/wh5a/Algorithm-W-Step-By-Step/master/AlgorithmW.lhs)。

## 引言

类型推断是一件棘手的事。学习它的基础也更加困难，因为大多数资料讨论的都是很高级的话题，例如 rank-N 多态、预测性/非预测性类型系统、全称类型和存在类型，等等。

我最适合的学习方式是实际把问题的解法实现出来，因此我决定写一篇类型推断的基础教程，实现一种最基本、但仍然具有实际用途的类型推断算法。ML、Haskell 等语言的类型检查器都可以以这种算法为基础。

本文研究的类型推断算法，是 Milner 提出的经典 Algorithm W。关于这个算法及其变体和扩展，一份很易读的介绍见 Heeren 等人的论文。本文的若干方面也受到 Jones 的论文启发。[^1] [^2] [^3]

## Algorithm W

先从必要的导入开始。

为了表示环境（文献中也称为上下文）和替换，我们导入 `Data.Map` 模块。类型变量等集合将用 `Data.Set` 模块中的集合表示。

```haskell
import qualified Data.Map as Map
import qualified Data.Set as Set
```

由于还会使用多种 monad transformer，我们也从 monad 模板库中导入几个模块。

```haskell
import Control.Monad.Error
import Control.Monad.Reader
import Control.Monad.State
```

`Text.PrettyPrint` 模块提供了用于漂亮格式化和缩进输出的数据类型与函数。

```haskell
import qualified Text.PrettyPrint as PP
```

### 预备定义

我们先定义*表达式*（类型为 `Exp`）、*类型*（`Type`）和*类型方案*（`Scheme`）的抽象语法。

类型方案 $\forall a_1, ..., a_n. t$ 是一种类型，其中若干多态类型变量被全称量词绑定。

```haskell
data Exp
  = EVar String
  | ELit Lit
  | EApp Exp Exp
  | EAbs String Exp
  | ELet String Exp Exp
  deriving (Eq, Ord)

data Lit
  = LInt Integer
  | LBool Bool
  deriving (Eq, Ord)

data Type
  = TVar String
  | TInt
  | TBool
  | TFun Type Type
  deriving (Eq, Ord)

data Scheme = Scheme [String] Type
```

我们需要判断一个类型中的自由类型变量。函数 `ftv` 实现了这个操作；这里把它放在类型类 `Types` 中，因为稍后定义类型环境时也会需要这个操作。另一个有用的操作，是对类型、类型方案等结构应用替换。替换只替换自由类型变量，因此类型方案中被量化的类型变量不会受到替换影响。

```haskell
class Types a where
  ftv   :: a -> Set.Set String
  apply :: Subst -> a -> a
```

```haskell
instance Types Type where
  ftv (TVar n)     = Set.singleton n
  ftv TInt         = Set.empty
  ftv TBool        = Set.empty
  ftv (TFun t1 t2) = ftv t1 `Set.union` ftv t2

  apply s (TVar n) = case Map.lookup n s of
                       Nothing -> TVar n
                       Just t  -> t
  apply s (TFun t1 t2) = TFun (apply s t1) (apply s t2)
  apply s t            = t
```

```haskell
instance Types Scheme where
  ftv (Scheme vars t)     = (ftv t) `Set.difference` (Set.fromList vars)
  apply s (Scheme vars t) = Scheme vars (apply (foldr Map.delete s vars) t)
```

有时把 `Types` 中的方法扩展到列表也会很方便。

```haskell
instance Types a => Types [a] where
  apply s = map (apply s)
  ftv l   = foldr Set.union Set.empty (map ftv l)
```

现在定义替换。替换是从类型变量到类型的有限映射。

```haskell
type Subst = Map.Map String Type

nullSubst :: Subst
nullSubst = Map.empty

composeSubst :: Subst -> Subst -> Subst
composeSubst s1 s2 = (Map.map (apply s1) s2) `Map.union` s1
```

类型环境在正文中称为 $\Gamma$，它把项变量映射到相应的类型方案。

```haskell
newtype TypeEnv = TypeEnv (Map.Map String Scheme)
```

接下来定义几个作用于类型环境的函数。操作 $\Gamma \backslash x$ 会从 $\Gamma$ 中移除 $x$ 的绑定，在代码中称为 `remove`。

```haskell
remove :: TypeEnv -> String -> TypeEnv
remove (TypeEnv env) var = TypeEnv (Map.delete var env)

instance Types TypeEnv where
  ftv (TypeEnv env)     = ftv (Map.elems env)
  apply s (TypeEnv env) = TypeEnv (Map.map (apply s) env)
```

`generalize` 函数会把一个类型中所有“不在给定类型环境中自由出现、但在该类型中自由出现”的类型变量抽象出来。

```haskell
generalize :: TypeEnv -> Type -> Scheme
generalize env t = Scheme vars t
  where vars = Set.toList ((ftv t) `Set.difference` (ftv env))
```

例如类型方案实例化等操作，需要为新引入的类型变量生成新鲜名称。这里用一个合适的 monad 来实现，它负责生成新鲜名称。它也能够传递动态作用域环境、处理错误并执行 I/O，但本文不会深入这些细节。

```haskell
data TIEnv = TIEnv {}

data TIState = TIState { tiSupply :: Int }

type TI a = ErrorT String (ReaderT TIEnv (StateT TIState IO)) a

runTI :: TI a -> IO (Either String a, TIState)
runTI t = do
  (res, st) <- runStateT (runReaderT (runErrorT t) initTIEnv) initTIState
  return (res, st)
  where
    initTIEnv = TIEnv
    initTIState = TIState{tiSupply = 0}

newTyVar :: String -> TI Type
newTyVar prefix = do
  s <- get
  put s{tiSupply = tiSupply s + 1}
  return (TVar (prefix ++ show (tiSupply s)))
```

实例化函数会把类型方案中所有绑定的类型变量替换为新鲜类型变量。

```haskell
instantiate :: Scheme -> TI Type
instantiate (Scheme vars t) = do
  nvars <- mapM (\ _ -> newTyVar "a") vars
  let s = Map.fromList (zip vars nvars)
  return $ apply s t
```

下面是类型的一元化函数。对于两个类型 $t_1$ 和 $t_2$，`mgu(t1, t2)` 返回最一般一元化器。所谓一元化器，是一个替换 $S$，使得 $S(t_1) = S(t_2)$。

`varBind` 函数尝试把一个类型变量绑定到一个类型，并把该绑定作为替换返回；它会避免把变量绑定到自身，也会执行 occurs check，以发现循环类型错误。

```haskell
mgu :: Type -> Type -> TI Subst
mgu (TFun l r) (TFun l' r') = do
  s1 <- mgu l l'
  s2 <- mgu (apply s1 r) (apply s1 r')
  return (s1 `composeSubst` s2)
mgu (TVar u) t = varBind u t
mgu t (TVar u) = varBind u t
mgu TInt TInt = return nullSubst
mgu TBool TBool = return nullSubst
mgu t1 t2 = throwError $ "types do not unify: " ++ show t1 ++ " vs. " ++ show t2

varBind :: String -> Type -> TI Subst
varBind u t
  | t == TVar u          = return nullSubst
  | u `Set.member` ftv t = throwError $ "occurs check fails: " ++ u ++ " vs. " ++ show t
  | otherwise            = return (Map.singleton u t)
```

### 主类型推断函数

字面量的类型由 `tiLit` 函数推断。

```haskell
tiLit :: Lit -> TI (Subst, Type)
tiLit (LInt _)  = return (nullSubst, TInt)
tiLit (LBool _) = return (nullSubst, TBool)
```

`ti` 函数推断表达式的类型。类型环境必须包含表达式所有自由变量的绑定。返回的替换记录表达式对类型变量施加的类型约束，返回的类型则是表达式本身的类型。

```haskell
ti :: TypeEnv -> Exp -> TI (Subst, Type)
ti (TypeEnv env) (EVar n) =
  case Map.lookup n env of
    Nothing    -> throwError $ "unbound variable: " ++ n
    Just sigma -> do t <- instantiate sigma
                     return (nullSubst, t)

ti _ (ELit l) = tiLit l

ti env (EAbs n e) = do
  tv <- newTyVar "a"
  let TypeEnv env' = remove env n
      env'' = TypeEnv (env' `Map.union` (Map.singleton n (Scheme [] tv)))
  (s1, t1) <- ti env'' e
  return (s1, TFun (apply s1 tv) t1)

ti env exp@(EApp e1 e2) = do
  tv <- newTyVar "a"
  (s1, t1) <- ti env e1
  (s2, t2) <- ti (apply s1 env) e2
  s3 <- mgu (apply s2 t1) (TFun t2 tv)
  return (s3 `composeSubst` s2 `composeSubst` s1, apply s3 tv)
  `catchError` \e -> throwError $ e ++ "\n in " ++ show exp

ti env (ELet x e1 e2) = do
  (s1, t1) <- ti env e1
  let TypeEnv env' = remove env x
      t' = generalize (apply s1 env) t1
      env'' = TypeEnv (Map.insert x t' env')
  (s2, t2) <- ti (apply s1 env'') e2
  return (s1 `composeSubst` s2, t2)
```

这是类型推断器的主入口。它只是调用 `ti`，然后把返回的替换应用到返回的类型上。

```haskell
typeInference :: Map.Map String Scheme -> Exp -> TI Type
typeInference env e = do
  (s, t) <- ti (TypeEnv env) e
  return (apply s t)
```

### 测试

下面这些简单表达式（部分取自 Heeren 等人的论文）用于测试类型推断函数。

```haskell
e0 = ELet "id" (EAbs "x" (EVar "x"))
     (EVar "id")

e1 = ELet "id" (EAbs "x" (EVar "x"))
     (EApp (EVar "id") (EVar "id"))

e2 = ELet "id" (EAbs "x" (ELet "y" (EVar "x") (EVar "y")))
     (EApp (EVar "id") (EVar "id"))

e3 = ELet "id" (EAbs "x" (ELet "y" (EVar "x") (EVar "y")))
     (EApp (EApp (EVar "id") (EVar "id")) (ELit (LInt 2)))

e4 = ELet "id" (EAbs "x" (EApp (EVar "x") (EVar "x")))
     (EVar "id")

e5 = EAbs "m" (ELet "y" (EVar "m")
     (ELet "x" (EApp (EVar "y") (ELit (LBool True))) (EVar "x")))

e6 = EApp (ELit (LInt 2)) (ELit (LInt 2))
```

这个简单的测试函数会尝试推断给定表达式的类型。成功时，它会打印表达式及其类型；失败时，它会打印错误信息。

```haskell
test :: Exp -> IO ()
test e = do
  (res, _) <- runTI (typeInference Map.empty e)
  case res of
    Left err -> putStrLn $ show e ++ "\n " ++ err ++ "\n"
    Right t  -> putStrLn $ show e ++ " :: " ++ show t ++ "\n"
```

### 主程序

主程序只是对“测试”一节中给出的所有示例表达式推断类型，并将表达式及其推断出的类型一起打印出来；如果类型推断失败，就打印错误信息。

```haskell
main :: IO ()
main = mapM_ test [e0, e1, e2, e3, e4, e5, e6]

-- Collecting Constraints
-- main = mapM_ test' [e0, e1, e2, e3, e4, e5]
```

至此，类型推断算法的实现完成。

## 附录：漂亮打印

本附录为所有相关类型定义 `Show` 实例和漂亮打印函数。

```haskell
instance Show Type where
  showsPrec _ x = shows (prType x)

prType :: Type -> PP.Doc
prType (TVar n)   = PP.text n
prType TInt       = PP.text "Int"
prType TBool      = PP.text "Bool"
prType (TFun t s) = prParenType t PP.<+> PP.text "->" PP.<+> prType s

prParenType :: Type -> PP.Doc
prParenType t = case t of
  TFun _ _ -> PP.parens (prType t)
  _        -> prType t

instance Show Exp where
  showsPrec _ x = shows (prExp x)

prExp :: Exp -> PP.Doc
prExp (EVar name)     = PP.text name
prExp (ELit lit)      = prLit lit
prExp (ELet x b body) = PP.text "let" PP.<+> PP.text x PP.<+>
                        PP.text "=" PP.<+> prExp b PP.<+> PP.text "in" PP.$$ 
                        PP.nest 2 (prExp body)
prExp (EApp e1 e2)    = prExp e1 PP.<+> prParenExp e2
prExp (EAbs n e)      = PP.char '\\' PP.<> PP.text n PP.<+>
                        PP.text "->" PP.<+> prExp e

prParenExp :: Exp -> PP.Doc
prParenExp t = case t of
  ELet _ _ _ -> PP.parens (prExp t)
  EApp _ _   -> PP.parens (prExp t)
  EAbs _ _   -> PP.parens (prExp t)
  _          -> prExp t

instance Show Lit where
  showsPrec _ x = shows (prLit x)

prLit :: Lit -> PP.Doc
prLit (LInt i)  = PP.integer i
prLit (LBool b) = if b then PP.text "True" else PP.text "False"

instance Show Scheme where
  showsPrec _ x = shows (prScheme x)

prScheme :: Scheme -> PP.Doc
prScheme (Scheme vars t) =
  PP.text "All" PP.<+> PP.hcat (PP.punctuate PP.comma (map PP.text vars)) PP.<>
  PP.text "." PP.<+> prType t
```

## 附加：收集约束

原始文件在文档结束之后还保留了一段用于“收集约束”的代码。它不是正文排版的一部分，但仍随源文件给出；这里一并收录。

```haskell
test' :: Exp -> IO ()
test' e = do
  (res, _) <- runTI (bu Set.empty e)
  case res of
    Left err -> putStrLn $ "error: " ++ err
    Right t  -> putStrLn $ show e ++ " :: " ++ show t

data Constraint
  = CEquivalent Type Type
  | CExplicitInstance Type Scheme
  | CImplicitInstance Type (Set.Set String) Type

instance Show Constraint where
  showsPrec _ x = shows (prConstraint x)

prConstraint :: Constraint -> PP.Doc
prConstraint (CEquivalent t1 t2) =
  PP.hsep [prType t1, PP.text "=", prType t2]
prConstraint (CExplicitInstance t s) =
  PP.hsep [prType t, PP.text "<~", prScheme s]
prConstraint (CImplicitInstance t1 m t2) =
  PP.hsep [prType t1,
           PP.text "<=" PP.<> PP.parens
             (PP.hcat (PP.punctuate PP.comma (map PP.text (Set.toList m)))),
           prType t2]

type Assum = [(String, Type)]
type CSet = [Constraint]

bu :: Set.Set String -> Exp -> TI (Assum, CSet, Type)
bu m (EVar n) = do
  b <- newTyVar "b"
  return ([(n, b)], [], b)
bu m (ELit (LInt _)) = do
  b <- newTyVar "b"
  return ([], [CEquivalent b TInt], b)
bu m (ELit (LBool _)) = do
  b <- newTyVar "b"
  return ([], [CEquivalent b TBool], b)
bu m (EApp e1 e2) = do
  (a1, c1, t1) <- bu m e1
  (a2, c2, t2) <- bu m e2
  b <- newTyVar "b"
  return (a1 ++ a2, c1 ++ c2 ++ [CEquivalent t1 (TFun t2 b)], b)
bu m (EAbs x body) = do
  b@(TVar vn) <- newTyVar "b"
  (a, c, t) <- bu (vn `Set.insert` m) body
  return (a `removeAssum` x,
          c ++ [CEquivalent t' b | (x', t') <- a, x == x'],
          TFun b t)
bu m (ELet x e1 e2) = do
  (a1, c1, t1) <- bu m e1
  (a2, c2, t2) <- bu (x `Set.delete` m) e2
  return (a1 ++ removeAssum a2 x,
          c1 ++ c2 ++ [CImplicitInstance t' m t1 | (x', t') <- a2, x' == x],
          t2)

removeAssum [] _ = []
removeAssum ((n', _) : as) n | n == n' = removeAssum as n
removeAssum (a:as) n = a : removeAssum as n
```

[^1]: 原文复制自 [Martin Grabmüller 的 Algorithm W 页面](http://www.grabmueller.de/martin/www/pub/AlgorithmW.en.html) 并由 Wei Hu 编辑。遗憾的是，参考文献列表缺失。
[^2]: 最有帮助的参考资料是 [Generalizing Hindley-Milner Type Inference Algorithms](http://www.cs.uu.nl/research/techreps/repo/CS-2002/2002-031.pdf)，以及《Types and Programming Languages》第 22 章。
[^3]: Cornell 的一门课程也涉及该主题，并给出了 OCaml 实现：<http://www.cs.cornell.edu/Courses/cs3110/2009fa/lectures/lec26a.htm>。
