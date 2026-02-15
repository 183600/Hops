注意:README.md和PLAN.md里的代码有可能不是最好的
# incremental — Implementation Plan

## Core Concepts

### Computation Graph

The system maintains a DAG (Directed Acyclic Graph) where:
- **Input nodes**: mutable values set by the user
- **Compute nodes**: derived values with tracked dependencies
- **Edges**: dependency relationships (compute → input or compute → compute)

### Change Propagation Strategy

We use a **two-phase** approach:

1. **Dirty Phase** (top-down): When an input changes, mark all
   transitive dependents as "possibly dirty"
2. **Recompute Phase** (bottom-up / demand-driven): When a node is
   observed, recompute only if truly dirty, with early cutoff

This combines the best of push-based and pull-based approaches.

## Architecture

```
┌─────────────────────────────────────────┐
│            Public API                   │
│  newInput, memo, observe, writeInput    │
├─────────────────────────────────────────┤
│         Computation Graph               │
│  Node registry, dependency edges        │
├─────────────────────────────────────────┤
│        Change Propagation               │
│  Dirty marking, demand-driven recompute │
├─────────────────────────────────────────┤
│        Dependency Tracker               │
│  Runtime tracking via Reader context    │
├─────────────────────────────────────────┤
│        Memoization / Cache              │
│  Value cache with equality cutoff       │
├─────────────────────────────────────────┤
│        Cycle Detection                  │
│  DFS-based cycle detection              │
└─────────────────────────────────────────┘
```

## Module Structure

```
src/
  Data/
    Incremental.hs                -- Re-export public API
    Incremental/
      Types.hs                    -- Core types
      Monad.hs                    -- Incremental monad
      Graph.hs                    -- Computation graph
      Node.hs                     -- Node operations
      Propagation.hs              -- Change propagation
      Tracker.hs                  -- Dependency tracking
      Cutoff.hs                   -- Early cutoff strategies
      Parallel.hs                 -- Parallel recomputation
      Debug.hs                    -- Tracing and visualization
```

## Phase 1: Core Types (Week 1)

### Types.hs

```haskell
module Data.Incremental.Types where

import Data.Unique (Unique)
import Data.Map.Strict (Map)
import Data.Set (Set)
import Data.IORef
import Data.Typeable (Typeable, TypeRep, typeOf)
import Unsafe.Coerce (unsafeCoerce)

-- | Unique identifier for a node
newtype NodeId = NodeId Unique
  deriving (Eq, Ord, Show)

-- | The global incremental state
data IncrState = IncrState
  { isNodes       :: !(IORef (Map NodeId SomeNode))
  , isCurrentEpoch :: !(IORef Epoch)
  , isDirtySet    :: !(IORef (Set NodeId))
  , isTracing     :: !(IORef Bool)
  , isTraceLog    :: !(IORef [TraceEvent])
  }

-- | Monotonically increasing change counter
newtype Epoch = Epoch Int
  deriving (Eq, Ord, Show, Num)

-- | Type-erased node wrapper
data SomeNode = forall a. Typeable a => SomeNode (Node a)

-- | A node in the computation graph
data Node a = Node
  { nodeId          :: !NodeId
  , nodeLabel       :: !String
  , nodeKind        :: !(NodeKind a)
  , nodeValue       :: !(IORef (Maybe a))
  , nodeStatus      :: !(IORef NodeStatus)
  , nodeDependsOn   :: !(IORef (Set NodeId))   -- nodes I read
  , nodeDependents  :: !(IORef (Set NodeId))    -- nodes that read me
  , nodeChangedAt   :: !(IORef Epoch)           -- when value last changed
  , nodeVerifiedAt  :: !(IORef Epoch)           -- when last verified clean
  , nodeCutoff      :: !(CutoffStrategy a)
  }

data NodeKind a where
  InputNode   :: NodeKind a
  ComputeNode :: Incremental a -> NodeKind a

data NodeStatus
  = Clean       -- Value is up to date
  | PossiblyDirty -- An input upstream changed; need to verify
  | Dirty       -- Definitely needs recomputation
  | Computing   -- Currently being recomputed (cycle detection)
  deriving (Show, Eq)

data CutoffStrategy a
  = NeverCutoff                    -- Always propagate
  | EqualityCutoff (a -> a -> Bool) -- Cutoff if equal
  | AlwaysCutoff                   -- Never propagate (constant)

data TraceEvent
  = TraceRecomputing !NodeId !String
  | TraceCutoff !NodeId !String
  | TraceSkipping !NodeId !String
  | TraceDirtyMarked !NodeId !String
  deriving (Show)

-- | Handle to an input node
newtype Input a = Input (Node a)

-- | Handle to a memoized computation node
newtype Memo a = Memo (Node a)
```

