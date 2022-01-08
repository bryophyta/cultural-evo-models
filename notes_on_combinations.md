# Notes on generating combinations in JavaScript

- [The problem](#the-problem)
- [Option 1: Compact one-liner with `map()` functions](#option-1-compact-one-liner-with-map-functions)
- [Option 2. Naive generalisation of (1)](#option-2-naive-generalisation-of-1)
- [Option 3. Generating Cartesian products by stepping through a lexicographic ordering of options](#option-3-generating-cartesian-products-by-stepping-through-a-lexicographic-ordering-of-options)
- [Notes](#notes)

## The problem

An issue I've run into a few times is how to generate the set of unique combinations of a given set of elements.

For example, I've recently been working on modelling the social learning of signalling systems in Lewisian signalling games. The intuitive puzzle raised by Lewis is something like this.[^1] There are lots of situations where we benefit from having a common 'convention'. The classic example is which side of the road to drive on. Fundamentally, it doesn't matter that much *which* side we choose, so much as it matters that everyone using the road drives on the *same* side as each other, to avoid collisions. So how do we choose which side? An obvious option is to talk about it and make a decision together. But Lewis pointed out that language itself is 'conventional' in just the same way, and we can't have got together to discuss what words should mean before we had any language. So how could conventional meaning (or 'signals') come about in the first place?

Lewis's own answer seems to imply that agents in a signalling system have at least an implicit grasp of some pretty complex game theory. The learning algorithms I've been looking at recently, by contrast, don't presuppose any strategic understanding of the underlying game being played. Instead, it's been shown that relatively simple reinforcement processes can lead to the widespread adoption of effective and mutually-intelligible signalling systems within a given population, even when the population begins with a random distribution of strategies taken from all possible combinations of signals and responses.[^2] (Which is pretty exciting and would, I think, have come as a surprise to Lewis himself!)

In the model a 'strategy' maps a given input on to an action. A round is played when the first player responds to some state of the world by emitting a signal, and then the second player responds to that *signal* by taking some other action. They both win if the second action matches the initial state of the world. So ideally they want to land on a common 'language', whereby each maps one type of input onto one type of output, and both use the same mappings. But because we're letting the learning process do the work here, we can't assume that each agent starts with a one-one mapping from inputs to outputs. Strategies are picked at random, and this includes the strategy of taking the same action every time regardless of input.

In the basic game, there are two possible states of the world, two 'signalling' actions, and two 'response' actions. Agents are assigned a 'signalling strategy' and a 'response' strategy. I've found it found useful to represent the list of possible strategies as a list of dictionaries,[^3] e.g.:

```js
signallingStrategies = [
    {
        '1': 'A',
        '2': 'A'
    },
    {
        '1': 'A',
        '2': 'B'
    },
    {
        '1': 'B',
        '2': 'A'
    },
    {
        '1': 'B',
        '2': 'B'
    }
]
```

For the basic game, it's easy enough to write the possible combinations out by hand, but even adding one more state of the world starts to make it a bit cumbersome. So in order to model more complex signalling games, I needed a way to generate combinations (with repetition) automatically.[^4]

Once I started looking at this I remembered that I've been here before and, of course, forgotten half of it since last time. In Python I could just reach for `product()` from itertools. After getting through most of the material discussed here, I also found the [`just-cartesian-product` utility function](https://github.com/angus-c/just#just-cartesian-product) in angus-c's Just library for JavaScript, which looks pretty great. I think I'd be tempted to use that in the future if I'm generating these combinations regularly in a larger codebase.

But before finding that library, I tried out a couple of options which are pretty quick and easy (from a developer-experience perspective), and also stumbled across a pretty neat algorithm which I rather like. So I thought that I'd document some of what I tried so far, largely as a note-to-self for the future.


## Option 1. Compact one-liner with `map()` functions

I started off with a set of embedded `map()` and `flatMap()` functions. For example,

```js
const actions = ['A', 'B'];
actionCombinations = actions.flatMap(
    el1 => actions.flatMap(
        el2 => actions.map(
            el3 => [el1, el2, el3]
        )
    )
);
```

This gives us the following value for `actionCombinations`:

```js
[
  ["A", "A", "A"],
  ["A", "A", "B"],
  ["A", "B", "A"],
  ["A", "B", "B"],
  ["B", "A", "A"],
  ["B", "A", "B"],
  ["B", "B", "A"],
  ["B", "B", "B"]
]
```
This code is pleasantly compact, but it has some definite downsides. It's not flexible, given that you have to hard-code the right number of `flatMap()` calls. And like most embedded functions it's fun to write but could be hard to read at first glance. And I'm pretty sure that it'll use a lot more memory than it needs to, as the mapping functions create a bunch of intermediate arrays which all need to be held in memory before the outer function returns our output.

That said, if I just need short combinations of a few elements, it seems basically fine for the kind of personal project I'm working on at the moment.


## Option 2. Naive generalisation of (1)

Looking for something a bit more flexible, I thought I'd wrap the embedded map structure in a function which would allow me to specify the length of the combination arrays to output, and apply successive `flatMap()` calls using a `while` loop:

```js
const actions = ['A', 'B'];

function combinations(arr, outputLength) {
    let combos = arr.map(el => [el]);
    while (combos[0].length < outputLength) {
        combos = combos.flatMap(el1 => arr.map(el2 => [...el1, el2]));
    }
    return combos;
}

actionCombinations = combinations(actions, 3); 
// this gives the same result as our snippet above
```

This has more flexibility, and it should also have lower memory demands, as each mapping function is allowed to complete before the next one is started. Given this, I think it would realistically be fine for all of my current needs.

However, I was pretty confident that my naive implementation wouldn't be anything like the most efficient you could get. So I thought I'd search around a bit.


## Option 3. Generating Cartesian products by stepping through a lexicographic ordering of options

I thought I'd take a look at how products are generated in Python's itertools library. [The itertools docs](https://docs.python.org/3/library/itertools.html#itertools.product) outline a process which is reasonably similar to my second option above, albeit using list comprehension instead of `map()`:

```py
def product(*args, repeat=1):
    # product('ABCD', 'xy') --> Ax Ay Bx By Cx Cy Dx Dy
    # product(range(2), repeat=3) --> 000 001 010 011 100 101 110 111
    pools = [tuple(pool) for pool in args] * repeat
    result = [[]]
    for pool in pools:
        result = [x+[y] for x in result for y in pool]
    for prod in result:
        yield tuple(prod)
```

[The utility function from Just](https://github.com/angus-c/just/blob/master/packages/array-cartesian-product/index.js) also uses a similar structure. What all of these implementations have in common is that they all generate the whole list of combinations or products together, and they do so by building up lists of intermediate combinations which aren't themselves instances of the product we're looking to output:

```
Stage 1:
    ['a'], ['b']

Stage 2:
    ['a', 'a'], ['a', 'b'], ['b', 'a'], ['b', 'b']

Stage 3:
    ['a', 'a', 'a'], ['a', 'a', 'b'], ...
```
A drawback of this approach is that we don't get any useable results until we've generated *all* of the combinations. Sometimes this will be exactly what we want, but it will have limits for very large lists of combinations.

However, the itertools docs also mention that the *actual* implementation in the itertools library doesn't generate all of the intermediate lists of combinations in memory. I was curious as to how they implement this, so I thought I'd take a look at the itertools source code. With the caveat that my understanding of C is pretty limited, the key part of the source code is [the `product_next()` function](https://github.com/python/cpython/blob/b48ac6fe38b2fca9963b097c04cdecfc6083104e/Modules/itertoolsmodule.c#L2344).

Itertools' `product()` function is actually rather more general than my specific use-case, but to produce the sorts of combinations that I'm after you can pass it a single array of elements, and then provide the length of the desired combinations as the `repeat` parameter:

```py
import itertools


p = itertools.product(['a', 'b'], repeat=2)
next(p)
# ('a', 'a', 'a')
next(p)
# ('a', 'a', 'b')
```
If I've understood it correctly, the itertools *source* implementation relies on a function with three key properties:

1. Given only the previous product (and some information about the original inputs), it is able to generate 'the next' product, without needing to know about anything that came earlier.
2. Given the right starting input, we can know that repeated applications of this 'next' procedure will not miss any product combinations out, or repeat them.
3. We can know when we've reached the end.

These properties, together, mean that we can generate each possible combination, without having to generate all of them at the same time.

The particular procedure used in the itertools implementation starts with a list of zeros which is as long as the product we're aiming to output. These zeros are used as indices to build a new list by selecting from each of our 'pools' (the input lists that we're generating the product of). We can imagine the list of indices as being a counter, like on a digital clock, which is counting in a base of the length of the corresponding pool.

So if there are three pools and each of the pools has `n` elements, the indices counter will start at `[0, 0, 0]`, then go to `[0, 0, 1]` and count up to `[n, n, n]`. At each step there's a simple procedure to get from the previous combination of indices to the next: you increment the rightmost index which is below n, or if it reaches n then you reset it to zero and increment the index to the left of it, and keep going until you reach an index which is less than n. 

This is definitely not necessary for my current applications, but it's pretty neat! And it means we can take proper advantage of the generator structure to avoid generating later products until we need them, which will be helpful for products of large lists. I've changed the structure a bit from the itertools source implementation, but here's my current reconstruction as a JavaScript [generator function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) and a `nextProduct()` function:

```js
function nextProduct(indices, pools) {
    let i;
    for (i = indices.length - 1; i >= 0; i--) {
        indices[i]++;
        // added modulo calculation to avoid duplicating pools
        if (indices[i] == pools[i % pools.length].length) {
            indices[i] = 0;
        } else {
            break;
        }
    }
    if (i < 0) {
        return;
    }
    return indices;
}

function* product(repeat, ...arrs) {
    /* you can't currently combine rest params and optional named params in js :( 
    so we have to rely on parameter order so that we can handle an indefinite 
    number of arrays */
    const pools = [...arrs];
    const numPools = pools.length;
    let result = [];
    let indices = [];
    // initialise indices as an array of zeros 
    for (let i = 0; i < numPools * repeat; i++) {
        indices.push(0);
    }
    while (indices != undefined) {
        result = indices.map((value, index) => pools[index % numPools][value]);
        //use indices array to pull values from the input pools
        yield result;
        /* get next result (productNext function returns void once it's exhausted
        the combinations, which will exit us from the loop) */
        indices = nextProduct(indices, pools);
    }
    // when there are no more values, close the generator
    return;
}
```


---
## Notes

[^1]: The classic work here is David Lewis, (1969) *Convention*, which draws on Thomas Schelling's (1960) *The Strategy of Conflict*. There is a really excellent critique of Lewis's own positive account of convention in Ruth Millikan's (2005) 'Language Conventions Made Simple'.
[^2]: The code I'm working on at the moment draws from the model in Kevin Zollman's (2005) 'Talking to Neighbors: The Evolution of Regional Meaning', but a similarly 'simple' learning algorithm is also found in Argiento et al. (2009) 'Learning to Signal: Analysis of a micro-level reinforcement model', which I've modelled [here](https://github.com/bryophyta/social-learning-models/blob/main/signal_learning_game.ipynb).
[^3]: The list format makes it easy to assign a strategy to an agent, e.g. `agent.signallingStrategy = signallingStrategies[0]`. Storing the strategies themselves as dictionaries then makes it easy to look up an agent's response to a given state of the world: if the world is in state `1`, the agent's signalling action will be `agent.signallingStrategy['1']`, i.e. `A`.
[^4]: The combinations we're generating here are to fill in the 'values' of the key-value pairs in the strategy dictionaries. The keys are just repeated, so once we've got the arrays of values we can 'zip' each of them with the list of keys using a helper function.