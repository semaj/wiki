# Monad Transformers

Let's say we have:

```haskell
primSend :: Msg -> IP -> Node -> IO ()
```

which sends a message to a node in IO. 

This is roughly what the State type looks like. It's a function.
```haskell
type State s a = \s -> (a, s)  -- in this case, s is a Node
```

This is roughly what the monad instance of State looks like:
```haskell
...

return x = \s -> (x, s)

(>>=) :: m a -> (a -> m b) -> m b
(f >>= g) =  \s ->
                let (a, s') = f s
                    nextState = g a
                in nextState s'

get :: State (s, s)
get = \s -> (s, s)

put :: s -> State s ()
put s = \s' -> ((), s) -- puts modified version of state


send :: Msg -> IP -> State Node ()
send m ip = do
  s <- get
  put (modified s)
```
All we've done there is modify the state of the Node. But what if we want send to call `primSend`?

Let's combine the State and IO monads.

```haskell
StateIO s a = s -> IO (a, s)

return x = \s -> return (x, s) -- this second return is the IO monad's return

liftIO :: IO a -> StateIO s a
liftIO m = \s -> do
              a <- m -- IO
              return (a, s)

run :: s -> StateIO s a -> IO a
run :: s -> (s -> IO (a, s)) -> IO a
```

Or more generally, since we've only used the IO monad interface when referencing IO...

```
StateT m s a = s -> m (a, s)

type StateIO s a = StateT IO s a 
```

More concretely:


```haskell
 
type StateIO s a = StateT.StateT s IO a
```



