---
title: Haskell Database Implementation - Part 1, Growing a Tree
---

Having started my career as a software developer in 2013 with an undergraduate degree in Mathematics, I never got the
chance to take some of the higher-level courses on subjects like compilers, programming languages, or databases. In 2020
I set out to correct some of the gaps in my knowledge, and one of the ways I did so was by writing my own database, in
Haskell, from scratch. This series of posts chronicles the decisions I made along the way.

If you'd like to skip ahead and read the code, it's [here](https://github.com/dfithian/dfdb).

## What I was interested in

In starting this project, I was most interested in (a) how the data was stored, (b) transactionality, and (c) how
indexes work.

I found out pretty quickly that data is stored in a B-tree (see below) in most circumstances. Since there are many
how-tos out there already I decided I was more interested in index creation and management. Nevertheless, in order to
index my data I'd still need a tree representation, and I hadn't written one in a while, so I made a halfhearted
attempt.

## Efficient searches use trees

In any most implementations you'd probably want to use a [B Tree](https://en.wikipedia.org/wiki/B-tree) as the
underlying data structure. The TL;DR is that B-trees are good at managing access to files while keeping only small
chunks of those files in memory at any time.

I ended up settling on using a [Red Black Tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree). It has many of
the same properties as the B Tree (self balancing, efficient), and is simpler since it doesn't need access to a file
system. Since this was mostly a project for my own learning, I figured I could just write the tree to disk whenever I
wished.

### Tree definition

I won't go into the implementation specifics, since the algorithm is copied from the Wikipedia article, but I'll quickly
go over the data declaration so that later discussion makes sense.

```haskell
data Color = Red | Black
  deriving (Eq, Ord, Generic)

data TreeMap a b = Node !a !b Color !(TreeMap a b) !(TreeMap a b) | Nil
  deriving (Eq, Ord, Generic)

type Tree a = TreeMap a ()
```

A `TreeMap a b` node either contains a key `a`, a value `b`, a color, and left/right branches, or is a leaf node (`Nil`).

For testing purposes, define the `Tree` type to be unitary values, since we really care about finding and inserting keys
efficiently, and we don't really care about the data (other than that it's there).

### Property testing

#### Definitions

Any recursive code always needs a good set of property tests. Luckily for us, Red Black Trees already have a list of
laws (source: [Wikipedia](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree#Properties)):

> In addition to the requirements imposed on a binary search tree the following must be satisfied by a red–black tree:
> 1. Each node is either red or black.
> 1. The root is black. This rule is sometimes omitted. Since the root can always be changed from red to black, but not
>    necessarily vice versa, this rule has little effect on analysis.
> 1. All leaves (NIL) are black.
> 1. If a node is red, then both its children are black.
> 1. Every path from a given node to any of its descendant NIL nodes goes through the same number of black nodes.

#### Laws

Right out the gate, the first three properties listed above are either correct by construction or trivial to test. We
defined a `Color` type, with only two constructors, so nodes of the tree can only inhabit one of those two colors.
Testing whether the root node is black isn't very interesting, since it's easy to set it to black after any operation.
Finally, by omitting a color in the definition of a leaf node we can ensure that its color is black.

##### Binary Search Law

Our first property test is to ensure that any tree we generate adheres to the binary search property.

```haskell
checkBinary :: (HasCallStack, Ord a) => Tree a -> Bool
checkBinary = \ case
  Tree.Nil -> True
  Tree.Node x _ _ tl tr -> all (\ l -> l < x) (Tree.setToList tl) && all (\ r -> r > x) (Tree.setToList tr)
```

Here, we check that every element on the left side of the root is less than the root, and vice-versa for the right side.
Since we will be running this test for hundreds of test cases, we don't care that we're only checking the root node, as
we'll be generating many different depths.

##### Children of Red Nodes

Our second property test is to ensure that children of a red node are black.

```haskell
checkColor :: HasCallStack => Tree a -> Bool
checkColor = \ case
  Tree.Nil -> True
  Tree.Node _ _ Tree.Red (Tree.Node _ _ Tree.Red _ _) _ -> False
  Tree.Node _ _ Tree.Red _ (Tree.Node _ _ Tree.Red _ _) -> False
  Tree.Node _ _ _ tl tr -> checkColor tl && checkColor tr
```

Pretty straighforward; if we unpack a red node, and it has a red child, fail the test case.

##### Path Counting

Finally, we want to test that the distance from the root to all leaves passes through the same number of black nodes.

```haskell
checkPathCount :: HasCallStack => Tree a -> Bool
checkPathCount t =
  let go = \ case
        Tree.Nil -> (0 :: Int, True)
        Tree.Node _ _ c tl tr ->
          let (il, bl) = go tl
              (ir, br) = go tr
              j = if c == Tree.Black then 1 + il else il
          in (j, bl && br && il == ir)
  in snd $ go t
```

Pass a counter through a traversal of the tree. For each subtree, verify that its subtrees satisfy the predicate, and
increment the counter if the root node of that subtree is black. Warning, you may have to squint.

#### Writing tests

We want to ensure that any mutation we apply to a tree results in a tree that satisfies the laws we set out above. For
simplicity, I have outlined only tree creation and element insertion here, but we would want to property test each
operation.

```haskell
genTree :: Ord a => Gen a -> Gen (Tree a)
genTree gen = Tree.setFromList <$> listOf gen

genIntTree :: Gen (Tree Int)
genIntTree = genTree arbitrary

runTests :: (HasCallStack, Ord a, Show b) => String -> Gen b -> (b -> Tree a) -> Spec
runTests d gen f = do
  prop (d <> ": binary") $ forAll gen $ checkBinary . f
  prop (d <> ": color") $ forAll gen $ checkColor . f
  prop (d <> ": path count") $ forAll gen $ checkPathCount . f

spec :: Spec
spec = describe "Tree" $ do
  runTests "generation" genIntTree id
  runTests "insertion" ((,) <$> arbitrary <*> genIntTree) $ uncurry Tree.insertSet
```

And that's it!

## To be continued

Thanks for reading! Part 2, on the DSL and transactions, is [here]({% post_url 2021-02-18-database-implementation-part-2 %}).

As always, I'd love to hear anything I've mixed up, especially in this case from database experts. Find me on fpchat
(`@dfithian`) or reddit (`/u/dfith`).
