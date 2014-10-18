A long Factor tutorial
======================

Factor is a mature, high level programming language, but starting with it can be rather taxing, as it follows a paradigm quite far from most mainstream languages. Still, even if it is a rather niche language, it is rather mature, running in its own optimized VM, usually reaching top performance for a dinamically typed language. It features a flexible object system, a FFI with C, and asynchronous I/O, much like node, but with a much simpler model for cooperative multithreading.

In this tutorial, we assume that you have downloaded a copy of Factor and that you are following along with the examples in the Listener.

Concatenative languages
-----------------------

Factor is a *concatenative* programming language in the spirit of Forth. What does this even mean? In a *concatenative* language, there is essentially only one operation, which is function composition. Since function composition is so pervasive, it is usually implicit, and functions can be literally juxtaposed in order to compose them. So if `f` and `g` are two functions, their composition is just `f g` (usually, functions are read from left to right, so this means first execute `f`, then `g`, unlike in mathematical notation).

This requires some explanation, since functions will usually have multiple inputs and outputs, and it is not always the case that the output of `f` matches the input of `g`. For this to work, `f` and `g` have essentially to be functions which take and return the whole state of the world.

There are various ways this global state can be encoded. The most naive would use a hashmap mapping variable names to their values. This turns out to be too flexible: if every function can access randomly any piece of global state, there is little control on what functions can do, little encapsulation and ultimately programs become an unstructured mess of routines mutating global variables.

It turns out in practice that a nice way is to encode the state of the world as a stack. Functions can only refer to the topmost element of the stack, so that elements below it are effectively out of scope. If a few primitives are given to manipulate a few elements on the stack (for instance `swap`, that swap the two elements on top), then it becomes possible to refer to values more down the stack, but the farthest the position, the hardest it becomes to refer to it.

So functions are encouraged to stay small and only refer to the top two or three elements. In a sense, there is no more a distinction between local and global variables, but values can be more or less local depending on their distance from the top of the stack.

Notice that if every function takes the state of the whole world and returns the next state, its input is never used anymore. So, even if it is more convenient to think of pure functions from stacks to stacks, the semantics of the language can be implemented more efficiently by mutating a fixed stack.

This leaves factor in a strange position, whereby it is both extremely functional - only allowing to compose simpler functions into more complex ones - and largely imperative - describing operations on a mutable stack.

Playing with the stack
----------------------

Let us start looking what Factor actually feels like. Our first words will be literals, like `3`, `12.58` or `"chuck norris"`. Literals can be thought as functions that push themselves on the stack. Try writing `5` in the listener and then press enter to confirm. You will see that the stack, initially empty, now looks like

    5

You can enter more that one number, separated by spaces, like `7 3 1`, and get

    5
    7
    3
    1

(the interface shows the top of the stack on the bottom). What about operations? If you write `+`, you will run the `+` function, which pops the two topmost elements and pushes their sum, leaving us with

    5
    7
    4

You can put, as before, more inputs in a single line, so for instance `- *` will leave the single number `15` on the stack (do you see why?). The function `.` (a dot) prints this item, while popping it out of the stack, leaving the stack empty.

If we write everything on one line, our program so far looks like

    5 7 3 1 + - * .

which shows the peculiar way of doing arithmetics by putting the arguments first and the operator last - a convention which is called Reverse Polish Notation (RPN). Notice that this requires no parenthesis, unlike the Lisp convention where the operator comes first, and no precedence rules, unlike most other systems. For instance in any Lisp, the same computation would be written like

    (* 5 (- 7 (+ 3 1)))

Also notice that we have been able to split our computation rather arbitrarily and that each subpiece of our line made sense in itself.

Defining our first word
-----------------------

We will now define our first function. Factor has a slightly odd naming: since functions are just written from left to right, they are simply called **words**, and this is what we will do from now on. Modules in Factor define words in terms of previous words and are then called **vocabularies**.

We will want to compute the factorial. To start with a concrete example, we compute the factorial of `10`, so we start by writing `10` on the stack. Now, the factorial is the product of the numbers from `1` to `10`, so we should produce such a list of numbers first. Tokenization is trivial in Factor, as words are always space separated, and this allows you to use any combination of non-whitespace characters as an identifier.

