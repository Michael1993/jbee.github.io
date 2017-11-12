---
layout: default
title: "HiperX"
categories: [project, parsing]
---

# *Hi*gh *per*formance e*X*pressions

* spec
* pattern matching
* parsing
* terminals

To implement a general parser a language designer
wants to describe patterns for terminals.

Let us understand a *terminal* as any sequence of 
characters that is one atomic thing. 
A leaf in a parse tree.
Like identifiers, numbers or string literals. 

> Question: Does the input start with a pattern that 
> makes it a certain terminal and where does it end?

With the goal of implementing the full parser in very
little code regular expressions are not an option. 
Besides, they are a complex hard to predict general tool.
A specific solution can be simpler and predictable for 
a language designer.


## Design
Goal is the minimal set of features that allows to
define intuitive patterns to match sophisticated 
terminals.

The algorithm should have a single allocation-free 
interpreter matching function.

Properties that can be tested on identified terminals,
like a maximum length, are not essential for matching
and thus require no support.


## Rules
Rules are byte instructions designed to give the 
appearance of syntax, but there is none.
It is a byte-encoded interpreted language.

Character sets:

* `{abc}`: a set of bytes `a`,`b` and `c`
* `{^abc}`: a set of any byte but `a`,`b` and `c`
* `{a-c}`: a set of `a`, `b` and `c` given as a range
* `<a-c>`: range of `a` to `c` and all non ASCII bytes
* `#` = `{0-9}`: a ASCII digit
* `@` = `{a-zA-Z}`: a ASCII letter
* `^`: any byte that is not a ASCII whitespace character
* `_`: any single byte
* `$`: any non ASCII byte (`1xxx xxxx`)

All bytes within a set `{`...`}` are literal.
Hence, `}` cannot be included in a set explicitly.
`{^}` matches `^` while any set starting with `^` 
defining further members is exclusive.

Repetition:

* `+`: previous set, group or literal more than once

Groups:

* `(abc)`: a group `abc` that must occur
* `[abc]`: a group `abc` that can occur

Groups are most useful when nested, like `(a[b(c)+])`.
The regex `*` can be build using `[abc]+`.

Scanning:

* `~`: skip until following set, group or literal matches

Any other byte is matched literally. Sets can be used
to match any of the rule symbols literally, like `{~}`.

UTF-8 can be matched literally but not in a set.
Such a support would be possible but is not added to
keep the matcher unaware of encoding as this is almost
never needed in actual languages.


## Principles and Properties

* a match is always sufficient 
* `+` is always greedy (stops on first mismatch)
* `~` is always non-greedy (stops on first match)
* there is no backtracking
* sets are limited to ASCII (a single byte)

Consequently the parser must make progress either in
input or pattern.
Otherwise a mismatch has been found at the current
input position.

The worst cases are scanning `~` and optional groups `[...]`
that try to match and otherwise recover by making progress
in either the input (`~`) or the pattern (`[...]`).
In all other cases progress is always made in both.
Almost always mismatches are identified immediately.

The computational complexity is O(n). 
In worst case n is the longer length (of input or pattern).
In best case n is the shorter length (of input or pattern).
No heap memory is needed to process matching. 


## Examples
Following gives examples of a pattern an the inputs it
matches.

Dates: 

* `####/##/##`: *yyyy/mm/dd*
* `##[##]/##/##`: *yy/mm/dd* or *yyyy/mm/dd*
* `##{/-.}##{/-.}##`: *dd/mm/yy*, *dd-mm-yy* or *dd.mm.yy*
* `#[#]/#[#]/##[##]`: *d/m/yy*, *dd/mm/yy*, *d/m/yyyy* or *dd/mm/yyyy*

Times:

* `##:##`: *hh:mm*
* `##:##[:##]` : *hh:mm* or *hh:mm:ss*
* `##:##:##[.###]`: *hh:mm:ss* or *hh:mm:ss.SSS* (ms)
* `##[:##][:##].###`: *hh:mm:ss.SSS*, *mm:ss.SSS* or *ss.SSS*

Numbers:

* `#+`: simple integers; `1`, `42`, `345`, ...
* `#+[,###]+`: with dividers; `42`, `42,000`, `42,000,000`, ...
* `#+[.#+]`: simple floating points; `0.2`, `10`, `12.45`, ...
* `#+[{_#}+#][{lL}]`: java integer literals; `5L`, `5_3`, `11__12`,
* `0b{01}+[_+{01}+]+`: java binary literal; `0b0000_1010`, ...
* `0x{0-9A-Fa-f}+[_+{0-9A-Fa-f}+]+`: java hex literals; `0xCAFE_BABE`, ...

Strings:

* `"{^"}+"`: quoted string (no escaping); `"foo"`, `"1"`, ...
* `"~"`: quoted string (no escaping); `"foo"`, `"1"`, ...
* `"~({^\}")`: quoted string (escaping); `"foo"`, `"foo\"bar"`, ... 
* `"""~("""")`: triple quoted string; `"""foo"""`, `"""foo"bar"baz"""`, ...

Identifiers:

* `@+[{-_}{a-zA-Z0-9}+]+`: general letters and digits; `foo`, `Foo`, `foo-bar`, `fooBar`, `foo_bar`, `foo1`, `foo2bar`, ...
* `{$}@<a-zA-Z0-9_>`: php style; `$foo`, `$föö`, `$FOO2`, `$foo_bar`

Instead of defining multiple cases for complex patterns 
a common pattern must be found. 
To match for example any java number literal the pattern could be:

```
{.0-9}[{.xb0-9}[{0-9A-Fa-f_}+][.#+]][{dDfFlL}]
```
This would wrongly match just `.` or `..` or `..0.0` or
hexadecimal numbers with a floating point tail.
As this still would not accept something as a number
that is another valid token this is less of a problem.
The number tokens would need a additional check at a 
later stage, similar to a overall length check.

A better solution, however, is to design the language
so that such irregularities do not occur. In case of
java's numbers we could disallow a floating point to 
start with `.`.
This limitation is less considered a problem and more a
help to keep a language regular. 
In the example we would gain the property of all number
literals starting with a digit.


