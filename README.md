# ECMAScript proposal: support properties of strings (a.k.a. “sequence properties”) in Unicode property escapes

## Status

This proposal is at stage 2 of [the TC39 process](https://tc39.es/process-document/).

Note that the [RegExp `v` flag proposal](https://github.com/tc39/proposal-regexp-v-flag) subsumes this proposal – and also adds set notation & string literals to character classes.

## Terminology

This proposal initially used the term “sequence properties”, but that is a misnomer. A sequence of characters is a string, and a string property is one whose *values* (the codomain) are strings, just like a binary property is one whose values are binary true/false (that is, whether the property applies or does not apply).

Unicode has since formalized this, using “property of code points” vs. “property of strings” for the *domain* of a property. See <https://www.unicode.org/reports/tr18/#domain_of_properties>.

Also, we mostly use “character” and “code point” interchangeably. More formally, “character” refers to assigned code points, but properties have values for all code points. (Most properties map all unassigned code points to one default value.)

## Motivation

The Unicode Standard assigns various properties and property values to every character/code point. For example, the Unicode Character Database provides data for determining exactly the set of characters whose `Script` property value is `Greek`.

[Unicode property escapes](https://github.com/tc39/proposal-regexp-unicode-property-escapes) enable JavaScript developers to access these Unicode character properties natively in ECMAScript regular expressions.

```js
const regexGreek = /\p{Script=Greek}/u;
regexGreek.test('π');
// → true
```

The Unicode properties and values that are [currently](https://tc39.es/ecma262/#table-nonbinary-unicode-properties) [supported](https://tc39.es/ecma262/#table-binary-unicode-properties) in Unicode property escapes have something in common: they all expand to **a set of code points**. Such escapes can be transpiled as a character class containing the code points they match individually. For example, `\p{ASCII_Hex_Digit}` is equivalent to `[0-9A-Fa-f]`: it only ever matches a single Unicode character/code point at a time.

However, the Unicode Standard also defines several properties of strings. In regular expressions, such properties translate to a set of alternatives. To illustrate this, imagine a Unicode property that applies to the strings `'a'`, `'b'`, `'c'`, `'W'`, `'xy'`, and `'xyz'`. This property translates to either of the following regular expression patterns (using alternation): `xyz|xy|a|b|c|W` or `xyz|xy|[a-cW]`. (Longest strings first, so that a prefix like `'xy'` does not hide a longer string like `'xyz'`.) Note how unlike existing Unicode property escapes, this pattern can match multi-character strings.

Hand-written regular expressions for these properties suffer from [the same issues that Unicode property escapes solve](https://github.com/tc39/proposal-regexp-unicode-property-escapes#motivation): they’re hard to write or maintain manually, they tend to be large, and they’re unreadable.

## Proposed solution

We propose the addition of several _properties of strings_ to the existing Unicode property escapes syntax.

With this feature, the above regular expression could be written as:

```js
const re = /\p{RGI_Emoji_ZWJ_Sequence}/u;
re.test('👨🏾‍⚕️'); // '\u{1F468}\u{1F3FE}\u200D\u2695\uFE0F'
// → true
```

We propose to support the following Unicode sequence properties defined in [UTS18](https://www.unicode.org/reports/tr18/#Full_Properties) and [UTS51](https://www.unicode.org/reports/tr51/):

- `Basic_Emoji`
- `Emoji_Keycap_Sequence`
- `RGI_Emoji_Modifier_Sequence`
- `RGI_Emoji_Flag_Sequence`
- `RGI_Emoji_Tag_Sequence`
- `RGI_Emoji_ZWJ_Sequence`
- `RGI_Emoji`

Each of these sequence properties expands to a finite, well-defined set of strings. (`Basic_Emoji` also applies to many single characters.)

Over time, we can choose to support additional properties of strings, following the upstream Unicode Standard.

## High-level API

Re-using the existing Unicode property escapes syntax for this new functionality seems appropriate:

<pre>\p{<b><i>PropertyName</i></b>}</pre>

Where `PropertyName` can be one of the properties of strings listed above.

The complement of such a property is not supported: both `\P{PropertyName}` and `[^…\p{PropertyName}…]` throw an early `SyntaxError` exception if `PropertyName` is a property of strings.

We have thought of possible definitions of such a complement, but we believe that they are not generally useful.

Some of the use cases for “not a property of strings” can be supported via a negative lookahead: `/(?!\p{RGI_Emoji_Flag_Sequence})\p{Symbol}/u`.

**Note:** Using a property of strings inside a character class is equivalent to an alternation of all of the strings and characters, such that the order of elements is irrelevant (e.g., listing the strings longest-first). (This could be optimized by retaining a character class of the single characters,
as illustrated in the Motivation section above.)

### FAQ

#### What about backwards compatibility?

Unicode property escapes for unsupported Unicode properties throw an early `SyntaxError`. As such, we can add support for new properties in a backwards-compatible way, as long as we re-use the existing syntax.

#### Properties of strings within character classes

Currently, each property escape and character class expands to a set of code points, equivalent to an alternation of single characters. With this proposal, a property escape and character class expands to a set of strings, equivalent to an alternation of strings. In most cases, most or all of those strings will still be single-character strings.

For example: `[\p{Emoji_Keycap_Sequence}\p{Symbol}]` = `#⃣|*⃣|0⃣|1⃣|…|9⃣|[\$+<->\^…℻⅀-⅄⅊-⅍…]`

#### Why re-use `\p{…}` and not introduce new syntax?

Introducing new syntax comes at a cost for JavaScript developers. In this case, we assert that the cost of adding new syntax for this functionality outweighs the benefits.

New syntax *could* be used for properties of strings. However, such new syntax should also allow for properties of code points, so that, when a Unicode property no longer applies to multi-character strings in a later Unicode version, existing regular expressions remain valid.

Therefore, developers would be expected to know which property does, or did at one point, apply to strings, but it would be easier for them to simply switch to the new syntax for all properties.

Regular expressions can be validated by a parser using information about which property applies to strings vs. only single characters, without need for a new escape.

The mental model is: `\p{…}` refers to a Unicode property. It matches the elements of the property’s domain for which its value is true. This proposal doesn’t change that. It’s reasonable to assume that developers opting in to the use of properties of strings know what to expect.

## Illustrative examples

### Matching emoji sequences

With this proposal, [the set of RGI (“recommended for general interchange”) emoji](https://unicode.org/reports/tr51/#def_rgi_set)
(characters _and_ sequences!) can be trivially represented as a RegExp pattern in JavaScript:

```js
const reRgiEmoji = /\p{RGI_Emoji}/u;
```

[An equivalent regular expression](https://github.com/mathiasbynens/emoji-regex) without the use of property escapes is ~7 kB in size. With property escapes, but without support for properties of strings, the size is still ~4.5 kB. The abovementioned regular expression with sequence properties takes up 16 bytes.

### Matching hashtags

Many applications (such as Twitter) use extended hashtags that allow for emoji characters. Unicode® Standard Annex \#31 defines [_Extended Hashtag Identifier Syntax (UAX31-R8)_](https://www.unicode.org/reports/tr31/#R8) as matching:

```
// From UAX #31, not in JavaScript syntax.
/[#﹟＃][\p{XID_Continue}\p{Extended_Pictographic}\p{Emoji_Component}[-+_]-[#﹟＃]]+/
```

The above pattern matches emoji, but also *syntactically invalid emoji* as well as emoji that are *not recommended* for general interchange. With the proposed feature however, matching hashtags with only valid and recommended emoji becomes feasible:

```js
const reHashtag = /[#﹟＃][[\p{XID_Continue}\p{RGI_Emoji}[-+_]]--[#﹟＃]]+/u;
```

[An equivalent regular expression](https://github.com/mathiasbynens/hashtag-regex) without the use of property escapes is ~12 kB in size. With property escapes, but without support for properties of strings, the size is still ~3 kB. The abovementioned regular expression with sequence properties takes up 62 bytes.

## Related UTC proposals

- [L2/18-337 Broaden the scope of what Unicode calls “properties”](https://unicode.org/L2/L2018/18337-broaden-properties.pdf)
    - [L2/19-056 Comments on L2/18-337](https://unicode.org/L2/L2019/19056-prop-cmts.pdf)
        - [Response to L2/19-056](https://docs.google.com/document/d/1YW_4PTRUYz0mRvWmXwtUSd6jaG-lDQnfvs5gpKuix5E/edit)
- [L2/19-168 Supporting string properties in regular expressions](https://unicode.org/L2/L2019/19168-regex-string-prop.pdf) ([slides with speaker notes](https://docs.google.com/presentation/d/1KtrP7TWIX9XJ_K-3zYyEvpSIKhl5PscG6lgMDyizRAU/edit))
- [Working Draft #20 of Unicode® Technical Standard #18](https://unicode.org/L2/L2019/19249-wd-uts18-20-draft.pdf)
- [Presentation by Mark Davis](https://docs.google.com/presentation/d/1wM3L5f3HF8nkCkGY0SR_gT_HvmkDO0VLxcuhjQCJUtc/edit)
- [L2/20-056: UTS #18 regex ad hoc: `\p` vs. `\m` for properties of strings](https://unicode.org/L2/L2020/20056-regex-adhoc-rept.pdf)

## TC39 meeting notes

- [May 2018](https://github.com/tc39/notes/blob/master/meetings/2018-05/may-22.md#11ia-sequence-properties-in-unicode-property-escapes)
- [September 2018](https://github.com/tc39/notes/blob/master/meetings/2018-09/sept-26.md#sequence-properties-in-unicode-property-escapes-for-stage-2)
- [January 2019](https://github.com/tc39/notes/blob/master/meetings/2019-01/jan-31.md#update-on-sequence-properties-in-unicode-property-escapes)
- October 2019:
    - [discussion 1](https://github.com/tc39/notes/blob/master/meetings/2019-10/october-2.md#update-on-sequence-property-escapes-in-unicode-regular-expressions)
    - [discussion 2](https://github.com/tc39/notes/blob/master/meetings/2019-10/october-3.md#redux-update-on-sequence-property-escapes-in-unicode-regular-expressions)

## Specification

* [Ecmarkup source](https://github.com/tc39/proposal-regexp-unicode-sequence-properties/blob/master/spec.html)
* [HTML version](https://tc39.es/proposal-regexp-unicode-sequence-properties/)

## Implementations

* none yet