### Monad.hs

```haskell
module Data.Incremental.Monad where

import Data.Incremental.Types
import Control.Monad.IO.Class
import Control.Monad.Reader

-- | The incremental computation monad.
-- Reader carries the current state + the current "tracking context"
-- (which node is currently being computed, so we can record dependencies).
newtype Incremental a = Incremental
  { unIncremental :: ReaderT IncrContext IO a }
  deriving (Functor, Applicative, Monad, MonadIO)

data IncrContext = IncrContext
  { icState     :: !IncrState
  , icCurrentNode :: !(Maybe NodeId)  -- Nothing = top-level observation
  }

-- | Run an incremental computation
runIncremental :: Incremental a -> IO a
runIncremental (Incremental m) = do
  state <- initState
  runReaderT m (IncrContext state Nothing)

initState :: IO IncrState
initState = do
  nodes  <- newIORef Map.empty
  epoch  <- newIORef (Epoch 0)
  dirty  <- newIORef Set.empty
  tracing <- newIORef False
  traceLog <- newIORef []
  pure IncrState
    { isNodes        = nodes
    , isCurrentEpoch = epoch
    , isDirtySet     = dirty
    , isTracing      = tracing
    , isTraceLog     = traceLog
    }
```

## Phase 2: Node Operations (Week 2)

### Node.hs

```haskell
module Data.Incremental.Node where

import Data.Incremental.Types
import Data.Incremental.Monad

-- | Create a new input node
newInput :: (Typeable a) => String -> a -> Incremental (Input a)
newInput label initialValue = Incremental $ do
  state <- asks icState
  nid <- liftIO $ NodeId <$> newUnique
  node <- liftIO $ do
    val      <- newIORef (Just initialValue)
    status   <- newIORef Clean
    deps     <- newIORef Set.empty
    dependents <- newIORef Set.empty
    changed  <- newIORef (Epoch 0)
    verified <- newIORef (Epoch 0)
    pure Node
      { nodeId        = nid
      , nodeLabel     = label
      , nodeKind      = InputNode
      , nodeValue     = val
      , nodeStatus    = status
      , nodeDependsOn = deps
      , nodeDependents = dependents
      , nodeChangedAt = changed
      , nodeVerifiedAt = verified
      , nodeCutoff    = NeverCutoff
      }
  liftIO $ modifyIORef' (isNodes state)
                        (Map.insert nid (SomeNode node))
  pure (Input node)

-- | Write a new value to an input node
writeInput :: (Typeable a) => Input a -> a -> Incremental ()
writeInput (Input node) value = Incremental $ do
  state <- asks icState
  liftIO $ do
    -- Update value
    writeIORef (nodeValue node) (Just value)
    -- Bump epoch
    epoch <- atomicModifyIORef' (isCurrentEpoch state) (\e -> (e+1, e+1))
    writeIORef (nodeChangedAt node) epoch
    writeIORef (nodeStatus node) Clean
    -- Mark all transitive dependents as possibly dirty
    markDependentsDirty state (nodeId node)

-- | Read an input value (and track dependency)
readInput :: (Typeable a) => Input a -> Incremental a
readInput (Input node) = do
  trackDependency (nodeId node)
  ensureUpToDate node

-- | Create a memoized derived computation
memo :: (Typeable a) => String -> Incremental a -> Incremental (Memo a)
memo label computation = Incremental $ do
  state <- asks icState
  nid <- liftIO $ NodeId <$> newUnique
  node <- liftIO $ do
    val      <- newIORef Nothing
    status   <- newIORef Dirty  -- Initial: needs computation
    deps     <- newIORef Set.empty
    dependents <- newIORef Set.empty
    changed  <- newIORef (Epoch 0)
    verified <- newIORef (Epoch 0)
    pure Node
      { nodeId        = nid
      , nodeLabel     = label
      , nodeKind      = ComputeNode computation
      , nodeValue     = val
      , nodeStatus    = status
      , nodeDependsOn = deps
      , nodeDependents = dependents
      , nodeChangedAt = changed
      , nodeVerifiedAt = verified
      , nodeCutoff    = NeverCutoff
      }
  liftIO $ modifyIORef' (isNodes state)
                        (Map.insert nid (SomeNode node))
  pure (Memo node)

memoWithEq :: (Typeable a) => (a -> a -> Bool) -> String -> Incremental a -> Incremental (Memo a)
memoWithEq eq label computation = do
  m@(Memo node) <- memo label computation
  liftIO $ -- set cutoff strategy (would need mutable field or redesign)
  pure m -- simplified; real impl sets cutoff on the node

-- | Read a memo's value (and track dependency)
readMemo :: (Typeable a) => Memo a -> Incremental a
readMemo (Memo node) = do
  trackDependency (nodeId node)
  ensureUpToDate node

-- | Observe a node's value (top-level, no tracking)
observe :: (Typeable a) => Memo a -> Incremental a
observe (Memo node) = ensureUpToDate node
```

