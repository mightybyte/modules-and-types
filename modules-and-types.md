% Doing More with Modules and Types
% Writing code that can't go wrong
% Doug Beardsley

# Programmers will ALWAYS make mistakes

- ![oops](http://i.imgur.com/Uko3wuw.png)
- Progress dictates that complexity will be near the limits of our
  capabilities.
- What do we do about this?
    - Make the computer prevent as many of our mistakes as possible
    - Encapsulation
    - Invariants
- How does Haskell accomplish this?
    - Strong static type system
    - Purity
    - Module system

# Two Misconceptions

- Haskell's type system isn't much more useful than, say, Java's
    - Might not be stated in those words, but that's the main thrust of the
      criticism.
    - Common in the dynamic language camp, where some seem to think that just
      associating a type with something doesn't gain you all that much.

- Haskell's module system is not very powerful
    - Usually in comparison to ML.
    - Lacks interfaces
    - But still quite useful
        - Encapsulation / Implementation hiding
        - Enforcing invariants

# Outline

- Four examples:
    - Encapsulation: A simple stack
    - Interdependent data
    - Order of computations
    - Security checks

# Example: Stack

```haskell
module Stack where

newtype Stack a = Stack { unStack :: [a] }

push :: a -> Stack a -> Stack a
push a (Stack as) = Stack (a:as)

pop :: Stack a -> (Maybe a, Stack a)
pop (Stack []) = (Nothing, Stack [])
pop (Stack (a:as)) = (Just a, Stack as)

emptyStack :: Stack a
emptyStack = Stack []
```

- Standard OO-style example
- Module exports the following symbols:
    - Stack (type constructor), Stack (data constructor), unStack
      (getter and setter), push, pop, emptyStack

# Encapsulation

- We want to hide our stack's implementation
- In Java, encapsulation is mostly done by classes.
- Haskell data types don't have this
- Instead, Haskell uses modules

# Module Syntax

```haskell
module Stack where
```

- Exports everything

# Module Syntax

```haskell
module Stack where
```

- Exports everything

```haskell
module Stack
  ( Stack(..)
  ) where
```

- Exports Stack (type constructor), Stack (data constructor),
  getters and setters for all fields.

# Module Syntax

```haskell
module Stack where
```

- Exports everything

```haskell
module Stack
  ( Stack(..)
  ) where
```

- Exports Stack (type constructor), Stack (data constructor),
  getters and setters for all fields.

```haskell
module Stack
  ( Stack
  ) where
```

- Only exports the type constructor, not the data constructor or any of the
  field accessors.
- Other modules can't construct a Stack

# Encapsulation with Modules

```haskell
module Stack
  ( Stack
  , push, pop, emptyStack
  ) where

newtype Stack a = Stack { unStack :: [a] }

push :: a -> Stack a -> Stack a
push a (Stack as) = Stack (a:as)

pop :: Stack a -> (Maybe a, Stack a)
pop (Stack []) = (Nothing, Stack [])
pop (Stack (a:as)) = (Just a, Stack as)

emptyStack :: Stack a
emptyStack = Stack []
```

- Module exports the following symbols:
    - Stack (type constructor), push, pop, emptyStack
- Implementation is hidden, emptyStack is the only constructor
- We could change the implementation to an Array or Vector and no user-facing
  code would be affected.

# Real World: Encapsulation in Snap

- Snap makes heavy use of this kind of encapsulation.
- Snap 0.3

```haskell
newtype Snap a = Snap
    { unSnap :: StateT SnapState (Iteratee ByteString IO) (Maybe (Either Response a)) }
```

- Snap 0.9

```haskell
newtype Snap a = Snap
    { unSnap :: StateT SnapState (Iteratee ByteString IO) (SnapResult a) }
```

# What if you need more flexibility?

- Sometimes users legitimately need access to the internals.
- Two conventions
    - Internal modules (Snap.Internal)
    - unsafeFoo functions
- If a user uses an unsafe function or imports a .Internal module, they can't
  complain if something breaks.
- Very common convention, used by bytestring, text, lens, etc.

# Example: Enforcing Invariants

- Simple data structure

```haskell
data Team = Team
    { teamName    :: Text
    , teamCountry :: Text
    , teamAddress :: Address
    }

data Address = Address
    { addrStreet  :: Text
    , addrCity    :: Text
    , addrState   :: Text
    , addrPostal  :: Text
    , addrCountry :: Text
    }
```

# Example: Enforcing Invariants

- Simple data structure

```haskell
data Team = Team
    { teamName    :: Text
    , teamCountry :: Text -- Duplicate data
    , teamAddress :: Address
    }

data Address = Address
    { addrStreet  :: Text
    , addrCity    :: Text
    , addrState   :: Text
    , addrPostal  :: Text
    , addrCountry :: Text -- Duplicate data
    }
```

- Problem
    - Country stored in two places
    - Might get out of sync


# Solution: Smart Constructor Pattern

```haskell
module Types.Team
  ( Team -- Note this is not Team(..)
  , mkTeam
  , teamName
  ) where

data Team = Team
    { teamName    :: Text
    , teamCountry :: Text -- Duplicate data
    , teamAddress :: Address
    }

mkTeam :: Text -> Address -> Team
mkTeam name addr = Team name (addrCountry addr) addr
```

- Only export the type constructor, not the data constructor.
- Define our own "smart constructor" and export that instead.
- Don't export field symbols for teamCountry and teamAddress.
- Now it is **impossible** to ever have a team with inconsistent addresses
  (outside the Types.Team module).

# Solution: Smart Constructor Pattern

```haskell
module Types.Team
  ( Team -- Note this is not Team(..)
  , mkTeam
  , teamName
  ) where

data Team = Team
    { teamName    :: Text
    , _teamCountry :: Text -- Duplicate data
    , _teamAddress :: Address
    }

mkTeam :: Text -> Address -> Team
mkTeam name addr = Team name (addrCountry addr) addr
```

- Only export the type constructor, not the data constructor.
- Define our own "smart constructor" and export that instead.
- Don't export field symbols for teamCountry and teamAddress.
- Now it is **impossible** to ever have a team with inconsistent addresses
  (outside the Types.Team module).
- If you really want to export those names, then name your field accessors
  something different.

# Analysis

- What features of Haskell made this work?

# Analysis

- What features of Haskell made this work?

1. Strong static type system

# Analysis

- What features of Haskell made this work?

1. Strong static type system
2. Purity

# Analysis

- What features of Haskell made this work?

1. Strong static type system
2. Purity
3. Module system


# Analysis

- What features of Haskell made this work?

1. Strong static type system
2. Purity
3. Module system

- If any one of these three is missing, then you lose the guarantee.

# Analysis

- What features of Haskell made this work?

1. Strong static type system
2. Purity
3. Module system

- If any one of these three is missing, then you lose the guarantee.
- All of this may seem simple and obvious, but it can get more complicated...

# More powerful possibilities

- mkTeam can do **turing-complete** calculations
    - Serialization of some kind of blob structure
    - Sorting to ensure structures are kept in a certain order
    - Caching expensive computations and ensuring that the cache is up to date
    - Smart constructor can be monadic
    - Enforcing all kinds of other invariants

# More powerful possibilities

- mkTeam can do **turing-complete** calculations
    - Serialization of some kind of blob structure
    - Sorting to ensure structures are kept in a certain order
    - Caching expensive computations and ensuring that the cache is up to date
    - Smart constructor can be monadic
    - Enforcing all kinds of other invariants
- Big point: Haskell types give you arbitrarily complex turing-complete
  guarantees!
- This comes directly from the synergy between string static types, purity,
  and the module system.

# Example: Encapsulation and Template Haskell

- You're generating symbols from TemplateHaskell (e.g. groundhog, persistent,
  lens)

