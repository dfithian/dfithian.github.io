---
title: Applying File Changes with `fix` and GADTs
---

For a description of `prune-juice`, see previous post on [Pruning unused Haskell dependencies]({% post_url 2021-03-08-pruning-unused-haskell-dependencies %}).

Prune Juice 0.7 has been [released]({% post_url 2021-11-12-prune-juice-0.7-released %})!

Since releasing [prune-juice](https://hackage.haskell.org/package/prune-juice), I have received a number of requests
asking to apply the unused dependencies directly to the cabal files. It ended up being a lot harder than I expected
to implement, and I'm proud to say that it's finally supported!

This post goes into the challenges I encountered implementing incremental file changes, and how I solved them with
custom parsers, `fix`, and GADTs.

## Simple Solution to Apply Changes

The first thing I wanted to do was figure out a simple solution overwrite the cabal file. The easiest way to do that
was to use the built-in Cabal library, iterate over the targets with unused dependencies, and overwrite the file with
the changes. That was simple enough by folding over each target and stripping out the dependencies:

```haskell
stripDependencies :: Set T.DependencyName -> [Dependency] -> [Dependency]
stripDependencies dependencies = filter (\x -> not (Set.member (T.mkDependencyName next) dependencies))
```

However, while many _import_ formats are supported for Cabal, only one _output_ format is supported. Standard
indentation, multiple `default-extensions` per line, and adding version ranges for `build-depends` entries are a few
examples of Cabal's pretty-printing behavior. If I were to use this strategy in the wild, for almost every user,
running `prune-juice` would make changes to the cabal file other than the pruned dependencies.

## Using the Relevant Bits

I set out to parse the cabal file into its relevant parts - targets, their dependencies, and everything else - and
then attempt to modify each section in-place so as to minimize the changes to the file.

### Parsing

For reference, I used [megaparsec](https://hackage.haskell.org/package/megaparsec) with the type synonym `type Parser =
Parsec Void String`

Here's the AST:

```haskell
-- |An indented section.
data NestedSection
  = BuildDependsNestedSection Int [String]
  | OtherNestedSection Int [String]
  deriving (Eq, Ord, Show)

-- |A top-level section.
data Section
  = TargetSection T.CompilableType (Maybe T.CompilableName) [NestedSection]
  | OtherSection [String]
  deriving (Eq, Ord, Show)
```

And the parser for the top-level `library` section:

```haskell
indentedLines :: Int -> Parser [String]
indentedLines numSpaces =
  -- parses lines, including empty lines, until the indentation is less than `numSpaces`

nestedSection :: Parser NestedSection
nestedSection = do
  numSpaces <- length <$> some (char ' ')
  let buildDepends = do
        void $ string "build-depends:"
        BuildDependsNestedSection numSpaces <$> indentedLines numSpaces
      ...
  buildDepends <|> ...

nestedSections :: Parser [NestedSection]
nestedSections = some nestedSection

section :: Parser Section
section =
  let lib = do
        void $ string "library"
        hspace
        void eol
        TargetSection T.CompilableTypeLibrary Nothing <$> nestedSections
      ...
  in lib <|> ...
```

I also wrote a `render` function plus some tests to verify that parsing, followed by rendering, resulted in the
original input.

### Fold/filter

Once it was parsed, it was pretty easy to apply the changes. I wrote a regular expression to match dependency names,
then filtered the parsed dependencies.

```haskell
dependencyNameRegex :: Regex
dependencyNameRegex = mkRegex "^ *([a-zA-Z0-9\\-]+).*$"

matchDependencyName :: String -> Maybe T.DependencyName
matchDependencyName str = Just . T.DependencyName . pack =<< T.headMay =<< matchRegex dependencyNameRegex str

stripOneBuildDepends :: String -> Set T.DependencyName -> Maybe String
stripOneBuildDepends input dependencies =
  let output = intercalate "," . mapMaybe go . fmap unpack . splitOn "," . pack $ input
  in case not (null output) && all ((==) ' ') output of
      True -> Nothing
      False -> Just output
  where
    go x = case matchDependencyName x of
      Nothing -> Just x
      Just dep -> case Set.member dep dependencies of
        True -> Nothing
        False -> Just x

-- |Strip any dependencies from @build-depends@.
stripBuildDepends :: [String] -> Set T.DependencyName -> [String]
stripBuildDepends buildDepends dependencies = mapMaybe (\x -> stripOneBuildDepends x dependencies) buildDepends
```

I traversed each leaf by folding over the `Section`s.

```haskell
stripNestedSection :: NestedSection -> Set T.DependencyName -> NestedSection
stripNestedSection nested dependencies = case nested of
  T.BuildDependsNestedSection numSpaces buildDepends -> T.BuildDependsNestedSection numSpaces (stripBuildDepends buildDepends dependencies)
  other -> other

stripNestedSections :: [NestedSection] -> Set T.DependencyName -> [NestedSection]
stripNestedSections nested dependencies = fmap (\x -> stripNestedSection x dependencies) nested

stripSection :: Section -> Set T.DependencyName -> Maybe T.CompilableName -> Section
stripSection section dependencies compilableMay = case (section, compilableMay) of
  (TargetSection T.CompilableTypeLibrary Nothing nested, Nothing) ->
    TargetSection T.CompilableTypeLibrary Nothing (stripNestedSections nested dependencies)
  (TargetSection typ (Just name) nested, Just T.Compilable {..}) | typ == compilableType && name == compilableName ->
    TargetSection typ (Just name) (stripNestedSections nested dependencies)
  (other, _) -> other

stripSections :: [Section] -> Set T.DependencyName -> Maybe T.Compilable -> [Section]
stripSections sections dependencies compilableMay =
  fmap (\x -> stripSection x dependencies compilableMay) sections
```

Simple and successful! Or so I thought...

## Beware `common` stanzas (`fix` to the rescue!)

Turns out, I had completely forgotten about `common` stanzas! For those that aren't familiar, here's [a primer](
https://vrom911.github.io/blog/common-stanzas). These are essentially placeholders for shared dependencies, language
pragmas, GHC options, etc which can then be included with a call to `import`. Similar to anchors in YAML, they meant my
previous strategy of folding over each individual section was not going to work anymore.

I had to change the AST to account for `common` top-level stanzas as well as `import` calls:

```haskell
newtype CommonName = CommonName { unCommonName :: Text }
  deriving (Eq, Ord, Show)

data NestedSection
  = BuildDependsNestedSection Int [String]
  | ImportNestedSection Int [String]
  | OtherNestedSection Int [String]
  deriving (Eq, Ord, Show)

data Section
  = TargetSection T.CompilableType (Maybe T.CompilableName) [NestedSection]
  | CommonSection CommonName [NestedSection]
  | OtherSection [String]
  deriving (Eq, Ord, Show)
```

Parsing was very easy to add, and similar to the above. The big challenge was figuring out how to resolve an `import`
call, especially since `common` stanzas can, themselves, have calls to `import`. In the end, I decided to return the
`import` set when folding over the sections and recurse while that set was non-empty.

I added a type for stripping from `common` stanzas, and updated every `strip`-type function to return `Set CommonName`
along with the updated portion it was responsible for:

```haskell
data StripTarget
  = StripTargetBaseLibrary
  -- ^ The base library
  | StripTargetCompilable T.Compilable
  -- ^ Any @library@, @executable@, @test-suite@, @benchmark@, etc stanza.
  | StripTargetCommonStanza (Set CommonName)
  -- ^ Any @common@ stanza matching the set.

stripNestedSection :: NestedSection -> Set T.DependencyName -> (NestedSection, Set CommonName)

stripNestedSections :: [NestedSection] -> Set T.DependencyName -> ([NestedSection], Set CommonName)

stripSection :: Section -> Set T.DependencyName -> StripTarget -> (Section, Set CommonName)
```

Then, I changed `stripSections` to recurse using `fix`:

```haskell
stripSections :: [Section] -> Set T.DependencyName -> Maybe T.Compilable -> [Section]
stripSections sections dependencies compilableMay =
  let run target = second mconcat . unzip . fmap (\x -> stripSection x dependencies target)
      firstTarget = maybe StripTargetBaseLibrary StripTargetCompilable compilableMay
      firstPass = run firstTarget sections
  in flip fix firstPass $ \recurse -> \case
       (final, none) | Set.null none -> final
       (next, common) -> recurse (run (StripTargetCommonStanza common) next)
```

In this way, I was able to ensure that nested calls to `import` would eventually strip out the relevant dependencies.
However, I knew that this strategy probably might not be perfect, so I had one final feature to add.

## Multiple Strategies (GADTs!)

I figured that since I had ended up writing two strategies for applying the changes, each with their own shortcomings,
I might as well allow the user to specify which one they wanted to use. The issues with doing that were:

1. They had different input and output types
1. Since there are multiple targets per file, the changes need to be applied incrementally
1. Other parts of the code didn't care _how_ the changes were being applied, but were gathering confirmation etc from
   the user and needed to know that _something_ was applying the changes

I ended up using a continuation-style GADT which would apply the changes incrementally and hide the implementation
details from the user.

```haskell
-- |Continuation GADT for applying changes to a cabal file.
data Apply (a :: T.ApplyStrategy) where
  ApplySafe :: FilePath -> GenericPackageDescription -> Endo GenericPackageDescription -> Apply 'T.ApplyStrategySafe
  ApplySmart :: FilePath -> [T.Section] -> Endo [T.Section] -> Apply 'T.ApplyStrategySmart

-- |Wrap 'Apply' in a data type so that it can be passed to functions without escaping the inner type.
data SomeApply = forall (a :: T.ApplyStrategy). SomeApply { unSomeApply :: Apply a }
```

I then wrote implementations for each and applied the incremental changes all at once when the file was overwritten:

```haskell
-- |Iterate on a cabal file by pruning one target at a time. Return whether the command-line call to @prune-juice@ should fail.
runApply :: SomeApply -> T.Package -> Set T.DependencyName -> Maybe T.Compilable -> T.ShouldApply -> IO (Bool, SomeApply)
runApply (SomeApply ap) T.Package {..} dependencies compilableMay = \case
  T.ShouldNotApply -> do
    printDependencies
    pure (True, applyNoop)
  T.ShouldApply -> do
    printDependencies
    confirm "Apply these changes? (Y/n)" >>= \case
      False -> pure (False, applyNoop)
      True -> pure (False, applyOnce)
  T.ShouldApplyNoVerify -> do
    printDependencies
    pure (False, applyOnce)
  where
    printDependencies =
      -- print the output
    applyNoop = case ap of
      ApplySafe x y z -> SomeApply $ ApplySafe x y z
      ApplySmart x y z -> SomeApply $ ApplySmart x y z
    applyOnce = case ap of
      ApplySafe x y z -> SomeApply $ ApplySafe x y $ z <> Endo (\w -> stripGenericPackageDescription w dependencies compilableMay)
      ApplySmart x y z -> SomeApply $ ApplySmart x y $ z <> Endo (\w -> stripSections w dependencies compilableMay)

-- |Write the series of changes to the cabal file.
writeApply :: SomeApply -> IO ()
writeApply (SomeApply ap) = case ap of
  ApplySafe fp description endo -> writeGenericPackageDescription fp (appEndo endo description)
  ApplySmart fp parsed endo -> writeCabalSections fp (appEndo endo parsed)
```

## Conclusion

In the end, I was extremely happy once again to be using Haskell for parsing, recursion, and dependent typing. In
addition, I was able to implement an oft-requested feature!
