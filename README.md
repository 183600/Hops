# incremental

A demand-driven incremental computation framework for Haskell.

Efficiently recompute only what has changed.

Inspired by [Adapton](http://adapton.org/),
Jane Street's [Incremental](https://github.com/janestreet/incremental),
and [Salsa](https://github.com/salsa-rs/salsa) (Rust).

## Motivation

Many applications need to repeatedly compute values over changing inputs:

- **Compilers / Language Servers**: re-typecheck only changed files
- **Build Systems**: rebuild only affected targets
- **Spreadsheets**: recalculate only dependent cells
- **UI Frameworks**: re-render only changed components

`incremental` provides a framework where you declare computations
once, and the framework automatically tracks dependencies and
minimizes recomputation.

## Quick Start

```haskell
import Data.Incremental

example :: IO ()
example = runIncremental $ do
  -- Create mutable inputs
  x <- newInput "x" (10 :: Int)
  y <- newInput "y" (20 :: Int)

  -- Define derived computations (automatically tracked)
  sum_ <- memo "sum" $ do
    a <- readInput x
    b <- readInput y
    pure (a + b)

  product_ <- memo "product" $ do
    a <- readInput x
    b <- readInput y
    pure (a * b)

  combined <- memo "combined" $ do
    s <- readMemo sum_
    p <- readMemo product_
    pure (s + p)

  -- First computation
  r1 <- observe combined
  liftIO $ print r1  -- 230 (10+20 + 10*20)

  -- Change only x
  writeInput x 15

  -- Only "sum", "product", and "combined" recompute
  -- The framework knows the dependency graph
  r2 <- observe combined
  liftIO $ print r2  -- 335 (15+20 + 15*20)
```

## Features

### Automatic Dependency Tracking

```haskell
-- Dependencies are tracked at runtime.
-- No need to declare them manually.
node <- memo "conditional" $ do
  flag <- readInput useFeatureX
  if flag
    then readInput featureXConfig  -- dependency on featureXConfig
    else pure defaultConfig        -- NO dependency on featureXConfig
```

### Early Cutoff

```haskell
-- If a node's recomputed value is the same as before,
-- downstream nodes are NOT recomputed.
normalized <- memoWithEq (==) "normalized" $ do
  raw <- readInput rawText
  pure (map toLower raw)

-- If rawText changes from "Hello" to "hello",
-- normalized stays "hello" → downstream skipped
```

### Memoization Scopes

```haskell
-- Hierarchical memos for structured invalidation
withMemoScope "module-A" $ do
  ast   <- memo "parse" $ parseFile "A.hs"
  typed <- memo "typecheck" $ typecheckAST =<< readMemo ast

-- Invalidate everything under "module-A"
invalidateScope "module-A"
```

### Cycle Detection

```haskell
-- Cycles are detected at runtime with clear error messages
a <- memo "a" $ readMemo b  -- Error: cycle detected: a -> b -> a
b <- memo "b" $ readMemo a
```

### Debug Tracing

```haskell
-- See what gets recomputed
withTracing $ do
  writeInput x 42
  observe result
  -- [trace] recomputing "sum" (input "x" changed)
  -- [trace] recomputing "combined" ("sum" changed)
  -- [trace] skipping "product" (inputs unchanged)
```

### Parallel Recomputation

```haskell
-- Independent nodes can be recomputed in parallel
r <- observeParallel [node1, node2, node3]
```

## Use Cases

### Build System Core

```haskell
buildSystem :: Incremental ()
buildSystem = do
  src <- newInput "src" =<< readSourceFiles
  deps <- memo "deps" $ analyzeDeps <$> readInput src
  obj  <- memo "compile" $ compile <$> readInput src
  bin  <- memo "link" $ link <$> readMemo obj <*> readMemo deps
  observe bin
```

### Language Server

```haskell
lspLoop :: Incremental ()
lspLoop = forever $ do
  change <- liftIO getNextFileChange
  writeInput (fileInput change) (newContents change)

  diagnostics <- observe diagnosticsNode
  liftIO $ publishDiagnostics diagnostics
```

## Installation

```
cabal install incremental
```
```