```haskell
module Types.Team where

data Team = Team
    { teamName    :: Text
    , teamCountry :: Text
    , teamAddress :: Address
    }

mkPersist defCodegen [groundhog|
- entity: Team
  dbName: team
|]
```

# Example: Encapsulation and Template Haskell

- You're generating symbols from TemplateHaskell (e.g. groundhog, persistent,
  lens)

```haskell
module Types.Team where

data Team = Team
    { teamName    :: Text
    , teamCountry :: Text
    , teamAddress :: Address
    }

mkPersist defCodegen [groundhog|
- entity: Team
  dbName: team
|]
```

- Problem
    - mkPersist exports a LOT of symbols.
    - You don't know what they are.
    - Exporting them explicitly is too painful.
    - If you don't export explicitly, you export too much.

# Solution: Separate Modules

```haskell
module Types.Team.Smart
  ( Team
  , mkTeam
  , teamName
  ) where

data Team = Team
    { teamName    :: Text
    , teamCountry :: Text
    , teamAddress :: Address
    }

mkTeam :: Text -> Address -> Team
mkTeam name addr = Team name (addrCountry addr) addr
```

```haskell
module Types.Team
  ( module Types.Team
  , module Types.Team.Smart
  ) where

import Types.Team.Smart

mkPersist defCodegen [groundhog|
- entity: Team
  dbName: team
|]
```

