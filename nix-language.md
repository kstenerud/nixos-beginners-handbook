A Brief Tour of the Nix Expression Language
===========================================

I like to learn about new things by first looking at a mix of the fundamentals and top-level, followed by the middle bits. All NixOS configurations are defined using the Nix expression language, so we'll need to understand how the language works at a basic level in order to make any sense of the configurations.


Why a functional language?
--------------------------

Functional languages behave more in a manner that mathematicians would expect: A function yields a value based on a formula. The restrictions of mathematical-style functions produce certain behaviors that Nix relies upon in order to make it easy to rebuild the *exact same* OS every single time:

### [Purity](https://en.wikipedia.org/wiki/Pure_function)

A pure function is a function where:

- Calling the function with the same arguments yields the same results every time.
- The function has no side effects (no mutation of global or static state).

Consider the following function in a fictional language:

```
my_function(x):
     a = func_1(x)
     b = func_2(a)
     return func_3(a, b)
```

If this were an imperative language, `my_function` very quickly becomes difficult to reason about temporally. Calling `my_function(10)` a bunch of times could yield wildly different results if for example `func_2` were using an internal persistent counter and `func_3` were storing past values to use in future calculations. With pure functions, `my_function(10)` yields the same value every time it's called.

### [Immutability](https://en.wikipedia.org/wiki/Immutable_object)

When variables are immutable, it makes program listings much easier to follow. Even if the above function were hundreds of lines long, we'd know for certain that the `a` variable if used near the end of the function *still* contains the result of `func_1`.

### [Lazy Evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation)

Because expressions can get quite big in functional languages (many gigabytes in the case of Nix), we need a way to stop the program from bringing in the entire world just to pluck a single value we're interested in. Lazy evaluation helps by only evaluating as much as is actually needed in order to get the values we're interested in.



Nix expression language is a functional language
------------------------------------------------

Because the Nix expression language is functional, it's very easy to reason about the configurations it will generate:

### All "variables" are immutable.

You can't say `a = 1` and then change it like `a = 2`. Once you bind a value to a Nix variable, it can't be changed. On the surface this is incredibly restricting, but the benefit is that you don't need to look at your program temporally anymore.

I'll continue calling them "variables" in this document even though they can't change.

### Everything is an expression

Since Nix functions are pure, calling a function and not storing its result is useless.

```
my_function(x):
     a = func_1()
     func_x(x)
     b = func_2(a)
     return func_3(a, b)
```

In the above example (still using a fictional functional language), `func_x` has no effect because it is pure, and the result of its call is not used anywhere. In fact, this function will return the same value no matter what value of `x` you pass in, rendering `x` useless!

Because the effects of a function are useless unless stored or used somewhere, everything in Nix is essentially an expression. Every bit of code is either calculating a value, or temporarily storing an intermediate value for the calculation of its final value. At the end of your program, you're left with a single value that represents the result of the program. Anything that wasn't returned is lost.

In fact, your NixOS configuration is basically one big expression yielding the set of your operting system's configuration (the programs installed, the users of the system, the services running, etc). This is what makes deterministic builds possible: Pure functions that yield the same results for the same inputs every time are put together into a final expression that generates the same running system every time.



Nix REPL
--------

As a brief aside, I'll introduce the Nix REPL. You'll want to make use of this while learning how the language works.

The Nix REPL (read-evaluate-print-loop) allows you to play around in the language and see what would happen if you tried different things. For example:

```
$ nix repl
nix-repl> a = 1

nix-repl> b = 2

nix-repl> a + b
3
```

Hint: Press CTRL-D (the code for end-of-file) to exit the repl.


### Whitespace

Be careful when omitting whitespace. '-' is allowed in identifiers, and '/' is allowed in paths:

```
nix-repl> a-b
error: undefined variable 'a-b'

       at «string»:1:1:

            1| a-b
             | ^
            2|
```
```
nix-repl> a/b
/home/mynixuser/a/b
```



Nix Data Types
--------------

Nix expression language supports the following data types:

- Null
- Boolean
- Integer
- Float
- String
- Path
- List
- Set
- Function

Null, Boolean, integer, and float behave similarly to what you'd be used to in other languages. The others need more explanation:


### String

Strings are enclosed in double quotes (either using the double-quote character `"` or two single-quote characters `''`):

```
nix-repl> "foo \" bar"
"foo \" bar"

nix-repl> ''foo " bar''
"foo \" bar"
```

Strings support concatenation with other strings using the `+` operator:

```
nix-repl> "hello" + "world"
"helloworld"
```

Strings support embedded expressions that resolve to strings (using `${}`):

```
nix-repl> foo="world"

nix-repl> "hello ${foo}" 
"hello world"
```

Note that the result of the `${}` expression MUST be a string (or manually converted to a string):

```
nix-repl> foo = 1

nix-repl> "hello ${foo}"
error: cannot coerce an integer to a string

       at «string»:1:2:

            1| "hello ${foo}"
             |  ^
            2|

nix-repl> "hello ${builtins.toString foo}"
"hello 1"
```


### Path

A path is at the core just a string, but it is treated specially by Nix because Nix deals with files so much. There's no real magic otherwise.


### List

A list is just an ordered collection of values:

```
nix-repl> [1 2 "a" "b"]
[ 1 2 "a" "b" ]
```


