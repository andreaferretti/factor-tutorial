A long Factor tutorial
======================

[Factor](http://factorcode.org) is a mature, dynamically typed language based on the concatenative paradigm. Getting started with Factor can be rather daunting, as it follows a paradigm quite far from most mainstream languages. This tutorial will guide you through the basics so that you will be able to appreciate its simplicity and power. I will assume that you are familiar with some functional language, as I will mention freely concepts like folding, higher-order functions, or currying.

Even if Factor is a rather niche language, it is mature and features a comprehensive standard library covering tasks from JSON serialization to socket programming or HTML templating. It runs in its own optimized VM, usually reaching top performance for a dynamically typed language. It also has a flexible object system, a FFI with C, and asynchronous I/O - much like node, but with a much simpler model for cooperative multithreading.

You may wonder why should you care enough to read this long tutorial. Factor has a few significant advantages over other languages, most arising from the fact that it has essentially no syntax:

* refactoring is very easy, leading to short and meaningful definitions;
* it is extremely succinct, letting the programmer concentrate on what has to be done instead of boilerplate;
* it has powerful metaprogramming capabilities, exceeding those of LISPs;
* it is ideal to embed DSLs;
* it integrates easily with powerful tools.

In this tutorial, we assume that you have [downloaded a copy of Factor](http://factorcode.org) and that you are following along with the examples in the listener (the Factor REPL). The first section gives some motivation for the rather peculiar model of computation, but feel free to skip it if you want to get your feets wet and return to it after some practice.

I will also assume that you are using some distribution of Linux, but everything should work the same on other systems, provided you adjust the paths in the examples.

Concatenative languages
-----------------------

Factor is a *concatenative* programming language in the spirit of Forth. What does this even mean?

Imagine a world where every value is a function, and the only operation allowed is function composition. Since function composition is so pervasive, it is usually implicit, and functions can be literally juxtaposed in order to compose them. So if `f` and `g` are two functions, their composition is just `f g` (usually, functions are read from left to right, so this means first execute `f`, then `g`, unlike in mathematical notation).

This requires some explanation, since functions will usually have multiple inputs and outputs, and it is not always the case that the output of `f` matches the input of `g`. For instance, `g` may need access to values computed by earlier functions. But the only thing that `g` can see is the output of `f`, so this is the whole state of the world, as far as `g` is concerned. Hence, to make this work, functions have to thread the global state, passing it to each other.

There are various ways this global state can be encoded. The most naive would use a hashmap that maps variable names to their values. This turns out to be too flexible:  if every function can access randomly any piece of global state, there is little control on what functions can do, little encapsulation, and ultimately programs become an unstructured mess of routines mutating global variables.

It works well in practice to represent the state of the world as a stack. Functions can only refer to the topmost element of the stack, so that elements below it are effectively out of scope. If a few primitives are given to manipulate a few elements on the stack (e.g., `swap`, that exchanges the two elements on top), then it becomes possible to refer to values more down the stack, but the farthest the position, the hardest it becomes to refer to it.

So, functions are encouraged to stay small and only refer to the top two or three elements. In a sense, there is no more a distinction between local and global variables, but values can be more or less local depending on their distance from the top of the stack.

Notice that if every function takes the state of the whole world and returns the next state, its input is never used anymore. So, even if it is more convenient to think of pure functions from stacks to stacks, the semantics of the language can be implemented more efficiently by mutating a fixed stack.

This leaves Factor in a strange position, whereby it is both extremely functional - only allowing to compose simpler functions into more complex ones - and largely imperative - describing operations on a mutable stack.

Playing with the stack
----------------------

Let us start looking what Factor actually feels like. Our first words will be literals, like `3`, `12.58` or `"Chuck Norris"`. Literals can be thought as functions that push themselves on the stack. Try writing `5` in the listener and then press enter to confirm. You will see that the stack, initially empty, now looks like

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

We will want to compute the factorial. To start with a concrete example, we compute the factorial of `10`, so we start by writing `10` on the stack. Now, the factorial is the product of the numbers from `1` to `10`, so we should produce such a list of numbers first.

The function to produce a range is reasonably called `[a,b]` (tokenization is trivial in Factor, as words are always space separated, and this allows you to use any combination of non-whitespace characters as an identifier). In our case one of the extremes is just `1`, so we can use the simpler word `[1,b]` instead. If you write that in the listener, you will be prompted with a choice, because the name `[1,b]` is not imported by default. Factor is able to suggest to import the `math.ranges` vocabulary, so choose that option and proceed.

You should now have on your stack a rather opaque structure which looks like

    T{ range f 1 10 1 }

This is because our range functions are lazy. To confirm that we actually have created the list of numbers from `1` to `10`, we convert it into an array using the word `>array`. Your stack should now look like

    { 1 2 3 4 5 6 7 8 9 10 }

which is promising.

Next, we want to take the product of those numbers. In many languages, this could be done with a function called reduce or fold. Let us look for one. Pressing `F1` will open a contextual help, where you can search for `reduce`. It turns out that `reduce` is actually the word we are looking for, but at this point it may not be obvious how to use it.

Try writing `1 [ * ] reduce` and look at the output: it is indeed the factorial of `10`. Now, `reduce` usually takes three arguments: a sequence (and we had one), a starting point (and this is the `1`) and a binary operation. This must certainly be the `*`, but what about those square brackets around it?

If we had written just `*`, Factor would have tried to apply multiplication to the topmost two elements, which is not what we wanted. What we need is a way to mention a word without applying it. Keeping our textual metaphor, this mechanism is called **quotation**. To quote one or more words, you just surround them by `[` and `]` (leaving spaces). What you get is akin to an anonymous function in other languages.

Let us `drop` the result to empty the stack, and try writing what we have done so far in a single shot: `10 [1,b] 1 [ * ] reduce`. This will output `3628800` as expected.

We now want to define a word that can be called whenever we want. We will call our word `!` as it is customary. To define it, we need to use the word `:`. Then we put the name of the word being defined, the **stack effects** and finally the body, ending with the `;` word:

    : ! ( n -- n! ) [1,b] 1 [ * ] reduce ;

What are the stack effects? These are the part like `( n -- n! )` in our case. They have no effect on the function, but allow you to name your inputs and outputs for documenting purposes. You can use any identifier to name them, and Factor will only make a consistency check that the number of inputs and outputs agrees with what the body does.

If you try to write

    : ! ( m n -- n! ) [1,b] 1 [ * ] reduce ;

Factor will signal an error that the number of inputs is not consistent. To restore the previous correct definition press `Ctrl+P` two times to get back to the previous input and then enter it.

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

Now that you know the basics of Factor, you may want to start assembling more complex words. This sometimes may require to use variables that are not on top of the stack, or to use variables more that once. There are a few words that can be used to this effect. I will mention them now, since you'd better be aware of them, but warn you that code using many of those words can quickly become hard to write and harder to read. It requires mentally simulating moving values on a stack, which is not a natural way to program. We will see a much more effective way to handle most needs in next section.

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

This immediately shows that `n` is used in two places: as an upper bound for the sequence, and as the number to test for divisibility. The word `bi` applies two different quotations to an element on the stack, and this is precisely what we need. For instance `5 [ 2 * ] [ 3 + ] bi` yields

    10
    8

To continue with our example, will need a word to test divisibility, and a quick search in the online help shows that `divisor?` is what we want. We will also need a way to make a range starting from `2`, which we can define like

    : [2,b] ( n -- {2,...,n} ) 2 swap [a,b] ; inline

What's up with that `inline` word? This is one of the modifiers we can use after defining a word, another one being `recursive`. This will allow us to have the definition of the word inlined wherever it is used, rather than incurring a function call.

It may also help to have the arguments for divisibility in the other direction, so we define

    : multiple? ( a b -- ? ) swap divisor? ; inline

Now, producing the range of numbers from `2` to the square root of `n` is easy: `sqrt floor [2,b]` (`floor` is not even necessary, as `[a,b]` works for non-integer bounds). What about the second function?

We must produce a function that test for being a divisor of `n` - in other words we need to partially apply the word `multiple?`. This can be done with the word `curry`, like this: `[ multiple? ] curry`.

Finally, once we have the range and the test function on the stack, we can test whether any element satisfies the divisibility with `any?`. Our full definition looks like

    : prime? ( n -- ? ) [ sqrt [2,b] ] [ [ multiple? ] curry ] bi any? not ;

Altough the definition is slightly complicated, the stack shuffling is minimal and limited to the small helper functions, which are much simpler to reason about than `prime?`.

Notice that our `prime?` word uses two levels of quotation nesting. In general, Factor words tend to be rather shallow, using one level of nesting for each higher-order function, unlike Lisps or more generally languages based on the lambda calculus, which use one level of nesting for each function, higher-order or not.

Many more combinators exists other than `bi` (and its relative `tri`), and you should become acquainted at least with `bi*` and `bi@`.

Vocabularies
------------

It is now time to start writing your functions in files and learn how to import them in the listener. Factor organizes words into nested namespaces called **vocabularies**. You can import all names from a vocabulary with the word `USE:`. In fact, you may have seen something like

    USE: math.ranges

when you asked the listener to import the word `[1,b]` for you. You can also use more than one vocabulary at a time with the word `USING:`, which is followed by a list of vocabularies and terminated by `;`, like

    USING: math.ranges sequences.deep ;

Finally, you define the vocabulary where your definitions are stored with the word `IN:`. If you search the online help for a word you have defined so far, like `prime?`, you will see that your definitions have been grouped under the default `scratchpad` vocabulary. By the way, this shows that the online help automatically collects information about your own words, which is a very useful feature.

There are a few more words, like `QUALIFIED:`, `FROM:`, `EXCLUDE:` and `RENAME:`, that allow more fine-grained control over the imports, but `USING:` is the most common.

On disk, vocabularies are stored under a few root directories, much like with the classpath in JVM languages. By default, the system starts looking up into the directories `basis`, `core`, `extra`, `work` under the Factor home. You can add more, both at runtime with the word `add-vocab-root`, and by creating a configuration file `.factor-rc`, but for now we will store our vocabularies under the `work` directory, which is reserved for the user.

Generate a template for a vocabulary writing

    USE: tools.scaffold
    "github.tutorial" scaffold-work

You will find a file `work/github/tutorial/tutorial.factor` containing an empty vocabulary. You can add the definitions of the previous paragraph, so that it looks like

    ! Copyright (C) 2014 Andrea Ferretti.
    ! See http://factorcode.org/license.txt for BSD license.
    USING: ;
    IN: github.tutorial

    : [2,b] ( n -- {2,...,n} ) 2 swap [a,b] ; inline

    : multiple? ( a b -- ? ) swap divisor? ; inline

    : prime? ( n -- ? ) [ sqrt [2,b] ] [ [ multiple? ] curry ] bi any? not ;

Since the vocabulary was already loaded when you scaffolded it, we need a way to refresh it from disk. You can do this with `"github.tutorial" refresh`. There is also a `refresh-all` word, with a shortcut `F2`.

You will be prompted a few times to use vocabularies, since your `USING:` statement is empty. After having accepted all of them, Factor suggests you a new header with all the needed imports:

    USING: kernel math.functions math.ranges sequences ;
    IN: github.tutorial

You now want to come back and edit the file to add this header. Of course, you still have the file open your editor, but if this was not the case, you could make use of Factor editor integration.

For instance, I am using Sublime Text right now, so I load the integration with

    USE: editors.sublime

After having done that, I can edit, say, the `multiple?` word with `\ multiple? edit`. I find Sublime Text open on the relevant line of the right file. Locate your favorite editor with the online help and do the same. This also works for words in the Factor distribution, although it may be a bad idea to modify them.

This `\` word requires a little explanation. It works like a sort of escape, allowing us to put a reference to the next word on the stack, without executing it. This is exactly what we need, because `edit` is a word that takes words themselves as arguments. This mechanism is similar to quotations, but while a quotation creates a new anonymous function, here we are directly refering to the word `multiple?`.

Back to our task, you may notice that the words `[2,b]` and `multiple?` are just helper functions that you may not want to expose directly. To hide them from view, you can wrap them in a private block like this

    <PRIVATE

    : [2,b] ( n -- {2,...,n} ) 2 swap [a,b] ; inline

    : multiple? ( a b -- ? ) swap divisor? ; inline

    PRIVATE>

After making this change and refreshed the vocabulary, you will see that the listener is not able to refer to words like `[2,b]` anymore. The `<PRIVATE` word works by putting all definitions in the private block under a different vocabulary, in our case `github.tutorial.private`.

It is still possible to refer to words in private vocabularies, as you can confirm by searching for `[2,b]` in the online help, but of course this is discouraged, since people do not guarantee any API stability for private words. Words under `github.tutorial` can refer to words in `github.tutorial.private` directly, like `prime?` does.

Tests and documentation
-----------------------

This is a good time to start writing some unit tests. You can create a skeleton with

    "github.tutorial" scaffold-tests

You fill find a generated file under `work/github/tutorial/tutorial-tests.factor`. Notice the line

    USING: tools.test github.tutorial ;

that imports the unit testing module as well as your own. We will only test the public `prime?` function.

Tests are written using the `unit-test` word, which expects two quotations: the first one containing the expected outputs and the second one containing the words to run in order to get that output. Add these lines to `github.tutorial-tests`:

    [ t ] [ 2 prime? ] unit-test
    [ t ] [ 13 prime? ] unit-test
    [ t ] [ 29 prime? ] unit-test
    [ f ] [ 15 prime? ] unit-test
    [ f ] [ 377 prime? ] unit-test
    [ f ] [ 1 prime? ] unit-test
    [ t ] [ 20750750228539 prime? ] unit-test

You can now run the tests with `github.tutorial` test. You will see that we have actually made a mistake, and pressing `F3` will show more details. It seems that our assertions fails for `2`.

In fact, if you manually try to run our functions for `2`, you will see that our defition of `[2,b]` returns `{ 2 }` for `2 sqrt`, due to the fact that the square root of two is less than two, so we get a descending interval. Try making a fix so that the tests now pass.

There are a few more words to test errors and inference of stack effects. `unit-test` suffices for now, but later on you may want to check `must-fail` and `must-infer`.

We can also add some documentation to our vocabulary. Autogenerated documentation is always available for user-defined words (even in the listener), but we can write some useful comments manually, or even add custom articles that will appear in the online help. Predictably, we start with

    "github.tutorial" scaffold-docs

The generated file `work/github/tutorial-docs.factor` import `help.markup` and `help.syntax`. These two vocabularies define words to generate documentation. The actual help page is generated by the `HELP:` parsing word.

The arguments to `HELP:` are nested array of the form `{ $directive content... }`. In particular, you see here the directives `$values` and`$description`, but a few more exist, such as `$errors`, `$examples` and `$see-also`.

Notice that the type of the output `?` has been inferred to be boolean. Change the first lines to look like

    USING: help.markup help.syntax kernel math ;
    IN: github.tutorial

    HELP: prime?
    { $values
        { "n" fixnum }
        { "?" boolean }
    }
    { $description "Tests if n is prime. n is assumed to be a positive integer." } ;

and refresh the `github.tutorial` vocabulary. If you now look at the help for `prime?`, for instance with `\ prime? help`, you will see the updated documentation.

You can also render the directives in the listener for quicker feedback. For instance, try writing

    { $values
        { "n" integer }
        { "?" boolean }
    } print-content

The help markup contains a lot of possible directives, and you can use them to write stand-alone articles in the help system. Have a look at some more with `"element-types" help`.

The object system and protocols
-------------------------------

Although it is not apparent from what we have said so far, Factor has object-oriented features, and many core words are actually method invocations. To better understand how objects behave in Factor, a quote is in order:

> I invented the term Object-Oriented and I can tell you I did not have C++ in mind.

> Alan Kay

The term object-oriented has as many different meanings as people using it. One point of view - which was actually central to the work of Alan Kay - is that it is about late binding of function names. In Smalltalk, the language where this concept was born, people do not talk about calling a method, but rather sending a message to an object. It is up to the object to decide how to respond to this message, and the caller should not know about the implementation. For instance, one can send the message `map` both to an array and a linked list, but internally the iteration will be handled differently.

The binding of the message name to the method implementation is dynamic, and this is regarded as the core strenght of objects. As a result, fairly complex systems can evolve from the cooperation of independent objects who do not mess with each other internals.

To be fair, Factor is very different from Smalltalk, but still there is the concept of classes, and generic words can defined having different implementations on different classes.

Some classes are builtin in Factor, such as `string`, `boolean`, `fixnum` or `word`. Next, the most common way to define a class is as a **tuple**. Tuples are defined with the `TUPLE:` parsing word, followed by the tuple name and the fields of the class that we want to define, which are called **slots** in Factor parlance.

Let us define a class for movies:

    TUPLE: movie title director actors ;

This also generates setters `>>title`, `>>director` and `>>actors` and getters `title>>`, `director>>` and `actors>>`. For instance, we can create a new movie with

    movie new "The prestige" >>title
      "Christopher Nolan" >>director
      { "Hugh Jackman" "Christian Bale" "Scarlett Johansson" } >>actors

We can also shorten this to

    "The prestige" "Christopher Nolan"
    { "Hugh Jackman" "Christian Bale" "Scarlett Johansson" }
    movie boa

The word `boa` stands for 'by-order-of-arguments' and is a constructor that fills the slots of the tuple with the items on the stack in order. `movie boa` is called a **boa constructor**, a pun on the Boa Constrictor. It is customary to define a most common constructor called `<movie>`, which in our case could be simply

    : <movie> ( title director actors -- movie ) movie boa ;

In other cases, you may want to use some defaults, or compute some fields.

The functional minded will be worried about the mutability of tuples. Actually, slots can be declared to be read-only with `{ slot-name read-only }`. In this case, the field setter will not be generated, and the value must be set a the beginning with a boa constructor. Other valid slot modifiers are `initial:` - to declare a default value - and a class word, such as `integer`, to restrict the values that can be inserted.

As an example, we define another tuple class for rock bands

    TUPLE: band { keyboards string read-only } { guitar string read-only }
      { bass string read-only } { drums string read-only } ;
    : <band> ( keyboards guitar bass drums -- band ) band boa ;

together with one instance

	"Richard Wright" "David Gilmour" "Roger Waters" "Nick Mason" <band>

Now, of course everyone knows that the star in a movie is the first actor, while in a rock band it is the bass player. To encode this, we first define a **generic word**

    GENERIC: star ( item -- star )

As you can see, it is declared with the parsing word `GENERIC:` and declares its stack effects but it has no implementation right now, hence no need for the closing `;`. Generic words are used to perform dynamic dispatch. We can define implementations for various classes using the word `M:`

    M: movie star actors>> first ;
    M: band star bass>> ;

If you write `star .` two times, you can see the different effect of calling a generic word on instances of different classes.

Builtin and tuple classes are not all that there is to the object system: more classes can be defined with set operations like `UNION:` and `INTERSECTION:`. Another way to define a class is as a **mixin**.

Mixins are defined with the `MIXIN:` word, and existing classes can be added to the mixin writing

    INSTANCE: class mixin

Methods defined on the mixin will then be available on all classes that belong to the mixin. If you are familiar with Haskell typeclasses, you will recognize a resemblance, although Haskell enforces at compile time that instance of typeclasses implent certain functions, while in Factor this is informally specified in documentation.

Two important examples of mixins are `sequence` and `assoc`. The former defines a protocol that is available to all concrete sequences, such as strings, linked lists or arrays, while the latter defines a protocol for associative arrays, such as hashtables or association lists.

This enables all sequences in Factor to be acted upon with a common set of words, while differing in implementation and minimizing code repetition (because only few primitives are needed, and other operations are defined for the `sequence` class). The most common operations you will use on sequences are `map`, `filter` and `reduce`, but there are many more - as you can see with `"sequences" help`.

Learning the tools
------------------

A big part of the productivity of Factor comes from the deep integration of the language and libraries with the tools around them, which are embodied in the listener. Many functions of the listener can be used programmatically, and viceversa. You have seen some examples of this:

* the help is navigable online, but you can also invoke it with `help` and print help items with `print-content`;
* the `F2` shortcut or the words `refresh` and `refresh-all` can be used to refresh vocabularies from disk while continuing working in the listener;
* the `edit` word gives you editor integration, but you can also click on file names in the help pages for vocabularies to open them.

The refresh is actually quite smart. Whenever a word is redefined, words that depend on it are recompiled against the new defition. You can check by yourself doing

    : inc ( x -- y ) 1 + ;
    : inc-print ( x -- ) inc . ;
    5 inc-print

and then

    : inc ( x -- y ) 2 + ;
    5 inc-print

This allows you to always keep a listener open, improving your definitions, periodically saving your definitions to file and refreshing, without ever having to reload Factor.

You can also save the whole state of Factor with the word `save-image` and later restore it by starting Factor with

    ./factor -i=path-to-image

In fact, Factor is image-based and only uses files when loading and refreshing vocabularies.

The power of the listener does not end here. Elements of the stack can be inspected by clicking on them, or by calling the word `inspector`. For instance try writing

    TUPLE: trilogy first second third ;
    : <trilogy> ( first second third -- trilogy ) trilogy boa ;
    "A new hope" "The Empire strikes back" "Return of the Jedi" <trilogy>
    "George Lucas" 2array

You will get an item that looks like

    { ~trilogy~ "George Lucas" }

on the stack. Try clicking on it: you will be able to see the slots of the array and focus on the trilogy or on the string by double-clicking on them. This is extremely useful for interactive prototyping. Special objects can customize the inspector by implementing the `content-gadget` method.

There is another inspector for errors. Whenever an error arises, it can be inspected with `F3`. This allows you to investigate exceptions, bad stack effects declarations and so on. The debugger allows you to step into code, both forwards and backwards, and you should take a moment to get some familiarity with it. You can also trigger the debugger manually, by entering some code in the listener and pressing `Ctrl+w`.

Another feature of the listener allows you to benchmark code. As an example, we write an intentionally inefficient Fibonacci:

    DEFER: fib-rec
    : fib ( n -- f(n) ) dup 2 < [ ] [ fib-rec ] if ;
    : fib-rec ( n -- f(n) ) [ 1 - fib ] [ 2 - fib ] bi + ;

(notice the use of `DEFER:` to define two mutually recursive words). You can benchmark the running time writing `40 fib` and then pressing Ctrl+t instead of Enter. You will get timing information, as well as other statistics. Programmatically, you can use the `time` word on a quotation to do the same.

You can also add watches on words, to print inputs and outputs on entry and exit. Try writing

    \ fib watch

and then run `10 fib` to see what happens. You can then remove the watch with `\ fib reset`.

Another very useful tool is the `lint` vocabulary. This scans word definitions to find duplicated code that can be factored out. As an example, let us define a word to check if a string starts with another one. Create a test vocabulary

    "lintme" scaffold-work

and add the following definition

    USING: kernel sequences ;
    IN: lintme

    : startswith? ( str sub -- ? ) dup length swapd head = ;

Load the lint tool with `USE: lint` and write `"lintme" lint-vocab`. You will get a report mentioning that the word sequence `length swapd` is already used in the word `(split)` of `splitting.private`, hence it could be factored out.

Now, you would not certainly want to modify the source of a word in the standard library - let alone a private one - but in more complex cases the lint tool is able to find actual repetitions. It is a good idea to lint your vocabularies from time to time, to avoid code duplication and as a good way to discover library words that you may have accidentally redefined.

Finally, there are a few utilities to inspect words. You can see the definition of a word in the help tool, but a quicker way can be `see`. Or, viceversa, you may use `usage.` to inspect the callers of a given word. Try `\ reverse see` and `\ reverse usage.`.

Metaprogramming
---------------

We now venture into the metaprogramming world, and write our first parsing word. By now, you have seen a lot of parsing words, such as `[`. `{`, `H{`, `USE:`, `IN:`, `<PRIVATE`, `GENERIC:` and so on. Each of those is defined with the parsing word `SYNTAX:` and interacts with Factor's parser.

The parser accumulates tokens onto an accumulator vector, unless it finds a parsing word, which is executed immediately. Since parsing words execute at compile time, they cannot interact with the stack, but they have access to the accumulator vector. Their stack effect must be `( accum -- accum )`. Usually what they do is ask the parser for some more tokens, do something with them, and finally push a result on the accumulator vector with the word `suffix!`.

As an example, we will define a literal for DNA sequences. A DNA sequence is a sequence of one of the bases cytosine, guanine, adenine and thymine, which we will denote by the letters c, g, a, t. Since there are four possible bases, we can encode each with two bits. Let use define a word

    : dna>bits ( token -- bits ) {
      { "a" [ { f f } ] }
      { "c" [ { t t } ] }
      { "g" [ { f t } ] }
      { "t" [ { t f } ] }
    } case ;

where the first bit represents whether the basis is a purine or a pyrimidine, and the second one identifies bases that pair together.

Our aim is to read a sequence of letters a, c, g, t - possibly with spaces - and convert them to a bit array. Factor supports bit arrays, and literal bit arrays look like `?{ f f t }`.

Our syntax for DNA will start with `DNA{` and get all tokens until the closing token `}` is found. The intermediate tokens will be put into a string, and using our function `dna>bits` we will map this string into a bit array. To read tokens, we will use the word `parse-tokens`. There are a few higher-level words to interact with the parser, such as `parse-until` and `parse-literal`, but we cannot apply them in our case, since the tokens we will find are just sequences of a c g t, instead of valid Factor words. Let us start with a simple approximation that just reads tokens between our delimiters and outputs the string obtained by concatenation

    SYNTAX: DNA{ "}" parse-tokens concat suffix!

You can test the effect by doing `DNA{ a ccg t a g }`, which should output `"accgtag"`. As a second approximation, we transform each letter into a boolean pair:

    SYNTAX: DNA{ "}" parse-tokens concat
      [ 1string dna>bits ] { } map-as suffix! ;

Notice the use of `map-as` instead of `map`. Since the target collection is not a string, we did not use `map`, which preserves the type, but `map-as`, which take as an additional argument an examplar of the target collection - here `{ }`.  Our final version flattens the array of pairs with `concat` and finally makes into a bit array:

    SYNTAX: DNA{ "}" parse-tokens concat
      [ 1string dna>bits ] { } map-as
      concat >bit-array suffix! ;

If you try it with `DNA{ a ccg t a g }` you should get

    `?{ f f t t t t f t t f f f f t }`

We only scratched the surface of parsing words by defining a simple literal. In general, they allow you to perform arbitrary computations at compile time, enabling powerful forms of metaprogramming.

In a sense, Factor syntax is completely flat, and parsing words allow you to introduce syntaxes more complex than a stream of tokens to be used locally. This permits to increase the Factor language by adding many new features as libraries. In principle, it would even be possible to have an external language compile to Factor - say JavaScript - and embed it as a Factor DSL inside the boundaries of a `<JS ... JS>` parsing word. Some taste is needed not to abuse too much of this to introduce styles that are much too alien in the concatenative world.

When the stack is not enough
----------------------------

Until now I have cheated a bit, and tried to avoid writing examples that would have been too complex to write in concatenative style. Truth is, you *will* find occasions where this is too restrictive. Fortunately, parsing words allow you to break these restrictions, and Factor comes with a few to handle the most common annoyances.

One thing you may want to do is to actually name local variables. The `::` word works like `:`, but allows you to actually bind the name of stack parameters to variables, so that you can use them multiple times, in the order you want. For instance, let us define a word to solve quadratic equations. I will spare you the purely stack-based version, and present you a version with locals (this will require the `locals` vocabulary):

    :: solveq ( a b c -- x )
      b neg
      b b * 4 a c * * - sqrt
      +
      2 a * / ;

In this case we have chosen the + sign, but we can do better and output both solutions:

    :: solveq ( a b c -- x1 x2 )
      b neg
      b b * 4 a c * * - sqrt
      [ + ] [ - ] 2bi
      [ 2 a * / ] bi@ ;

You can check that this definition works with something like `2 -16 30 solveq`, which should output both `3.0` and `5.0`. Apart from being written in RPN style, our first version of `solveq` looks exactly the same it would in a language with local variables. For the second definition, we apply both the `+` and `-` operations to -b and delta, using the combinator `2bi`, and then divide both results by 2a using `bi@`.

There is also support for locals in quotations - using `[|` - and methods - using `M::` - and one can also create a scope where to bind local variables outside definitions using `[let`. Of course, all of these are actually compiled to concatenative code with some stack shuffling. I encourage you to browse examples for these words, but bear in mind that their usage in practice is actually much less prominent than one would expect - about 1% of Factor's own codebase.

Another more common case happens when you need to specialize a quotation to some values, but these do not appear in the right place. Remember that you can partially apply a quotation using `curry`. But this assumes that the value you are applying should appear leftmost in the quotation; in the other cases you need some stack shuffling.

The `fry` vocabulary defines **fried quotations**; these are quotations that have holes in them - marked by `_` - that are filled with values from the stack. For instance, let me take again the example with `prime?`, but this time write it without using helper words:

    : prime? ( n -- ? ) [ sqrt 2 swap [a,b] ] [ [ swap divisor? ] curry ] bi any? not ;

The first quotation is rewritten more simply as

    [ '[ 2 _ sqrt [a,b] ] call ]

Here we use a fried quotation - starting with `'[` - to inject the element on the top of the stack in the second position, and then use `call` to evaluate the resulting quotation. The second quotation becomes simply

    [ '[ _ swap divisor? ] ]

so an alternative defition of `prime?` is

    : prime? ( n -- ? ) [ '[ 2 _ sqrt [a,b] ] call ] [ '[ _ swap divisor? ] ] bi any? not ;

Depending on you taste, you may find this version more readable. In this case, the added clarity is probably lost due to the fact that the fried quotations are themselves inside quotations, but occasionally their use can do a lot to simplify the flow.

Finally, there are times where one just wants to give names to variables that are available inside some scope, and use them where necessary. These values can hold values that are global, or at least not local to a single word. A typical example could be the input and output streams, or database connections.

Factor allows you to create **dynamic variables** and bind them in scopes. The first thing is to create a **symbol** for a variable, say

    SYMBOL: favorite-language

Then one can use the word `set` to bind the variable and `get` to retrieve its values, like

    "Factor" favorite-language set
    favorite-language get

Scopes are nested, and new scopes can be created with the word `with-scope`. Try for instance

    : on-the-jvm ( -- ) [
      "Scala" favorite-language set
      favorite-language get .
    ] with-scope ;

If you run `on-the-jvm` you will get `"Scala"` printed, but still after execution `favorite-language get` return `"Factor"`.

All the tools we have seen in this section should be used when necessary, as they break concatenativity and makes words less easy to factor, but greatly increase clarity when needed. Factor has a very practical approach and does not shy from offering features that are less pure but nevertheless often useful.


Input/Output
------------

We now leave the tour of the language, and start investigating how to interact with the outside world. I will begin in this section with some examples of input/output, but inevitably this will lead into a discussion of asynchrony. The rest of the tutorial will then go in more detail about parallel and distributed computing.

Factor implements efficient asynchronous input/output facilities, similar to NIO on the JVM or the Node.js I/O system. This means that input and output operations are performed in the background, leaving the foreground task free to perform work while the disk is spinning or the network is buffering packets. Factor is currently single threaded, but asynchrony allows it to be rather performant for applications that are I/O-bound.

All of Factor input/output words are centered on **streams**. Streams are lazy sequences which can be read or written to, typical examples being files, network ports or the standard input and output. Factor holds a couple of dynamic variables called `input-stream` and `output-stream`, which are used by most I/O words. These variables can be rebound locally using `with-input-stream`, `with-output-stream` and `with-streams`. When you are in the listener, the default streams write and read in the listener, but once you deploy your application as an executable, they are usually bound to the standard input and output of your console.

The words `<file-reader>` and `<file-writer>` (or `<file-appender>`) can be used to create a read or write stream to a file, given its path and encoding. Putting everything together, we make a simple example of a word that reads each line of a file encoded in UTF8, and write the first letter of the line to the listener.

First, we want a `safe-head` word, that works like `head`, but returns its input if the sequence is too short. To do so, we will use the word `recover`, which allows us to declare a try-catch block. It requires two quotations: the first one is executed, and on failure, the second one is executed with the error as input. Hence we can define

    : safe-head ( seq n -- seq' ) [ head ] [ 2drop ] recover ;

With this definition, we can make a word to read the first character of the first line:

    : read-first-letters ( path -- )
      utf8 <file-reader> [
        readln 1 safe-head write nl
      ] with-input-stream ;

Using the helper word `with-file-reader`, we can also shorten this to

    : read-first-letters ( path -- )
      utf8 [
        readln 1 safe-head write nl
      ] with-file-reader ;

Unfortunately, we are limited to one line. To read more lines, we should chain calls to `readln` until one returns `f`. Factor helps us with the word `file-lines`, which lazily iterates over lines. Our final definition becomes

    : read-first-letters ( path -- )
      utf8 file-lines [ 1 safe-head write nl ] each ;

When the file is small, one can also use `file-contents` to read the whole contents of a file in a single string. Factor defines many more words for input/output, which cover many more cases, such as binary files or sockets.

We end this section investigating some words to walk the filesystem. Our aim is a very minimal implementation of the `ls` command.

The word `directory-entries` lists the contents of a directory, giving a list of tuple elements, each one having the slots `name` and `type`. You can see this by trying `"/home" directory-entries [ name>> ] map`. If you inspect the directory entries, you will see that the type is either `+directory+` or `+regular-file+` (well, there are symlinks as well, but we will ignore them for simplicity). Hence we can define a word that lists files and directories with

    : list-files-and-dirs ( path -- files dirs )
        directory-entries [ type>> +regular-file+ = ] partition ;

With this, we can define a word `ls` that will print directory contents as follows:

    : ls ( path -- )
      list-files-and-dirs
      "DIRECTORIES:" write nl
      "------------" write nl
      [ name>> write nl ] each
      "FILES:" write nl
      "------" write nl
      [ name>> write nl ] each ;

Try the word on your home directory to see the effects. In the next section, we shall look at how to create an executable for our simple program.

Deploying programs
------------------

There are two ways to run Factor programs outside the listener: as scripts, which are interpreted by Factor, or as standalone executable compiled for your platform. Both require to define a vocabulary with an entry point (altough there is an even simpler way for scripts), so let's do that first.

Start by creating our `ls` vocabulary with `"ls" scaffold-work` and make it look like this:

    ! Copyright (C) 2014 Andrea Ferretti.
    ! See http://factorcode.org/license.txt for BSD license.
    USING: accessors command-line io io.directories io.files.types
      kernel namespaces sequences ;
    IN: ls

    <PRIVATE

    : list-files-and-dirs ( path -- files dirs )
        directory-entries [ type>> +regular-file+ = ] partition ;

    PRIVATE>

    : ls ( path -- )
      list-files-and-dirs
      "DIRECTORIES:" write nl
      "------------" write nl
      [ name>> write nl ] each
      "FILES:" write nl
      "------" write nl
      [ name>> write nl ] each ;

When we run our vocabulary, we will need to read arguments from the command line. Command-line arguments are stored under the `command-line` dynamic variable, which holds an array of strings. Hence - forgetting any error checking - we can define a word which runs `ls` on the first command-line argument with

    : ls-run ( -- ) command-line get first ls ;

Finally, we use the word `MAIN:` to declare the main word of our vocabulary:

    MAIN: ls-run

Having added those two lines to your vocabulary, you are now ready to run it. The simplest way is to run the vocabulary as a script with the `-run` flag passed to Factor. For instance to list the contents of my home I can do

    ./factor -run=ls /home/andrea

In order to produce an executable, we must set some options and call the `deploy` word. The simplest way to this graphically is to invoke the `deploy-tool` word. If you write `"ls" deploy-tool`, you will be presented with a window to choose deployment options. For our simple case, we will leave the default options and choose Deploy.

After a little while, you should be presented with an executable that you can run like

    cd ls
    ./ls /home/andrea

Try making the `ls` program more robust by handling missing command-line arguments and non-existent or non-directory arguments.

Multithreading
--------------

As we have said, the Factor runtime is single-threaded, like Node. Still, one can emulate concurrency in a single-threaded setting by making use of **coroutines**. This are essentially cooperative threads, which periodically release control with the `yield` word, so that the scheduler can decide which coroutine run next.

Although cooperative threads do not allow to make use of multiple cores, they still have some benefits:
* input/output operations can avoid to block the entire runtime, so that one can implement quite performant applications if I/O is the bottleneck;
* user interfaces are naturally a multithreaded construct, and they can be implemented in this model, as the listener itself shows;
* finally, some problems may just naturally be easier to write making use of multithreaded constructs.

For the cases where one wants to make use of multiple cores, Factor offers the possibility of spawning other processes and communicating between them with the use of **channels**, as we will see in a later section.

Threads in Factors are created out of a quotation and a name, with the `spawn` word. Let use use this to print the first few lines of Star Wars, one per second, each line being printed inside its own thread. First, let us write those lines inside a dynamic variable:

    SYMBOL: star-wars

    "A long time ago, in a galaxy far, far away....

    It is a period of civil war. Rebel
    spaceships, striking from a hidden
    base, have won their first victory
    against the evil Galactic Empire.

    During the battle, rebel spies managed
    to steal secret plans to the Empire's
    ultimate weapon, the DEATH STAR, an
    armored space station with enough
    power to destroy an entire planet.

    Pursued by the Empire's sinister agents,
    Princess Leia races home aboard her
    starship, custodian of the stolen plans
    that can save her people and restore
    freedom to the galaxy...."
    "\n" split star-wars set

We will spawn 18 threads, each one printing a line. The operation that a thread must run amounts to

    star-wars get ?nth print

Note that dynamic variables are shared between threads, so each one had access to star-wars. This is fine, since it is read-only, but the usual caveats about shared memory in a multithreaded settings apply.

Let us define a word for the thread workload

    : print-a-line ( i -- ) star-wars get ?nth print ;

If we give the i-th thread the name "i", our example amounts to

    18 [0,b) [
      [ [ print-a-line ] curry ]
      [ number>string ]
      bi spawn
    ] each

Note the use of `curry` to send i to the quotation that prints the i-th line. This is almost what we want, but it runs too fast. We need to put the thread in sleep for a while. So we `clear` the stack that now contains a lot of thread objects and look for a `sleep` word in the help.

It turns out that `sleep` does exactly what we need, but it takes a **duration** object as input. We can create a duration of i seconds with... well `i seconds`. So we define

    : wait-and-print ( i -- ) dup seconds sleep print-a-line ;

Let us try

    18 [0,b) [
      [ [ wait-and-print ] curry ]
      [ number>string ]
      bi spawn
    ] each

Instead of `spawn` we can also use `in-thread` which uses a dummy thread name and discards the returned thread, simplifying the above to

    18 [0,b) [
      [ wait-and-print ] curry in-thread
    ] each

This is good enough for our simple purpose. In serious applications, theads will be long running. In order to make them cooperate, one can use the `yield` word to signal that the thread has done a unit of work, and other threads can gain control. You also may want to have a look at other words to `stop`, `suspend` or `resume` threads.

Servers and Furnace
-------------------

A very common case for using more than one thread is when writing server applications. When writing network applications, it is common to start a thread for each incoming connection (remember that this are green threads, so they are much more lightweight than OS threads).

To simplify this, Factor has the word `spawn-server`, which works like `spawn`, but in addition repeatedly spawns the quotation until it returns `f`. This is still a very low-level word: in reality one has to do much more: listen for TCP connections on a given port, handle connection limits and so on.

The vocabulary `io.servers` allows to write and configure TCP servers. A server is created with the word `<threaded-server>`, which requires an encoding as a parameter. Its slots can the be set to configure logging, connection limits, ports and so on. The most important slot to fill is `handler`, which contains a quotation that is executed for each incoming connection. You can see a very simple example of server with

    "resource:extra/time-server/time-server.factor" edit-file

We will raise the level of abstraction even more and show how to run a simple HTTP server. First, `USE: http.server`.

An HTTP application is built out of a **responder**. A responder is essentially a function from a path and an HTTP request to an HTTP response, but more concretely is anything that implements the method `call-responder*`. Responses are instances of the tuple `response`, so are usually generated calling `<response>` and customizing a few slots. Let us write a simple echo responder:

    TUPLE: echo-responder ;

    : <echo-responder> ( -- responder ) echo-responder new ;

    M: echo-responder call-responder*
      drop
      <response>
        200 >>code
        "Document follows" >>message
        "text/plain" >>content-type
        swap concat >>body ;

Responders are usually combined to form more complex responders, in order to implement routing and other features. In our simplistic example, we will use just this one responder, and set it globally with

    <echo-responder> main-responder set-global

Once you have done this, you can start the server with `8080 httpd`- You can then visit `http://localhost:8080/hello/%20/from/%20/factor` in your browser to see your first responder in action. You can then stop the server with `stop-server`.

Now, if this was all that Factor offers to write web applications, it would still be rather low level. In reality, web applications are usually written using a web framework called **Furnace**.

Furnace allows us -among other thing - to write more complex actions using a template language. Actually, there are two template languages shipped by default, and we will use **Chloe**. Furnace allows us to use create **page actions** from Chloe templates, and in order to create a responder we will need to add routing.

Let use first investigate a simple example of routing. To do this, we create a special type of responder called a **dispatcher**, that dispatches requests based on path parameters. Let us create a simple dispatcher that will choose between our echo responder and a default responder used to serve static files.

    dispatcher new-dispatcher
      <echo-responder> "echo" add-responder
      "/home/andrea" <static> "home" add-responder
      main-responder set-global

Of course, substitute the path `/home/andrea` with any folder you like. If you start again the server with `8080 httpd`, you should be able to see both our simple echo responder (under `/echo`) and the contents of your files (under `/home`). Notice that directory listing is disabled by default, you can only access the content of files.

Now that you know how to do routing, we can write page actions in Chloe. Things are starting to become complicated, so we scaffold a vocabulary with `"hello-furnace" scaffold-work`. Make it look like this:


    ! Copyright (C) 2014 Andrea Ferretti.
    ! See http://factorcode.org/license.txt for BSD license.
    USING: accessors furnace.actions http http.server
      http.server.dispatchers http.server.static kernel sequences ;
    IN: hello-furnace


    TUPLE: echo-responder ;

    : <echo-responder> ( -- responder ) echo-responder new ;

    M: echo-responder call-responder*
      drop
      <response>
        200 >>code
        "Document follows" >>message
        "text/plain" >>content-type
        swap concat >>body ;

    TUPLE: hello-dispatcher < dispatcher ;

    : <example-responder> ( -- responder )
      hello-dispatcher new-dispatcher
        <echo-responder> "echo" add-responder
        "/home/andrea" <static> "home" add-responder
        <page-action>
          { hello-dispatcher "greetings" } >>template
        "chloe" add-responder ;

Most things are the same we have done in the listener. The only difference is that we have added a third responder in our dispatcher, under `chloe`. This responder is created with a page action. The page action has many slots - say to declare the behaviour of receiving the result of a form - but we only set its template. THis is a pair with the dispatcher class and the relative path of the template file.

In order for all this to work, create a file `work/hello-furnace/greetings.xml` with a content like

    <?xml version='1.0' ?>

    <t:chloe xmlns:t="http://factorcode.org/chloe/1.0">
      <p>Hello from Chloe</p>
    </t:chloe>

Reload the `hello-furnace` vocabulary and `<example-responder> main-responder set-global`. You should be able to see the results of your efforts under `http://localhost:8080/chloe`. Notice that there was no need to restart the server, but we can change the main responder dynamically.

This ends our very brief tour of Furnace. It actually does much more than this: form validation and handling, authentication, logging and more. But this section is already getting too long, and you will have to find out more in the documentation.

Processes and channels
----------------------

As I said, Factor is single-threaded from the point of view of the OS. If we want to make use of multiple cores, we need a way to spawn Factor processes and communicate between them. Factor implements two different models of message-passing concurrency: the actor model, which is based on the idea of sending messages asynchronously between threads, and the CSP model, based on the use of **channels**.

As a warm-up, we will make a simple example of communication between threads in the same process.

    FROM: concurrency.messaging => send receive ;

We can start a thread that will receive a message and print it repeatedly:

    : print-repeatedly ( -- ) receive . print-repeatedly ;
    [ print-repeatedly ] "printer" spawn

A thread whose quotation starts with `receive` and calls itself recursively behaves like an actor in Erlang or Akka. We can then use `send` to send messages to it. Try `"hello" over send` and then `"threading" over send`.

Channels are a slightly different abstractions, used for instance in Go and in Clojure core.async. They decouple the sender and the receiver, and are usually used synchronously. For instance, one side can receive from a channel before the some other party sends something to it. This just means that the receiving end yields control to the scheduler, which waits for the send of the message before giving control to the receiver again. This feature sometimes makes it easier to synchronize multithreaded applications.

Again, we first use a channel to communicate between threads in the same process. As expected, `USE: channels`. You can create a channel with `<channel>`, write to it with `to` and read from it with `from`. Note that both operations are blocking: `to` will block until the value is read in a different thread, and `from` will block until a value is available.

We create a channel and give it a name with

    SYMBOL: ch
    <channel> ch set

Then we write on it in a separate thread, in order not to block the UI

    [ "hello" ch get to ] in-thread

We can then read the value in the UI with

    ch get from

We can also invert the order:

    [ ch get from . ] in-thread
    "hello" ch get to

This works fine, since we had set the reader first.

Now, for the interesting part: we will start a second Factor instance and communicate via message sending. Factor transparently supports sending messages over the network, serializing values with the `serialize` vocabulary.

Start another instance of Factor, and run a node server on it. We will use the word `<inet4>`, that creates an IPv4 address from a host and a port, and the `<node-server>` constructor

    USE: concurrency.distributed
    f 9000 <inet4> <node-server> start-server

Here we have used `f` as host, which just stands for localhost. We will also start a thread that keeps a running count of the numbers it has received.

    FROM: concurrency.messaging => send receive ;
    : add ( x -- y ) receive + dup . add ;
    [ 0 add ] "adder" spawn

Once we have started the server, we can make a thread available with `register-remote-thread`:

    dup name>> register-remote-thread

Now we switch to the other instance of Factor. Here we will receive a reference to the remote thread and start sending numbers to it. The address of a thread is just the address of its server and the name we have registered the thread with, so we obtain a reference to our adder thread with

    f 9000 <inet4> "adder" <remote-thread>

Now, we reimport `send` just to be sure (there is an overlap with a word having the same name in `io.sockets`, that we have imported)

    FROM: concurrency.messaging => send receive ;

and we can start sending numbers to it. Try `3 over send`, and then `8 over send` - you should see the running total printed in the other Factor instance.

What about channels? We go back to our server, and start a channel there, just as above. This time, though, we `publish` it to make it available remotely:

    USING: channels channels.remote ;
    <channel> dup publish

What you get in return is an id you can use remotely to communicate. For instance, I just got `72581372615274718877979307388951222312843084896785643440879198703359628058956` (yes, they really want to be sure it is unique!).

We will wait on this channel, thereby blocking the UI:

    swap from .

In the other Factor instance we use the id to get a reference to the remote channel and write to it

    f 9000 <inet4> 72581372615274718877979307388951222312843084896785643440879198703359628058956 <remote-channel>
    "Hello, channels" over to

In the server instance, the message should be printed.

Remote channels and threads are both useful to implement distributed applications and make good use of multicore servers. Of course, it remains the question how to start worker nodes in the first place. Here we have done it manually - if the set of nodes is fixed, this is actually an option.

Otherwise, one could use the `io.launcher` vocabulary to start other Factor instances programmatically.

Where to go from here?
----------------------

We have covered a lot of ground, and I hope that by now you have a feeling whether Factor clicks for you. You can now work your way through the documentation, and hopefully contribute to Factor yourself.

Let me end with a few tips:

- when starting to write Factor, it is *very* easy to deal a lot with stack shuffling. Learn the combinators well, and do not fear to throw away oyur first examples;
- no definition is too short: aim for one line;
- the help system and the inspector are your best friends.

To be fair, we also have to mention some drawbacks of Factor:

- first, the community is really small. What they have done is impressive, but do not hope to find a lot of information on the internet;
- the concatenative model is very powerful, but also very hard to get right;
- Factor lacks threads: although the distributed processes make up for it, they incur some cost in serialization;
- finally, Factor does not currently have a package manager, and this probably hinders contribution.

We have to balance the last observation with the convenience of having the whole source tree of Factor available in the image, which certainly makes it easier to learn about libraries. Let me suggest a few vocabularies that you may want to have a look at:

- first, I have not talked a lot about errors and exceptions. Learn more with `"errors" help`;
- the `macros` vocabulary implements a form of compile time metaprogramming less general than parsing words, but still quite convenient;
- the `models` vocabulary lets you implement a form of dataflow programming using objects with observable slots;
- the `match` vocabulary implements a form of pattern matching;
- the `monads` vocabulary implements Haskell style monads.

I think these vocabulary are a testament to the power and expressivity of Factor. Happy hacking!

    USE: images.http
    "http://factorcode.org/logo.png" http-image.
