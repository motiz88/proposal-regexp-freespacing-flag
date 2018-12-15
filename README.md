
# ECMAScript regular expression `/x` flag proposal

This is a proposal to add the `/x` flag to ECMAScript regular expressions, enabling free spacing, line comments and embedded line breaks within regular expressions - notably, within regular expression *literals*. It's not yet ready to be presented at TC39, but hopefully some day it could be.

> **Note**: This is [not the first time](https://esdiscuss.org/topic/regexp-free-spacing-comments) this problem has been discussed, though I haven't seen other developed proposals in this space. Also, it is possible that I'm missing something fundamental as to why line breaks simply cannot be allowed in regular expression literals, but I haven't personally seen this idea refuted directly. If you know otherwise, I'd love to hear about it!

## Motivation

[![xkcd 1171: Perl Problems](https://imgs.xkcd.com/comics/perl_problems_2x.png)](https://xkcd.com/1171/)

Regular expressions are useful<sup>\[citation needed\]</sup>. Used correctly, they provide a succinct and performant method of working with string inputs. _Complex_ regular expressions are also useful, but they have a readability problem that gets worse the longer they get. This can make code that uses them harder to maintain.

User-space solutions are possible; one notable library in this space is [XRegExp](http://xregexp.com/) with its Perl-inspired [`x` flag](http://xregexp.com/flags/#extended). However, runtime and code size concerns can dissuade people from adding a library to their programs to provide what is essentially a "developer convenience" feature.

Moreover, without language-level support, it is harder and less sound to build tooling around free-spaced regular expressions. Examples of potentially-useful tooling include:

* Syntax highlighting;
* A lint rule that enforces a complexity limit for single-line patterns, with free-spaced patterns as the recommended alternative;
* A code formatter that rewrites single-line patterns into free-spaced patterns automatically, and manages spacing and indentation within free-spaced patterns for consistency.

## Syntax

```
var pattern = /
  (?<year>  [0-9]{4} ) -?  # year
  (?<month> [0-9]{2} ) -?  # month
  (?<day>   [0-9]{2} )     # day
/x;
```

[_RegularExpressionNonTerminator_](https://tc39.github.io/ecma262/#prod-RegularExpressionNonTerminator) is redefined here as equivalent to [_SourceCharacter_](https://tc39.github.io/ecma262/#prod-SourceCharacter) (vs the current "_SourceCharacter_ but not [_LineTerminator_](https://tc39.github.io/ecma262/#prod-LineTerminator)").

It is a syntax error if a regular expression's body text contains a _LineTerminator_ but its flag text doesn't contain `x`.

### Note on feasibility

I believe these syntax changes are safe and strictly additive, in two senses:

1. Line terminators within regex literals are currently syntax errors (with ECMAScript implementations being explicitly [forbidden](https://tc39.github.io/ecma262/#sec-literals-regular-expression-literals) from changing this). Changing this can however cause _invalid_ code to take longer to reach a parse error (e.g. a single unmatched `/` that extends until the end of the program, or a matched pair across lines that doesn't specify the `x` flag).
2. To my knowledge, there is no potential for conflict with the `/` operator in well-formed code, because binary operators and regular expresion literals cannot appear in the same syntactic positions.

## Semantics

Patterns with the `x` flag are to be interpreted exactly as XRegExp interprets them:

> This flag has two complementary effects. First, it causes most whitespace to be ignored, so you can free-format the regex pattern for readability. Second, it allows comments with a leading `#`. Specifically, it turns most whitespace into an "ignore me" metacharacter, and `#` into an "ignore me, and everything else up to the next newline" metacharacter. They aren't taken as metacharacters within character classes (which means that classes are not free-format, even with `x`), and as with other metacharacters, you can escape whitespace and `#` that you want to be taken literally. Of course, you can always use `\s` to match whitespace.

See more at http://xregexp.com/flags/#extended.

## Alternatives

* As mentioned previously, XRegExp is a popular user-space solution that works today. The drawbacks of a user-space-only solution are discussed in the [Motivation](#motivation) section.
* Arguments could be made for deviating from XRegExp's `x` syntax in various ways, for example supporting embedded `//` and `/* ... */` comments instead of the current `#`. Counterarguments would include:
	* Added complexity - for example, practically anything to do with `/` would risk changing the meaning of existing programs;
	* Fragmentation relative to the established solution;
	* The fact that regular expressions already form their own distinct syntax within the language, so a different syntax for comments in regex is somewhat justifiable.
* There is prior art ([\[1\]](http://2ality.com/2017/07/re-template-tag.html), [\[2\]](http://lea.verou.me/2018/06/easy-dynamic-regular-expressions-with-tagged-template-literals-and-proxies/), probably more) for using tagged template literals as an alternative to regular expression literals. Template literals already accommodate line breaks, and so can be used to achieve something similar without any changes to the ECMAScript grammar. Problems with this as a standard feature include:
	* Embedded expressions introduce dynamism, which can tempt authors to write less-performant code;
	* The question of whether embedded strings should get auto-escaped (vs being treated as raw regex snippets) adds complexity and can cause confusion;
	* It's also probably a reasonable expectation that new regex features _have_ counterparts in the literal syntax, unless we take the more extreme step of deprecating regex literals outright.
