---
title: Building a Spell Checker
description: Creating a command line Spell Checker application.
date: 2019-01-22 00:44:00
---

In this article, we will implement a command line Spell Checker in Haskell and cover the basics over how to suggest corrections. We will see that it can be done using different techniques and how those choices critically impact on the suggestion accuracy and performance.

## String distance algorithms

First of all, we need a way to tell how similar one string is from another, for example, using the Levenshtein distance algorithm `cat` and `rat` distance is 1. Several algorithms approach this problem in different ways:

- Hamming distance
- Levenshtein distance
- Damerau-Levenshtein distance
- Optimal String Alignment
- Longest Common Substring distance
- q-gram distance
- Cosine distance
- Jaccard distance
- Jaro distance
- Jaro-Winkler distance

For this project, we will use the Levenshtein distance, that is probably one of the most famous and straightforward algorithms in this category. The Levenshtein distance is the minimum number of single-character edits (i.e., insertions, deletions or substitutions) required to change one word into the other. So, in the previous example to transform `cat` in `rat` we only need to substitute the first character from `c` to `r`, and that is why the distance is 1.

## Searching for near-matches Strings

Now that we can metrify how distant one string is from another, we need a way to go over our dictionary and suggest possible corrections.

The problem is that brute forcing a dictionary with a million of words to find near-matches can take some time, just imagine if we had to calculate the distance between two different words a million times and then filter the ones that has the distance below or equal N.

So to escape from a linear complexity, we should keep our dictionary in the smartest form as possible. To solve that we have some options:

- BK-tree
- Norving
- LinSpell
- SymSpell

We are going to use the BK-tree, as it is efficient enough for our purpose. BK-Trees or Burkhard-Keller Trees is a tree-based data structure commonly used for quickly finding near-matches to a string. As many articles already cover this topic, I will recommend one and left here my implementation.

```haskell
module BkTree 
    ( insert
    , search
    , BkTree.lookup
    , Node (Empty, Node)
    , Distance (metric)
    ) where

import qualified Data.Map as M
import Data.Maybe (mapMaybe)

class Distance a where
    metric :: a -> a -> Int

data Node a = Empty
            | Node { value :: a
                   , children :: M.Map Int (Node a)} deriving (Show, Eq, Read)

insert :: (Distance a) => a -> Node a -> Node a
insert str Empty = Node str M.empty
insert str (Node value children) = 
    Node value newChildren
    where
        distance         = metric value str
        sameDistanceNode = M.lookup distance children
        newChildren      = case sameDistanceNode of
            Nothing -> M.insert distance (Node str M.empty) children
            Just node -> M.insert distance (insert str node) children

search' :: (Distance a) => a -> Int -> [Node a] -> [a] -> [a]
search' _ _ [] acc = acc
search' input n (Node value children : tail) acc 
    | length acc == 3 = acc -- limiting to 3 suggestions
    | otherwise =
        let distance   = metric input value
            inRange    = mapMaybe (`M.lookup` children) [(distance-n)..(distance+n)]
            validNodes = inRange ++ tail
        in if distance <= n then
            search' input n validNodes (value:acc)
            else search' input n validNodes acc

search :: (Distance a) => a -> Int -> Node a -> [a]
search input n node = search' input n [node] []

lookup :: (Distance a) => a -> Node a -> Bool
lookup input (Node value children) =
    let distance = metric input value
    in (distance == 0) || case M.lookup distance children of
                            Nothing -> False
                            Just x -> BkTree.lookup input x
```

_Obs: Try to insert the elements in the tree ordered by word relevance, in that way you will get better accuracy as these words will be nearest to the root._

## Loading the dictionary

Now the missing part is the dictionary; you can pick one from the LibreOffice project [here](https://cgit.freedesktop.org/libreoffice/dictionaries/plain/). With the dictionary file in hands, we only need to load it in memory and build the BK-tree.

```haskell
module Main where

import qualified BkTree as BK
import qualified Data.Text as T
import qualified Data.Text.IO as TIO

fileLines :: FilePath -> IO [T.Text]
fileLines p = do 
    content <- TIO.readFile p
    return $ T.splitOn (T.pack "\n") content

bkTreeFromDictionary :: [T.Text] -> BK.Node T.Text
bkTreeFromDictionary = foldl (flip BK.insert) BK.Empty

main :: IO ()
main = do
  bkTree <- fileLines "dictionary" >>= \x -> return $ bkTreeFromDictionary x
```

## Finding typos and suggesting words

You could argue that for precise matches creating a HashSet from the dictionary could have a better performance than trying to find it in the BK-tree, and that is true, but the time that takes to build a HashSet with almost a million of Strings is significant, so for a command line spellchecker is just  faster to search in the BK-tree for a node with distance of 0.

```haskell
module Main where

import System.Environment
import Data.Char
import Data.Text.Metrics (levenshtein)
import qualified BkTree as BK
import qualified Data.Text as T
import qualified Data.Text.IO as TIO

spellCheck :: [T.Text] -> BK.Node T.Text -> [T.Text]
spellCheck words tree = filter (\x -> not $ BK.lookup x tree) words

guess :: BK.Node T.Text -> T.Text -> (T.Text, [T.Text])
guess bktree miss = (miss, BK.search miss 1 bktree)

bkTreeFromDictionary :: [T.Text] -> BK.Node T.Text
bkTreeFromDictionary = foldl (flip BK.insert) BK.Empty

instance BK.Distance T.Text where
    metric = levenshtein

main :: IO ()
main = do
  bkTree <- fileLines "dictionary" >>= \x -> return $ bkTreeFromDictionary x
  content <- TIO.getContents >>= \x -> return $ T.words x

  print $ map (guess tree) (spellCheck content tree)
```

In the code above we create a list of words from the input, and then for each one, we check if it exists in the dictionary. For the words that could not be found we search three possible corrections that are at a maximum distance of 1.

Note that we are using the Levenshtein function from the [text-metrics](https://hackage.haskell.org/package/text-metrics)  package.

## Running the application

Now we should be able to run our command line application. Follows some examples:

- Output

```bash
» echo "Testando o correror ortografico." | spell-checker
[("correror",["corredor","corretor"]),("ortografico",["ortográfico"])]
```
- Total Time

```bash
» wc input.txt
46  1776 11871 input.txt

» time cat input.txt | spell-checker
cat input.txt  0,00s user 0,00s system 79% cpu 0,001 total
spell-checker-exe  11,38s user 4,23s system 184% cpu 8,472 total
```

It took 8.472s to build the BK-Tree with 985563 nodes and process a file with 1776 words. Note that the most time consuming part of this process is to build the Tree.

_Obs: In Haskell, the expressions are not evaluated when they are bound. Because of that, the execution time drops considerably when analyzing smaller texts. For example, it should not take much more than one second to check a single word as just a tiny part of the Tree will be built in the process._

## Possible Optimizations

- Save the in-memory BK-Tree to disk in a binary format. Avoiding transversing the Tree and calculating the distance between nodes at every program startup.

- Use SymSpell instead of BK-Tree as it claims to be 1,870 times faster. Just note that it seems to take more time to build the SymSpell Tree, and may not worth the faster lookup as the startup time is the bottleneck for our command line spellchecker.

## Wrap up