The function to produce a range is reasonably called `[a,b]`. In our case one of the extremes is just `1`, so we can use the simpler word `[1,b]` instead. If you write that in the listener, you will be prompted with a choice, because the name `[1,b]` is not imported by default. Factor is able to suggest to import the `math.ranges` vocabulary, so choose that option and proceed.

You should now have on your stack a rather opaque structure which looks like

    T{ range f 1 10 1 }

This is because our range functions are lazy. To confirm that we actually have created the list of numbers from `1` to `10`, we convert it into an array using the word `>array`. Your stack should now look like

    { 1 2 3 4 5 6 7 8 9 10 }

which is promising.

Next, we want to take the product of those numbers. In many languages, this could be done with a function called reduce or fold. Let us look for one. Pressing `F1` will open a contextual help, where you can search for `reduce`. It turns out that `reduce` is actually the word we are looking for, but at this point it may not be obvious how to use it.

Try writing `1 [ * ] reduce` and look at the output: it is indeed the factorial of `10`. Now, `reduce` usually takes three arguments: a sequence (and we had one), a starting point (and this is the `1`) and a binary operation. This must certainly be the `*`, but what about those square brackets around it?

If we had written just `*`, Factor would have tried to apply multiplication to the topmost two elements, which is not what we wanted. What we need is a way to mention a word without applying it. Keeping our textual metaphor, this mechanism is called **quotation**. To quote one or more words, you just surround them by `[` and `]` (leaving spaces). What you get is akin to an anonymous function in other languages.

Let us `drop` the result to empty the stack, and try writing what we have done so far in a single shot `10 [1,b] 1 [ * ] reduce`. This will output `3628800` as expected.

We now want to define a word that can be called whenever we want. We will call our word `!` as it is customary. To define it, we need to use the word `:`. Then we put the name of the word being defined, the stack effects and finally the body, ending with the `;` word:

    : ! ( n -- n! ) [1,b] 1 [ * ] reduce ;

What are the stack effects? These are the part like `( n -- n! )` in our case. They have no effect on the function, but allow you to name your inputs and outputs for documenting purposes. You can use any identifier to name them, and Factor will only make a consistency check that the number of inputs and outputs agrees with what the body does.

If you try to write

    : ! ( m n -- n! ) [1,b] 1 [ * ] reduce ;

Factor will signal an error that the number of inputs is not consistent. To restore the previous correct definition press `Ctrl+P` two times to get back to the previous input and the enter it.

We can think at the stack effects in definitions both as a documenting tool and as a simple type system, which nevertheless does catch a few errors.

In any case, you have succesfully defined your first word: if you write `10 !` in the listener you can prove it.

Notice that the `1 [ * ] reduce` part of the definition sort of makes sense on its own, being the product of a sequence. The nice thing about a concatenative language is that we can just factor this part out and write

    : prod ( {x1,...,xn} -- x1*...*xn ) 1 [ * ] reduce ;
    : ! ( n -- n! ) [1,b] prod ;

Our definitions have become simpler and there was no need to pass parameters, rename local variables or anything that would have been necessary to factor out a part of the definition in a different language.

Parsing words
-------------

If you have paid attention until now, you will realized that I have lied to you. I have said that each word acts on the stack in order, but there a few words like `[`, `]`, `:` and `;` that certainly seems not to abide to this rule.

In fact these are **parsing words** and behave differently from simpler words like `5`, `[1,b]` or `drop`. We will get into more detail when we talk about metaprogramming, but for now it is enough to know that parsing words are special.

They are not defined using `:`, but using `SYNTAX:` instead. When a parsing words is encountered, it can interact with the parser using a well-defined API to influence how successive words are parsed. For instance `:` asks the next tokens from the parsers until `;` is found and tried to compile that stream of tokens into a word definition.

A common use of parsing words is to define literals. For instance, `{` is a parsing word that starts an array definition and is terminated by `}`. Everything in-between is part of the array. An example of array that we have seen before is `{ 1 2 3 4 5 6 7 8 9 10 }`.