# Benefits

- We still have all of the above safety
- Everyone else only imports Types.XYZ
- Good ideas
    - Put as little code as possible in the .Smart module.
    - Any time you're working on a .Smart module, you know you need to be
      careful to maintain the invariants establised by the type's smart
      constructor.
    - Easy to find .Smart modules and put more audit scrutiny into them.

# Example: Snap and Ordering Constraints

- The `Initializer` monad had ordering constraints.
- I wanted to make it impossible for the user to get the ordering wrong.
- But how do you enforce ordering with types?

# Adding a newtype

```haskell
newtype SnapletInit b v = SnapletInit (Initializer b v (Snaplet v))
```

```haskell
makeSnaplet :: Text
            -> Text
            -> Maybe (IO FilePath)
            -> Initializer b v v
            -> SnapletInit b v
```

```haskell
nestSnaplet :: ByteString
            -> SnapletLens v v1
            -> SnapletInit b v1
            -> Initializer b v (Snaplet v1)
```

- In order to use nestSnaplet (the place that had ordering constraints) you have to have a SnapletInit.
- The **only** way to get a SnapletInit is with makeSnaplet.
- makeSnaplet contains the stuff that has to happen first.
- nestSnaplet contains stuff that has to happen second.
- The user never sees anything except Initializer.
- Impossible for the end user to get it wrong

# Example: Security

- Problem
    - Don't want to think about security 100% of the time
    - Can't forget about security when it matters
- Solution
    - High-level code is security-oblivious
    - Running low-level code requires a security check
- How do we enforce this?

# Layered architecture

- Haskell can enforce invariants on computations, too
    - e.g. "This computation always makes a security check before calling low-level code"
- Technique: Monad with smart constructor

# SecureT

- `SecureT` is a monad transformer that requires a check before running lower-level code from the higher-level monad

```haskell
module SecureT (SecureT, runSecureT, liftChecked) where
-- ...

data User = User { userCapabilities :: Set Capability }

newtype SecureT m a = SecureT (ReaderT User m a)
  deriving (Functor, Monad)

runSecureT :: SecureT m a -> User -> m a
runSecureT (SecureT action) user = runReaderT action user

getCurrentUser :: Monad m => SecureT m User
getCurrentUser = SecureT ask

liftChecked :: Monad m => Capability -> m a -> SecureT m a
liftChecked requiredCapability action = do
  user <- getCurrentUser
  if requiredCapability `Set.member` userCapabilities user
    then SecureT $ lift action
    else fail $ "Permission denied: " ++ show requiredCapability
```

# Making it Safe
- Export `SecureT` type, but not `SecureT` constructor
    - `liftChecked` is the smart constructor
- Note: instances are always exported (implicitly)
    - So, do NOT implement `MonadTrans` or `MonadIO`
        - `lift` and `liftIO` would bypass smart constructor

# Making it abstract
- What if we want multiple implementations?
- Make a typeclass

```haskell
class MonadChecked capability t where
  liftChecked :: Monad m => capability -> m a -> t m a

instance MonadChecked Capability SecureT where
  liftChecked requiredCapability action = do
    user <- getCurrentUser
    if requiredCapability `Set.member` userCapabilities user
      then SecureT $ lift action
      else fail $ "Permission denied: " ++ show requiredCapability

```

- To move `liftChecked` into a typeclass, we just replace the hard-coded `Capability` with typeclass parameter `capability` and `SecureT` with `t`

# Notes:
- Typeclasses that describe a monad with particular behavior are traditionally named "`Monad`*"
    - `MonadReader`, `MonadWriter`, `MonadState`, `MonadIO`, etc.
- Implementers of `MonadChecked` must still be careful about exports, just like before
    - However, code that is fully polymorphic is safe, regardless:
```haskell
f :: (Monad m, MonadChecked Capability t) => t m a
```
    - Since `f`'s type signature doesn't include MonadIO, it can't use it, even if an instance exists for some types `t` and `m`

# Summary

- Haskell's module system is crucial to constructing ADTs that hide their
  representation.
- The combination of the module system with Haskell's purity and strong types
  makes it uniquely poweful compared to today's mainstream languages in
  providing guarantees about your code.
    - All the encapsulation OO languages talk about
    - Consistency guarantees that aren't possible in other languages (because
      of monads and purity)
    - Guarantee the order of computations
    - Ensure security checks happen