## Phase 3: Dependency Tracking (Week 3)

### Tracker.hs

```haskell
module Data.Incremental.Tracker where

import Data.Incremental.Types
import Data.Incremental.Monad

-- | Record that the currently-computing node depends on `depId`
trackDependency :: NodeId -> Incremental ()
trackDependency depId = Incremental $ do
  ctx <- ask
  case icCurrentNode ctx of
    Nothing -> pure ()  -- Top-level observation, no tracking
    Just currentId -> liftIO $ do
      let state = icState ctx
      nodes <- readIORef (isNodes state)

      -- Add forward edge: current -> dep
      case Map.lookup currentId nodes of
        Just (SomeNode currentNode) ->
          modifyIORef' (nodeDependsOn currentNode) (Set.insert depId)
        Nothing -> pure ()

      -- Add reverse edge: dep -> current
      case Map.lookup depId nodes of
        Just (SomeNode depNode) ->
          modifyIORef' (nodeDependents depNode) (Set.insert currentId)
        Nothing -> pure ()

-- | Execute a computation in a tracking context,
-- capturing all dependencies.
withTracking :: NodeId -> Incremental a -> Incremental (a, Set NodeId)
withTracking nid computation = Incremental $ do
  ctx <- ask
  let state = icState ctx

  -- Clear old dependencies for this node
  nodes <- liftIO $ readIORef (isNodes state)
  case Map.lookup nid nodes of
    Just (SomeNode node) -> liftIO $ do
      -- Remove reverse edges from old deps
      oldDeps <- readIORef (nodeDependsOn node)
      forM_ oldDeps $ \depId ->
        case Map.lookup depId nodes of
          Just (SomeNode depNode) ->
            modifyIORef' (nodeDependents depNode) (Set.delete nid)
          Nothing -> pure ()
      -- Clear forward edges
      writeIORef (nodeDependsOn node) Set.empty
    Nothing -> pure ()

  -- Run computation with this node as context
  let ctx' = ctx { icCurrentNode = Just nid }
  result <- liftIO $ runReaderT (unIncremental computation) ctx'

  -- Read back captured dependencies
  deps <- case Map.lookup nid nodes of
    Just (SomeNode node) -> liftIO $ readIORef (nodeDependsOn node)
    Nothing -> pure Set.empty

  pure (result, deps)
```