### Set

A set is a collection of associations of values to strings. In many languages this would be called a "map" or "dictionary". Values can be retrieved using the `.` operator.

Notes:
- Keys must be strings.
- Each key-value pair must be terminated with a semicolon.
- Double-quotes around the key may be omitted when only safe characters are present. 

```
nix-repl> x = {a="foo"; "b"="bar";}

nix-repl> x
{ a = "foo"; b = "bar"; }

nix-repl> x.a
"foo"
```

```
nix-repl> {1="foo";}                
error: syntax error, unexpected INT

       at «string»:1:2:

            1| {1="foo";}
             |  ^
            2|
```

#### Recursive Set

Nix also supports recursive sets, which will be important in more advanced configurations. A recursive set allows you to refer to another value in the same set:

```
nix-repl> { a = 1; b = a+2; }
error: undefined variable 'a'

       at «string»:1:14:

            1| { a = 1; b = a+2; }
             |              ^
            2|

nix-repl> rec { a = 1; b = a+2; }
{ a = 1; b = 3; }
```

Beware! Recursive sets are suceptible to infinite recursion!

```
nix-repl> rec { a = 1; b = a+2; }
{ a = 1; b = 3; }

nix-repl> rec {x = y; y = x;}.x
error: infinite recursion encountered

       at «string»:1:10:

            1| rec {x = y; y = x;}.x
             |          ^
            2|
```


### Function

A function takes a single value as input and produces a single value as output. The definition syntax is `<input variable>: <expression>`, and the calling syntax is `<function> <value>`

```
nix-repl> myfunc = x: x * 2

nix-repl> myfunc 5
10

nix-repl> myfunc 7 
14
```

To support more than one parameter, we use nested functions:

```
nix-repl> mul = a: (b: a * b)
```

In this case, we're assigning to `mul` a function that takes a value we call `a`, and then returns another function. This next-deeper function takes a value we call `b`, and has access to the scopes outside of itself, meaning that it has access to `a`. This inner function then returns the expression `a * b`.

So if we were to call:

```
nix-repl> mul 5
«lambda @ (string):1:6»
```

The REPL evaluates this and prints the result, which is the inner function (lambda). To get the expression we want, we need to *also* invoke this returned function:

```
nix-repl> (mul 5) 4         
20
```

The parentheses help to visualize what's going on, but the Nix parser is pretty good at figuring things out for itself, so usually you can define and call "multi-parameter" functions like this:

```
nix-repl> mul = a: b: a * b

nix-repl> mul 5 4
20
```

`mul` is still a function returning a function returning an expression, but it's much more human-friendly to write it this way.

#### Partial Application

Incidentally, another cool trick in functional programs is [partial application](https://en.wikipedia.org/wiki/Partial_application). Since multi-parameter functions are in reality functions returning functions, we can create "partial" functions that have a parameter already defined:

```
nix-repl> mul = a: b: a * b

nix-repl> mul5 = mul 5

nix-repl> mul10 = mul 10

nix-repl> a = 7

nix-repl> mul5 a         
35

nix-repl> mul10 a
70
```



Operators and Expressions
-------------------------


### Basic Operators

Nix expression language supports the basic operators:

- Arithmetic operators `+`, `-`, `*`, `/`
- Boolean operators `||`, `&&`, `!`
- Relational operators `!=`, `==`, `<`, `>`, `<=`, `>=`

```
nix-repl> 1 + 3
4
nix-repl> 4 > 3 && 5 < 4
false
```


### If

The `if` expression will be familiar to imperative programmers, except that it doesn't contain statements (it must yield an expression):

```
nix-repl> x = 5                             

nix-repl> if x > 10 then "foo" else null    
null
```


### Let ... in

The `let` expression feels in many ways like an anonymous function call yielding an expression:

```
nix-repl> let a = 6; b = 5; in a * b 
30
```

This means `let` the variable `a` equal 6 and `b` equal 5 `in` the expression that follows (`a * b`), and then take the result of that expression. `let` allows us to create temporary variables for a calculation rather than polluting the namespace with variables that won't be used again.

You can also refer to external variables:

```
nix-repl> x = 5

nix-repl> let a = 3; in a + x
8
```

You can nest `let` expressions:

```
nix-repl> let a = 3; in let b = 4; in a + b
7
```

`let` will be useful when dealing with repeating values in your configurations. A trivial example:

```
{ config, pkgs, ... }:

let
  username = "karl";
in {
  users.users.${username} = {
    home = "/home/${username}";
  };
}
```


### With

The `with` expression brings a set's symbols into a local inner scope (similar to using `import` to bring in library symbols in other languages). This helps keep repetition and boilerplate down.

```
nix-repl> some_set = {a="foo"; b="bar";}

nix-repl> with some_set; a + b
"foobar"
```

You can also nest `with` expressions:

```
nix-repl> some_set = {a="foo"; b="bar";}

nix-repl> another_set = {c="baz";}

nix-repl> with some_set; with another_set; a + c
"foobaz"
```

Note that symbols can be shadowed (as they can in most sub-expressions):

```
nix-repl> some_set = {a="foo"; b="bar";}

nix-repl> another_set = {a="baz";}

nix-repl> with some_set; with another_set; a + b
"bazbar"
```
