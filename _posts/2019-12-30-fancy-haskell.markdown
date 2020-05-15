---
title: Fancy Haskell
---

Inspired by posts from [@parsonsmatt](https://www.parsonsmatt.org/2019/12/26/write_junior_code.html)
and [@cvlad](https://cvlad.info/junior-developers/).

I have some observations and opinions about "Fancy Haskell" from my time working as a Haskell
engineer.

## Observations

### Some Haskell is better than no Haskell

I think we can all agree on this (otherwise you probably wouldn't be reading this). We'd all rather
be writing Haskell than pretty much anything else. We relish the safe feeling when code compiles
and we're confident it'll do what we ask it to do.

### Proficiency in [technology] leads to using [technology] in a more advanced way

It's true for anything! For Haskell specifically, as we level up we want to use the fancier bits.
We all want to experiment and learn for learning's sake, and that's one of the reasons we wanted to
learn Haskell in the first place! So I think it's a very natural tendency.

However, we don't have enough new blood coming in so people have to level up individually and never
as a group, which makes the barrier to entry higher. The tendency to always want to write fancier
Haskell plus the lack of junior Haskellers leads to an alienation of newcomers.

## Opinions

### Fancy Haskell has benefits that can't be overlooked

Fancy Haskell can always find its way into a codebase for junior developers so long as the reasons
for doing so are clear. As examples, I have two libraries or frameworks that I often see criticized
but which can serve important roles.

We simply don't have good nested record update syntax in Haskell, and `lens` fills that gap. It
provides an API for both simple and complex operations that can have enormous benefit. The
compilation errors involved with lenses are a big turnoff to using them regardless of how hard (or
easy) it is to learn to use them. Given those tradeoffs they're still immensely useful and
important for beginners to understand, especially because plenty of library APIs are based on
lenses.

The representation of an API as a type is largely helpful for documenting and generating client
libraries in other languages. `servant` combined with `servant-swagger` do this beautifully, and it
wouldn't be possible in `yesod` or other more beginner-friendly API frameworks without a lot of
manual intervention. While dealing with confusing compilation errors at the type level can be a
massive headache, in this case there are clear correctness and usability benefits arising from
using `servant`.

While fancy Haskell can lead to the alienation of the beginner demographic, some libraries or
concepts are so useful that an adequate alternative doesn't exist. We can still be beginner
friendly within these boundaries, and I leave the reasoning to the next section.

### Good engineers don't make themselves irreplaceable

In the business world you're never going to make an effective product unless you understand the
concerns of your stakeholders. Within the scope of an engineering team, it could be said that our
product as engineers is our code and our stakeholders are those in charge of hiring as well as
present and future Haskell engineers. Emphasis on _future_ engineer, like one who hasn't learned or
mastered Haskell yet. _That's_ our stakeholder, not Ed Kmett.

Put another way, everyone who's worked at an engineering company knows some infamous former
employee who wrote a bunch of bad code and whose name, when mentioned, makes everyone groan. Don't
be that engineer by writing fancy code for the sake of writing fancy code.

## Conclusion

Fancy Haskell code has a time and a place. Make sure to clearly identify the tradeoffs when
reaching for anything fancy. Incorporate those tradeoffs into an ideal manifestation of simple
code, and then interate until code represents that ideal. Always document and test.