## Phase 4: Change Propagation (Week 4)

### Propagation.hs

```haskell
module Data.Incremental.Propagation where

import Data.Incremental.Types
import Data.Incremental.Tracker

-- | Mark all transitive dependents as PossiblyDirty
markDependentsDirty :: IncrState -> NodeId -> IO ()
markDependentsDirty state startId = do
  nodes <- readIORef (isNodes state)
  go nodes Set.empty [startId]
  where
    go _ _ [] = pure ()
    go nodes visited (nid : queue)
      | nid `Set.member` visited = go nodes visited queue
      | otherwise = do
          case Map.lookup nid nodes of
            Just (SomeNode node) -> do
              dependents <- readIORef (nodeDependents node)
              forM_ dependents $ \depId ->
                case Map.lookup depId nodes of
                  Just (SomeNode depNode) -> do
                    status <- readIORef (nodeStatus depNode)
                    when (status == Clean) $
                      writeIORef (nodeStatus depNode) PossiblyDirty
                  Nothing -> pure ()
              go nodes (Set.insert nid visited)
                       (queue ++ Set.toList dependents)
            Nothing -> go nodes (Set.insert nid visited) queue

-- | Ensure a node's value is up to date.
-- This is the core demand-driven recomputation.
ensureUpToDate :: Typeable a => Node a -> Incremental a
ensureUpToDate node = Incremental $ do
  ctx <- ask
  let state = icState ctx
  status <- liftIO $ readIORef (nodeStatus node)
  currentEpoch <- liftIO $ readIORef (isCurrentEpoch state)

  case status of
    Clean -> do
      -- Already up to date
      Just val <- liftIO $ readIORef (nodeValue node)
      pure val

    Computing -> do
      -- Cycle detected!
      error $ "Cycle detected involving node: " ++ nodeLabel node

    PossiblyDirty -> do
      -- Check if any dependency actually changed
      liftIO $ writeIORef (nodeStatus node) Computing
      deps <- liftIO $ readIORef (nodeDependsOn node)
      anyChanged <- liftIO $ anyM (hasChanged state node) (Set.toList deps)

      if anyChanged
        then recompute ctx node
        else do
          -- No upstream change → still clean
          liftIO $ do
            writeIORef (nodeStatus node) Clean
            writeIORef (nodeVerifiedAt node) currentEpoch
          Just val <- liftIO $ readIORef (nodeValue node)
          pure val

    Dirty -> do
      liftIO $ writeIORef (nodeStatus node) Computing
      recompute ctx node

-- | Check if a dependency has changed since we last verified
hasChanged :: IncrState -> Node a -> NodeId -> IO Bool
hasChanged state consumer depId = do
  nodes <- readIORef (isNodes state)
  case Map.lookup depId nodes of
    Just (SomeNode depNode) -> do
      -- First, ensure the dep itself is up to date
      -- (recursive demand-driven computation)
      -- ... simplified: just check epochs
      depChanged  <- readIORef (nodeChangedAt depNode)
      myVerified  <- readIORef (nodeVerifiedAt consumer)
      pure (depChanged > myVerified)
    Nothing -> pure True

-- | Actually recompute a node
recompute :: IncrContext -> Node a -> ReaderT IncrContext IO a
recompute ctx node = do
  case nodeKind node of
    InputNode -> do
      Just val <- liftIO $ readIORef (nodeValue node)
      liftIO $ writeIORef (nodeStatus node) Clean
      pure val

    ComputeNode computation -> do
      let nid = nodeId node
      -- Run computation with dependency tracking
      (newVal, _newDeps) <- liftIO $
        runReaderT (unIncremental (withTracking nid computation)) ctx

      -- Check cutoff
      oldVal <- liftIO $ readIORef (nodeValue node)
      let changed = case (nodeCutoff node, oldVal) of
            (EqualityCutoff eq, Just old) -> not (eq old newVal)
            (AlwaysCutoff, _)             -> False
            _                             -> True

      currentEpoch <- liftIO $ readIORef (isCurrentEpoch (icState ctx))

      liftIO $ do
        writeIORef (nodeValue node) (Just newVal)
        writeIORef (nodeStatus node) Clean
        writeIORef (nodeVerifiedAt node) currentEpoch
        when changed $
          writeIORef (nodeChangedAt node) currentEpoch

      -- Trace
      tracing <- liftIO $ readIORef (isTracing (icState ctx))
      when tracing $ liftIO $
        modifyIORef' (isTraceLog (icState ctx))
          (TraceRecomputing nid (nodeLabel node) :)

      pure newVal

anyM :: Monad m => (a -> m Bool) -> [a] -> m Bool
anyM _ []     = pure False
anyM f (x:xs) = do
  b <- f x
  if b then pure True else anyM f xs
```