There are also literals for hashmaps, like `H{ { "Perl" "Larry Wall" } { "Factor" "Slava Pestov" } { "Scala" "Martin Odersky" } }`, or byte arrays, like `B{ 1 14 18 23 }`.

Other uses of parsing word include the module system, the object oriented features of Factor, enums, memoized functions, privacy modifiers and more. In theory, even `SYNTAX:` can be defined in terms of itself, although of course the system has to be bootstrapped somehow.

Stack shuffling
---------------

Now that you know the basics of Factor, you may want to start assembling more complex words. This sometimes may require to use variables that are not on top of the stack, or to use variables more that once. There are a few words that can be used to this effect. I will mention the know, since you'd better be aware of them, but warn you that code using many of those words can quickly become hard to write and harder to read. It requires mentally simulating moving values on a stack, which is not a natural way to program. We will see a much more effective way to handle most needs in next section.

I will just write the most common shuffling words together with their effect on the stack. Feel free to try them in the listener, and explore the online help to find out more.

    dup ( x -- x x )
    drop ( x -- )
    swap ( x y -- y x )
    over ( x y -- x y x )
    dupd ( x y -- x x y )
    swapd ( x y z -- y x z )
    nip ( x y -- y )
    rot ( x y z -- y z x )
    -rot ( x y z -- z x y )
    2dup ( x y -- x y x y )

Combinators
-----------

Although the words mentioned in the previous paragraph are occasionally useful (especially `dup`, `drop` and `swap`), one should aim to write code that does as little stack shuffling as possible. This requires a certain practice in putting the function arguments in the right order from the start.

Nevertheless, there are certain patterns of use that are better abstracted away into their own words. For instance, say we want to define a word to determine whether a given number `n` is prime. A simple algorithm would be to test each number from `2` to the square root of `n` and see whether it is a divisor of `n`.

This immediately shows that `n` is used in two places: as an upper bound for the sequence, and as the number to test for division. The word `bi` applies two different quotations to an element on the stack, and this is precisely what we need. For instance `5 [ 2 * ] [ 3 + ] bi` yields

    10
    8

To continue with our example, will need a word to test divisibility, and a quick search in the only help shows that `divisor?` is what we want. We will also need a way to make a range starting from `2`, which we can define like

    : [2,b] ( n -- {2,...,n} ) 2 swap [a,b] ; inline

What's up with that `inline` word? This is one of the modifiers we can use after defining a word, another one being `recursive`. This will allow us to have the definition of the word inlined wherever it is used, rather than incurring a function call.

It may also help to have the arguments for divisibility in the other direction, so we define

    : multiple? ( a b -- ? ) swap divisor? ; inline

Now, producing the range of numbers from `2` to the square root of `n` is easy: `sqrt floor [2,b]` (`floor` is not even necessary, as `[a,b]` works for non-integer bounds). What about the second function?

We must produce a function that test for being a divisor of `n` - in other words we need to partially apply the word `multiple?`. This can be done with the word `curry`, like this: `[ multiple? ] curry`.

Finally, once we have the range and the test function on the stack, we can test whether any element satisfies the divisibility with `find` and convert it to a boolean (`t` or `f`) with `>boolean`. Since `find` returns both the index and the element, we will want to define another word

    : exists? ( seq pred: ( elt -- ? ) -- ? ) find nip >boolean ; inline

Notice the use of nested stack effects. Our full definition looks like

    : prime? ( n -- ? ) [ sqrt [2,b] ] [ [ multiple? ] curry ] bi exists? not ;

Altough the definition is slightly complicated, the stack shuffling is minimal and limited to the small helper functions, which are much simpler to reasong about than `prime?`.

Many more combinators exists other than `bi` (and its relative `tri`), and you should become acquainted at least with `bi*` and `bi@`.

Vocabularies and tests
----------------------

The object system and protocols
-------------------------------

The listener
------------

Metaprogramming
---------------

macros, parsing words

When the stack is not enough
----------------------------

locals, fried quotations, global variables

Input/Output
------------

Deploying programs
------------------

Multithreading and processes
----------------------------

Servers and Furnace
-------------------