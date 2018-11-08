# Haskell code

Here is some simple code to partition an either list

```haskell
module Main (main) where

part :: [Either a b] -> ([a],[b])
part []        = ([],[])
part (eab:eabs) = case eab of
    Left a  -> (a:as,bs)
    Right b -> (as,b:bs)
  where
    (as,bs) = part eabs

xs :: [()]
ys :: [()]
(xs,ys) = part (repeat $ Left ())

main :: IO ()
main = do
    print $ length xs
    print $ length ys
```

# STG/Desugared code

From `-ddump-simpl` (and reading the haskell report) it compiles to the following

```haskell
part eabs = case eabs of
    []       -> []
    eab:eabs -> let asbs = part eabs in case eab in
        Left  a -> (a:case asbs of (as,_) -> as,   case asbs of (_,bs) -> bs)
        Right b -> (  case asbs of (as,_) -> as, b:case asbs of (_,bs) -> bs)

xsys = part (repeat $ Left ())

main = do
    print (length (case xsys of (xs,_ ) -> xs)
    print (length (case xsys of (_ ,ys) -> ys)
```

# Evaluation

Running this through my mental evaluator tells me that the following happens as `length xs` consumes xs

## First element from xs

* evaluate: `xsys = part (repeat $ Left ())`
* allocate: `asbs = part (repeat $ Left ())`
* update: `xsys = (a:case asbs of (as,_) -> as, case asbs of (_,bs) -> bs)`

```haskell
ys = case xsys of (_,ys) -> ys
xsys = (a:case asbs of (as,_) -> as, case asbs of (_,bs) -> bs)
asbs = part (repeat $ Left ())
```

## Second element from xs

* evaluate: `asbs = part (repeat $ Left ())`
* allocate: `asbs' = part (repeat $ Left ())`
* update: `asbs = (a:case asbs' of (as,_) -> as,   case asbs' of (_,bs) -> bs)`

```haskell
ys = case xsys of (_,ys) -> ys
xsys = (a:case asbs of (as,_) -> as, case asbs of (_,bs) -> bs)
asbs = (a:case asbs' of (as,_) -> as, case asbs' of (_,bs) -> bs)
asbs' = part (repeat $ Left ())
```

## Third element from xs

* evaluate: `asbs' = part (repeat $ Left ())`
* allocate: `asbs'' = part (repeat $ Left ())`
* update: `asbs' = (a:case asbs'' of (as,_) -> as,   case asbs'' of (_,bs) -> bs)`

```haskell
ys = case xsys of (_,ys) -> ys
xsys = (a:case asbs of (as,_) -> as, case asbs of (_,bs) -> bs)
asbs = (a:case asbs' of (as,_) -> as, case asbs' of (_,bs) -> bs)
asbs' = (a:case asbs'' of (as,_) -> as, case asbs'' of (_,bs) -> bs)
asbs'' = part (repeat $ Left ())
```

## And so on

```haskell
ys = case xsys of (_,ys) -> ys
xsys = (a:case asbs of (as,_) -> as, case asbs of (_,bs) -> bs)
asbs = (a:case asbs' of (as,_) -> as, case asbs' of (_,bs) -> bs)
asbs' = (a:case asbs'' of (as,_) -> as, case asbs'' of (_,bs) -> bs)
asbs'' = (a:case asbs''' of (as,_) -> as, case asbs''' of (_,bs) -> bs)
asbs''' = (a:case asbs'''' of (as,_) -> as, case asbs'''' of (_,bs) -> bs)
...
```

# Expectations vs reality

So I would expect this to consume all available memory from the build up of `asbs` thunks kept alive under `ys`.  But it doesn't.  It just sits there with constant memory usage and 100% CPU usage forever.

Why is this?  What is going wrong in my mental evaluation mode?
