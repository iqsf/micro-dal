
# micro-dal

The lightweight Data Access Layer

```haskell
      withEngine "mydatabase.sqlite" $ \eng -> do
        withTransaction eng $ do
          mapM_ (store @MyDataType eng) values

        values <- listAll @MyDataType eng
```

## Why

To decouple the programs from the specific of the data storage.

To make the work with persistent data simple and provide a possibility to evolve from a naive
approach during the prototyping to enterprise level gradually, without changing the client's code.

Some ideas behind this library are taken from Martin Fowler's books, Craig Larman's  books, some
from the own experience and previous attempts.

## Status

The current status of the library is under heavy development / experimental.

It is being used in some internal projects, evolving along with them.

The current implementation has a simple and basic SQLite backend for the DAL.

See the cases below.

## Introduction

The basic idea is to hide the complexity of working with different data storages like key-value
databases or relational databases or CAS-storages or other sort of storages behind the minimalistic
set of interfaces, without taking care of the specific of the concrete data storage, like
relational databases.

The minimal stored data item has to have an identity or primary key and a possibility to be
stored/loaded. This is pretty enough for a lot of cases, cause all other features may be
impemented on top of store/load primitives:

```haskell
class HasKey a where
  data KeyOf a :: *
  key   :: a -> KeyOf a
  ns    :: NS a

class (Monad m, HasKey a) => SourceStore a m e where
  store :: e -> a -> m (KeyOf a)
  load :: e -> KeyOf a -> m (Maybe a)
```

I.e. indexes and sets may be implemented as a Haskell collections of values,
and may be stored/loaded as far as those values fit the memory.

To make the moment of the memory exhaustion happen later, there is the ```HashRef```
abstraction that represents the hash-addressed object, that may be in fully-loaded
state or in unloaded state (in this case it's just a cryptographic hash of the
referenced value).

```haskell
newtype HashRef (ns :: Symbol) a = HashRef (Either B58 a)
                                   deriving(Eq,Ord,Show,Data,Generic)
```

Thus, to make the data value storable, you merely have to specify the HashRef type for it and the
instance of Store typeclass, for an instance:

```haskell
type HashedInt = HashRef "ints" Int
instance Store HashedInt

-- ...

withEngine optInMemory $ \eng -> do

	replicateM_ 1000 $ do
	  ivalues <- generate arbitrary :: IO [Int]
	  forM_ ivalues $ \i -> do
		k <- store @HashedInt eng (hashRefPack i)
		ii <- load @HashedInt eng k
		Just i `shouldBe` (fromJust $ hashRefUnpack <$> ii)

```

So far, we have the minimal set of the primitives that covers a significant
part of the persistent data use cases.

## TODO

  - Implement abstraction for querying the data
  - Make it suitable to work with relational databases, including
    complex queries, relations, indexes, etc.


## Cases

### General

```haskell

data SomeData = SomeData Word32 Word32
                deriving (Eq,Ord,Show,Data,Generic)

-- HaskKey means that the data item has, at least
-- a primary key and *namespace*.
-- The primary key is a primary key,
-- namespace in case of a relational database means
-- a table, in a case of any other kind of storage
-- it could mean "bag" or the namespace. I.e. it's a set
-- of all values of the given type.
-- Namespace is required to provide a possibility
-- to enumerate all items of the given type.

instance HasKey SomeData where

  newtype KeyOf SomeData = SomeDataKey Word32
                           deriving (Eq,Ord,Show,Generic,Store)

  key (SomeData a _) = SomeDataKey a
  ns = "somedata"

-- ...

replicateM 1000 $ do
	withEngine optInMemory $ \eng -> do

	  els  <- Map.fromList <$> generate arbitrary :: IO (Map Word32 Word32)
	  let vals  = [ SomeData k v | (k,v) <- Map.toList els ]
	  mapM (store eng) vals
	  vals2 <- listAll @SomeData eng

	  vals2 `shouldMatchList` vals

```



