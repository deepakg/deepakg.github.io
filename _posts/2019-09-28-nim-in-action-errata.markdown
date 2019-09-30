---
layout: post
title:  "Nim in Action: Book errata for Nim v1.0.0"
date:   2019-09-28 16:37:00 +0200
categories: nim
---

The Nim programming language recently hit the 1.0.0 milestone and I decided to pick it up. Since Manning's [Nim in Action](https://www.manning.com/books/nim-in-action) came back in 2017, it naturally targets an older version of Nim. Fortunately, there haven't been big user-visible changes to Nim over these two years so it's still possible to use this book to learn. That said, as is inevitable with a new language that's still under development, some things *have* changed.

I am compiling an errata of sorts as I work my way through the book. All page numbers correspond to the PDF/print version of the book. While a lot of compiler error messages have changed, I'll be highlighting only the important ones.

## Chapter 1

### Listing 1.7

```
import sequtils, future, strutils
     let list = @["Dominik Picheta", "Andreas Rumpf", "Desmond Hume"]
     list.map(
       (x: string) -> (string, string) => (x.split[0], x.split[1])
     ).echo
```

gives a warning during compilation:

```
Warning: Use the new 'sugar' module instead; future is deprecated [Deprecated]
```

Simply changing `future` to `sugar` fixes it without requiring any other change to the code in this listing.


## Chapter 2

### Section 2.2.2

#### Page 32

> I added the echo at the top of fillString’s body, in order to show you that it’s executed at compile time. Try compiling the example using Aporia or in a terminal by executing nim c file.nim.

Be aware that Aporia IDE is now obsolete. The [project's page mentions VSCode](https://github.com/nim-lang/Aporia) as an alternative. I personally use emacs with nim-mode and find it more than adequate.

There are plenty of references to Aporia throughout the book. I won't highlight them in this post any further.

#### Page 33

> For more information about Nim’s conventions, take a look at the “Style Guide for Nim Code” on GitHub: https://github.com/nim-lang/Nim/wiki/Style-Guide-for-Nim-Code.

The Style Guide page has now moved to:

[https://nim-lang.org/docs/nep1.html](https://nim-lang.org/docs/nep1.html)

### Section 2.3.1

#### page 40

> Compilation will fail with “Error: index out of bounds.”

It's still a compile-time out of bounds error but the error message is now worded differently:

```
Error: index 5000 not in 0 .. 2
```

### Section 2.3.2

#### page 38

```
doAssert getUserCity("Damien", "Lundi") == "Tokyo"
doAssert getUserCity(2) == "New York
```

It might be confusing to see that the book uses `assert` for previous code snippets but for this one, suddenly switches to `doAssert`.

The different between `assert` and `doAssert`is that `assert` is ignored by the nim compiler when using the `-d:release` or `--assertions:off` command line switches, whereas `doAssert` isn't affected by them. [Source](https://nim-lang.org/docs/assertions.html).

#### page 41

> “You should see that your program crashes with the following output:

```
Traceback (most recent call last)
segfault.nim(2)          segfault
SIGSEGV: Illegal storage access. (Attempt to read from nil?)
```

This still segfaults but the error message is now different and definitely more helpful:

```
segfault.nim(2) segfault
/usr/local/Cellar/nim/1.0.0/nim/lib/system/fatal.nim(39) sysFatal
Error: unhandled exception: index out of bounds, the container is empty [IndexError]
```

#### page 42

```
assert list[0] == nil
```

This construct now seems to be deprecated for strings. I get this compile-time error:

```
seq.nim(3, 16) Error: 'nil' is now invalid for 'string'; compile with --nilseqs:on for a migration period; usage of '==' is a user-defined error
```

Changing this to...

```
assert list[0] == ""
```

...works. So strings are now automatically initialised to "" rather than nil.