## Phase 5: Scoped Invalidation (Week 5)

```haskell
module Data.Incremental.Scope where

-- Hierarchical scopes for bulk invalidation

data MemoScope = MemoScope
  { msName    :: !String
  , msNodes   :: !(IORef (Set NodeId))
  , msParent  :: !(Maybe MemoScope)
  }

withMemoScope :: String -> Incremental a -> Incremental a
withMemoScope name action = do
  -- Create scope, run action, register nodes under scope
  undefined

invalidateScope :: String -> Incremental ()
invalidateScope name = do
  -- Mark all nodes in scope as Dirty
  undefined
```

## Phase 6: Parallel Recomputation (Week 6)

```haskell
module Data.Incremental.Parallel where

-- Identify independent dirty subgraphs and recompute in parallel

observeParallel :: [Memo a] -> Incremental [a]
observeParallel memos = do
  -- Build dependency levels (topological sort)
  -- Recompute each level in parallel using async
  undefined
```

## Phase 7: Debug Visualization (Week 6)

```haskell
module Data.Incremental.Debug where

-- Export computation graph as DOT format for Graphviz
exportDot :: IncrState -> IO String
exportDot state = do
  nodes <- readIORef (isNodes state)
  -- Generate DOT graph with node labels, statuses, edges
  undefined

withTracing :: Incremental a -> Incremental (a, [TraceEvent])
withTracing action = do
  -- Enable tracing, run action, collect events
  undefined
```

## Testing Strategy

```haskell
-- Core invariant: incremental result == from-scratch result
prop_incrementalCorrectness :: Property
prop_incrementalCorrectness = property $ do
  -- Generate random DAG structure
  -- Generate random sequence of input changes
  -- After each change: verify incremental result == full recompute
  undefined

-- Performance property: recomputation is minimal
prop_minimalRecomputation :: Property
prop_minimalRecomputation = property $ do
  -- Change one input
  -- Verify only nodes in dependency cone are recomputed
  undefined

-- Cycle detection works
prop_cycleDetection :: Property
prop_cycleDetection = property $ do
  -- Create cyclic dependency
  -- Verify error is raised
  undefined
```

## Benchmarks

- Diamond dependency graph (10k nodes)
- Deep chain (1000 levels)
- Wide fan-out (1 input → 10k dependents)
- Comparison: full recompute vs incremental
- Comparison with shake's approach

## Dependencies

```cabal
build-depends:
    base        >= 4.16 && < 5
  , containers  >= 0.6
  , hashable    >= 1.4
  , unordered-containers >= 0.2
```

## Milestones

| Week | Deliverable |
|------|-------------|
| 1    | Core types, IncrState, Node |
| 2    | Input/Memo creation, basic read/write |
| 3    | Dependency tracking |
| 4    | Change propagation with early cutoff |
| 5    | Scoped invalidation, cycle detection |
| 6    | Parallel recomputation, debug tools |
| 7    | Property tests, correctness proofs |
| 8    | Benchmarks, documentation, Hackage release |
