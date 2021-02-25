---
title: Haskell Database Implementation - Part 2, Domain Specific Language and Transactionality
---

This post is part 2 in a series on database implementation, part 1 is
[here]({% post_url 2021-02-15-database-implementation-part-1 %}).

In the last post I wrote about creating an underlying tree structure. This post is about creating a DSL and managing
transactions.

If you'd like to skip ahead and read the code, it's [here](https://github.com/dfithian/dfdb).

I could write an entire post on parsing, but I'll leave that for another day. For now, we'll assume that we can parse
all user input into our domain specific language.

## The DSL

In general, I wanted a DSL that could add and remove tables, and read and write data, and create and use indexes.

```haskell
newtype TableName = TableName { unTableName :: Text }

data AtomType
  = AtomTypeInt
  | AtomTypeString
  | AtomTypeBool

data Atom
  = AtomInt Int
  | AtomString Text
  | AtomBool Bool

newtype Row = Row { unRow :: [Atom] }

newtype ColumnName = ColumnName { unColumnName :: Text }

newtype IndexName = IndexName { unIndexName :: Text }

data WhereClause = WhereClause
  { _whereClauseColumn :: ColumnName
  , _whereClauseValue  :: Atom
  }

data ColumnDefinition = ColumnDefinition
  { _columnDefinitionName :: ColumnName
  , _columnDefinitionType :: AtomType
  }

data Statement
  = StatementSelect [ColumnName] TableName [WhereClause]
  | StatementInsert Row TableName
  | StatementCreate TableName [ColumnDefinition]
  | StatementCreateIndex IndexName TableName [ColumnName]
  | StatementDrop TableName
  | StatementDropIndex IndexName
```

I made a few important decisions for simplicity's sake:

1. A `SELECT` statement filters only using `AND`, and all comparisons must use equality
1. `INSERT` statements must specify every columnar value matching the order of the columns in the internal store
1. `DELETE` and `UPDATE` are not implemented

I'm sure there's a better way to enforce type safety internally than using `Atom` and `AtomType` but because I was
moving fast I didn't spend too much time on it.

## The database state

```haskell
newtype PrimaryKey = PrimaryKey { unPrimaryKey :: Int }

data Table = Table
  { _tableName           :: TableName
  , _tableDefinition     :: [ColumnDefinition]
  , _tableRows           :: TreeMap PrimaryKey Row
  , _tableNextPrimaryKey :: PrimaryKey
  , _tableIndices        :: [IndexName]
  }

data Index = Index
  { _indexName    :: IndexName
  , _indexTable   :: TableName
  , _indexColumns :: [ColumnName]
  , _indexData    :: TreeMap [Atom] [Row]
  }

data Database = Database
  { _databaseTables  :: Map TableName Table
  , _databaseIndices :: Map IndexName Index
  }
```

A `Table` consists of a name and definition, plus the actual data, and some helpers like the next primary key and the
names of the indexes defined on this table.

An `Index` refers to a subset of columns on a table, and, for simplicity, duplicates the data in the table (stores
`[Row]`) instead of using pointers.

A `Database` consists of tables and indexes.

## Transactionality

Having specified the DSL for the database, I was interested in how transactions on the database would work. Enumerating
some of the key features of transactions allowed me to investigate which ones I wanted to implement.

1. Primitive operations like `BEGIN`, `COMMIT`, and `ROLLBACK`
1. Concurrency, and relatedly, isolation levels

Because I had already made the decision to keep the database in memory in part 1, concurrency and isolation levels
didn't make much sense to implement. Instead I focused on primitive operations after implementing autocommit.

### Naive autocommit implementation

My first pass on transactionality was to implement autocommit. This was helpful in the case where a table had one or
more indexes that needed to be updated during an insert, and it provided a way to abstract transactions from the
underlying code.

```haskell
data StatementFailureCode
  = StatementFailureCodeSyntaxError Text
  | StatementFailureCodeInternalError Text

newtype Transaction a = Transaction (StateT Database (Except StatementFailureCode) a)
  deriving (Functor, Applicative, Monad)

type MonadDatabase m = (MonadState Database m, MonadError StatementFailureCode m)

runTransaction :: (MonadState Database m) => Transaction a -> m (Either StatementFailureCode a)
runTransaction (Transaction mx) = do
  pre <- get
  let result = runExcept $ runStateT mx pre
  traverse (\(out, post) -> put post >> pure out) result
```

As an example, execute any sequence of statements using this underlying monad:

```haskell
newtype Output = Output { unOutput :: Text }

execute :: MonadDatabase m => Statement -> m Output
execute = \case
  StatementSelect cols tableName wheres -> ...
  StatementInsert row tableName -> ...
  StatementCreate tableName cols -> ...
  StatementCreateIndex indexName tableName cols -> ...
  StatementDrop tableName -> ...
  StatementDropIndex indexName -> ...
```

And run it:

```haskell
  runTransaction (Transaction (execute statement)) >>= \case
    Right output -> putStrLn $ unOutput output
    Left code -> case code of
      StatementFailureCodeSyntaxError err -> putStrLn err
      StatementFailureCodeInternalError err -> putStrLn err
```

While autocommit is simple and prevents database corruption, it doesn't provide a basic feature set, namely the
primitives `BEGIN`, `ROLLBACK`, or `COMMIT`.

### Less naive implementation

#### Transaction lifecycle

In order to implement these underlying primitives, I added constructors to the `Statement` DSL, created a state machine
for a transaction, and further abstracted the database away from the interpreter.

```haskell
-- Same as before, plus three operations
data Statement
  ...
  | StatementBegin
  | StatementCommit
  | StatementRollback

data TransactionStatus
  = TransactionStatusBegin
  | TransactionStatusAborted
  | TransactionStatusCommit
  | TransactionStatusRollback

data TransactionalDatabase = TransactionalDatabase
  { _transactionalDatabaseLastSavepoint :: Database
  , _transactionalDatabaseInner         :: Maybe (TransactionStatus, Database)
  }
```

The interpreter still operated on a `Database`, but _which_ database is determined by whether or not there's a currently
executing transaction. The transaction runner changed to read the status and perform the appropriate operations. The
interpreter was modified to change the transaction status based on which `Statement` constructor was passed in.

```haskell
newtype Transaction a = Transaction (StateT (Database, Maybe TransactionStatus) (Except StatementFailureCode) a)
  deriving (Functor, Applicative, Monad)

type MonadDatabase m = (MonadState (Database, Maybe TransactionStatus) m, MonadError StatementFailureCode m)
```

The autocommit branch works mostly like it used to, modifying the last savepoint, but will also detect changes to the
transaction status and initialize the transaction.

```haskell
runAutocommit :: (MonadState DFDB.Types.TransactionalDatabase m) => DFDB.Types.Database -> Transaction a -> m (Either DFDB.Types.StatementFailureCode a)
runAutocommit pre (Transaction mx) = case runExcept $ runStateT mx (pre, Nothing) of
  Left err -> pure $ Left err
  Right (out, (post, postStatusMay)) -> do
    case postStatusMay of
      Nothing -> assign DFDB.Types.transactionalDatabaseLastSavepoint post
      Just postStatus -> do
        put DFDB.Types.TransactionalDatabase
          { _transactionalDatabaseLastSavepoint = pre
          , _transactionalDatabaseInner = Just (postStatus, post)
          }
    pure $ Right out
```

The transaction runner branch passes in the transient inner database, reverts when rolled back, and overwrites the last
savepoint when committed.

```haskell
runInner :: (MonadState TransactionalDatabase m) => TransactionStatus -> Database -> Transaction a -> m (Either StatementFailureCode a)
runInner preStatus pre (Transaction mx) = case runExcept $ runStateT mx (pre, Just preStatus) of
  Left err -> do
    assign (transactionalDatabaseInner . _Just . _1) TransactionStatusAborted
    pure $ Left err
  Right (out, (post, postStatusMay)) -> do
    case postStatusMay of
      Nothing -> put TransactionalDatabase
        { _transactionalDatabaseLastSavepoint = post
        , _transactionalDatabaseInner = Nothing
        }
      Just TransactionStatusBegin -> assign (transactionalDatabaseInner . _Just . _2) post
      Just TransactionStatusAborted -> pure ()
      Just TransactionStatusCommit -> put TransactionalDatabase
        { _transactionalDatabaseLastSavepoint = post
        , _transactionalDatabaseInner = Nothing
        }
      Just TransactionStatusRollback -> assign transactionalDatabaseInner Nothing
    pure $ Right out
```

Finally, `runTransaction` branches based on whether there's a currently executing transaction.

```haskell
runTransaction :: (MonadState .TransactionalDatabase m) => Transaction a -> m (Either StatementFailureCode a)
runTransaction tx = do
  pre <- get
  case view transactionalDatabaseInner pre of
    Nothing -> runAutocommit (view transactionalDatabaseLastSavepoint pre) tx
    Just (status, innerPre) -> runInner status innerPre tx
```

#### Transaction interpretation

The `execute` function, also known as the interpreter, added three branches. The branches enforce the state machine
transitions for `TransactionStatus`, and otherwise allow the transaction runner to handle success and failure.

```haskell
-- Same as before, plus three branches
execute :: MonadDatabase m => Statement -> m Output
execute = \case
  ...
  StatementBegin -> do
    whenM (uses _2 (has _Just)) $
      throwError $ StatementFailureCodeInternalError "Already in a transaction"
    assign _2 (Just TransactionStatusBegin)
    pure $ Output "BEGIN"

  StatementCommit -> do
    use _2 >>= \ case
      Nothing -> throwError $ StatementFailureCodeInternalError "Not in a transaction"
      Just TransactionStatusBegin -> do
        assign _2 (Just TransactionStatusCommit)
        pure $ Output "COMMIT"
      Just _ -> throwError $ StatementFailureCodeInternalError "Transaction in a funky state; must roll back"

  StatementRollback -> do
    use _2 >>= \ case
      Nothing -> throwError $ StatementFailureCodeInternalError "Not in a transaction"
      Just _ -> do
        assign _2 (Just TransactionStatusRollback)
        pure $ Output "ROLLBACK"
```

## To be continued

Thanks for reading! The final post in the series, on indexes and performance, is
[here]({% post_url 2021-02-25-database-implementation-part-3 %}).

As always, I'd love to hear anything I've mixed up, especially in this case from database experts. Find me on fpchat
(`@dfithian`) or reddit (`/u/dfith`).
