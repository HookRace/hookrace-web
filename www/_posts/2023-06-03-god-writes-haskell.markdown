---
layout: post
title: "God writes Haskell"
tags: Programming Haskell
permalink: /blog/god-writes-haskell/
---

God famously does not play dice with the universe, but he seems to enjoy writing [Haskell](https://www.haskell.org/):
<!--more-->

1. Consider the [wave-particle duality](https://en.wikipedia.org/wiki/Wave%E2%80%93particle_duality) in quantum mechanics. Every particle behaves as a wave, as long as you haven't interacted with it. Thanks to Haskell's [lazy evaluation](https://wiki.haskell.org/Lazy_evaluation) values are also only evaluated once they are accessed (interacted with particles), and stay unevaluated thunks (waves) in the meantime.

2. Two particles can be [Quantum-entangled](https://en.wikipedia.org/wiki/Quantum_entanglement), so that their states depend on each other, even though the particles are seperated by any distance. In Haskell a value, whether it's evaluated yet or not, can also be shared and then used in a totally different location in the program without having to copy it. The value is even immutable, so that you can't change it from one location and thus influence the other. Similarly for entangled particles you can't manipulate one to change the state of the other particle, which might be far away and thus break the maximum [speed of information](https://en.wikipedia.org/wiki/Speed_of_light).

3. Since values are immutable they have to be cleaned up more often in Haskell than typically in imperative languages. [GHC](https://www.haskell.org/ghc/), the most commonly used Haskell compiler, allocates new data in a special area. Only after a supernova will the still-relevant data be ejected into the larger universe.

4. Haskell beginners often use lists instead of arrays. You can't do random access in a linked list, but only access the first element and then the rest of the list. The real world also doesn't allow you random access, you are limited by the speed of light and have to go from one location to the next.

5. Time also seems to be a linked list, not even doubly linked, since you can't go back after accessing the current element. Seems like an awkward bug.

6. Since the Haskell type system is so good at catching bugs, you often feel like you don't even need to write tests. This is unfortunately untrue, as the strange physical bugs of our universe demonstrate: The speed of light happens to stay the same, no matter what speed you move at.

7. There seems to be a long-term memory leak in Haskell, which is kind of easy to happen since you can have unevaluated thunks growing in size as long as they are still reachable. Unfortunately this memory leak will keep growing until it consumes the entire universe, which will then die of its [heat death](https://en.wikipedia.org/wiki/Heat_death_of_the_universe).

Discuss on [Hacker News](https://news.ycombinator.com/item?id=37414624)
