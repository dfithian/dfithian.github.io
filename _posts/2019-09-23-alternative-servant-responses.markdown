Last week I had an interesting run-in with a Servant API type. I was requesting from an API that
could return two possible responses. The first was 200 OK, and a JSON body. The other was 204 No
Content, and, well, no content. I struggled for a while wrestling the compiler and searching in vain
online for resources before finally arriving at an answer that worked for me. Below is the code!

Full example [here](https://github.com/dfithian/alternative-servant-demo).

## Imports

Import the right stuff:

```haskell
import ClassyPrelude
import Data.Aeson (FromJSON, parseJSON)
import Data.Functor.Alt ((<!>))
import Data.Proxy (Proxy (Proxy))
import Servant ((:<|>) ((:<|>)), (:>), Get, GetNoContent, JSON, NoContent, QueryParam)
import Servant.Client (ClientEnv, ClientM, client, runClientM)
import Servant.Client.Core (ClientError)
```

## API Type

Here's our response and API type. In my particular case it was a paging endpoint, so I've included
that parameter. Depending on server logic, it can either return "200 OK" and a `Baz` or "204 No
Content" and no content.

```haskell
data Baz = Baz

instance FromJSON Baz where
  parseJSON _ = pure Baz

type BazApi = "foo" :> "bar" :> QueryParam "page" Int :> (Get '[JSON] [Baz] :<|> GetNoContent '[] NoContent)
```

## Client Function

Here's our function for invoking the API. It matches the type of the API itself. I typically write
these helpers so that I can erase the call to `client (Proxy @MyApiType)`. Notice that we have the
monad `ClientM` on either side of the `:<|>` type - we will have to pattern match and then examine
which response we will end up with.

```haskell
getBaz :: Maybe Int -> ClientM [Baz] :<|> ClientM NoContent
getBaz = client (Proxy @BazApi)
```

## Using the Response

Finally, we have to figure out how to use this response. The signature above is tricky, but the
important bit here is that `:<|>` is a constructor of two `ClientM` values. Since there are two
possible responses and the API will return exactly one of them, we have to figure out how to unify
the two `ClientM` somehow. We know that we can try to bind the first value (the `Baz`) and if it
fails it should be the second value (the `NoContent`). Initially I was unaware of how to do this but
while inspecting the `ClientM` type in Haddock I found a clue: `instance Alt ClientM`. Looking at
the definition of `Alt` I found `(<!>) :: f a -> f a -> f a`, which looked like what I wanted. Here
our `f` is `ClientM` and our `a` is `[Baz]`.

```haskell
runGetBaz :: (MonadIO m) => ClientEnv -> Maybe Int -> m (Either ClientError [Baz])
runGetBaz env page = liftIO . flip runClientM env $ do
  -- pattern match on the possible results and use the one that is successful via Alt (<!>)
  case getBaz page of
    someRecords :<|> noRecords -> someRecords <!> ([] <$ noRecords)
```

## Conclusion

That's it! The big takeaway here for me was that `:<|>` isn't just a type constructor, the API can
also return responses that are nested in the `:<|>` value constructor. Servant provides the `Alt`
instance on `ClientM` in order to bind on the possible return values.
