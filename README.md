# ECMAScript proposal: support sequence properties in Unicode property escapes

## Status

This proposal is at stage 2 of [the TC39 process](https://tc39.es/process-document/).

## Motivation

The Unicode Standard assigns various properties and property values to every symbol. For example, to get the set of symbols that are used exclusively in the Greek script, search the Unicode database for symbols whose `Script` property is set to `Greek`.

[Unicode property escapes](https://github.com/tc39/proposal-regexp-unicode-property-escapes) enable JavaScript developers to access these Unicode character properties natively in ECMAScript regular expressions.

```js
const regexGreekSymbol = /\p{Script=Greek}/u;
regexGreekSymbol.test('π');
// → true
```

The Unicode properties and values that are [currently](https://tc39.es/ecma262/#table-nonbinary-unicode-properties) [supported](https://tc39.es/ecma262/#table-binary-unicode-properties) in Unicode property escapes have something in common: they all expand to **a list of code points**. Such escapes can be transpiled as a character class containing the list of code points they match individually. For example, `\p{ASCII_Hex_Digit}` is equivalent to `[0-9A-Fa-f]`: it only ever matches a single Unicode symbol at a time.

However, the Unicode Standard [defines properties that instead expand to **a list of _sequences_ of code points**](https://unicode.org/reports/tr18/proposed.html#Categories). In regular expressions, such properties translate to a set of alternatives. To illustrate this, imagine a Unicode property that expands to the Unicode code point sequences `'a'`, `'mn'`, and `'xyz'`. This property translates to the following regular expression pattern: `a|mn|xyz`. Note how unlike existing Unicode property escapes, this pattern can match multiple Unicode symbols.

Hand-written regular expressions for these properties suffer from [the same issues that Unicode property escapes solve](https://github.com/tc39/proposal-regexp-unicode-sequence-properties#motivation): they’re hard to write or maintain manually, they tend to be large, and they’re unreadable.

## Proposed solution

We propose the addition of _Unicode sequence properties_ to the existing Unicode property escapes syntax.

With this feature, the above regular expression could be written as:

```js
const re = /\p{RGI_Emoji_ZWJ_Sequence}/u;
re.test('👨🏾‍⚕️'); // '\u{1F468}\u{1F3FE}\u200D\u2695\uFE0F'
// → true
```

We propose to support the following Unicode sequence properties defined in [UTS18](https://unicode.org/reports/tr18/proposed.html#Full_Properties) and [UTS51](https://unicode.org/reports/tr51/):

- `Basic_Emoji`
- `RGI_Emoji_Modifier_Sequence`
- `RGI_Emoji_Tag_Sequence`
- `RGI_Emoji_ZWJ_Sequence`
- `RGI_Emoji`

Each of these sequence properties expands to a finite, well-defined set of strings.

Over time, we can choose to support additional sequence properties, following the upstream Unicode Standard.

## High-level API

Re-using the existing Unicode property escapes syntax for this new functionality seems appropriate:

<pre>\p{<b><i>UnicodeSequencePropertyName</i></b>}</pre>

The negated `\P{…}` form is not supported for sequence properties as it would be [a footgun](https://github.com/tc39/proposal-regexp-unicode-sequence-properties/issues/6#issuecomment-368460069). It’s not generally useful, and is better expressed as a negative lookahead. Compare the unsupported `/\P{UnicodeSequenceProperty}/u` (what should it do?) with `/(?!\p{UnicodeSequenceProperty})/u` (clear what it does).

Given that `UnicodeSequencePropertyName` expands to a list of sequences of Unicode code points, the proposal also includes a static restriction that bans such properties within character classes.

### FAQ

#### What about backwards compatibility?

Unicode property escapes for unsupported Unicode properties throw an early `SyntaxError`. As such, we can add support for new properties in a backwards-compatible way, as long as we re-use the existing syntax.

#### Why ban the use of these properties within character classes?

Currently, each property escape expand to a list of code points. As such, their meaning is clear and unambiguous, even within a character class. For example, the following regular expression matches either a `Letter`, a `Number`, or an underscore:

```js
const re = /[\p{Letter}\p{Number}_]/u;
```

For the new properties introduced by this proposal, the expected behavior within character classes is unclear. A character class, when matched, always produces only a single character. Allowing sequence properties within character classes would change that, for no good reason.

```js
const re = /[\p{RGI_Emoji_ZWJ_Sequence}_a-z]/u;
// 🤔 What should this do?

// If the goal is to match either `\p{RGI_Emoji_ZWJ_Sequence}` or `_` or
// `[a-z]`, one could still use `|`:
const re = /\p{RGI_Emoji_ZWJ_Sequence}|[a-z_]/u;
```

To avoid confusion, the proposal throws a `SyntaxError` exception when sequence properties are used within character classes.

#### Why re-use `\p{…}` and not introduce new syntax?

Introducing new syntax comes at a cost for JavaScript developers. In this case, we assert that the cost of adding new syntax for this functionality outweighs the benefits.

New syntax would have the benefit of making the distinction between binary and regular properties vs. sequence properties more clearly.

The mental model is: `\p{…}` refers to a Unicode property. This proposal doesn’t change that. It’s reasonable to assume that developers opting in to the use of sequence properties know what to expect.

## Illustrative examples

### Matching emoji sequences

With this proposal, [the set of RGI (“recommended for general interchange”) emoji](https://unicode.org/reports/tr51/#def_rgi_set) (characters _and_ sequences!) can be trivially represented as a RegExp pattern in JavaScript:

```js
const reRgiEmoji = /\p{RGI_Emoji}/u;
```

[An equivalent regular expression](https://github.com/mathiasbynens/emoji-regex) without the use of property escapes is ~7 KB in size. With property escapes, but without sequence property support, the size is still ~4.5 KB. The abovementioned regular expression with sequence properties takes up 16 bytes.

### Matching hashtags

Unicode® Standard Annex #31 defines [hashtag identifiers](https://unicode.org/reports/tr31/#hashtag_identifiers) in two forms.

The _Default Hashtag Identifier Syntax (UAX31-D2)_ translates to the following JavaScript regular expression:

```js
const reHashtag = /[#\uFE5F\uFF03]\p{XID_Continue}+/u;
```

However, the _Extended Hashtag Identifier Syntax (UAX31-R8)_ currently cannot trivially be expressed as a JavaScript regular expression, as it includes emoji. An approximation without emoji sequence support would be:

```js
// This matches *some* emoji, but not those consisting of sequences.
const reHashtag = /[#\uFE5F\uFF03][\p{XID_Continue}_\p{Emoji}]+/u;
```

The above pattern matches *some* emoji, but not those consisting of sequences. It would also match emoji that render as text by default. With the proposed feature however, fully implementing the UAX31-R8 syntax becomes feasible:

```js
const reHashtag = /[#\uFE5F\uFF03](?:[\p{XID_Continue}_]|\p{RGI_Emoji})+/u;
```

[An equivalent regular expression](https://github.com/mathiasbynens/hashtag-regex) without the use of property escapes is ~12 KB in size. With property escapes, but without sequence property support, the size is still ~3 KB. The abovementioned regular expression with sequence properties takes up 50 bytes.

## Related UTC proposals

- [L2/18-337 Broaden the scope of what Unicode calls “properties”](https://unicode.org/L2/L2018/18337-broaden-properties.pdf)
    - [L2/19-056 Comments on L2/18-337](https://unicode.org/L2/L2019/19056-prop-cmts.pdf)
        - [Response to L2/19-056](https://docs.google.com/document/d/1YW_4PTRUYz0mRvWmXwtUSd6jaG-lDQnfvs5gpKuix5E/edit)
- [L2/19-168 Supporting string properties in regular expressions](https://unicode.org/L2/L2019/19168-regex-string-prop.pdf) ([slides with speaker notes](https://docs.google.com/presentation/d/1KtrP7TWIX9XJ_K-3zYyEvpSIKhl5PscG6lgMDyizRAU/edit))
- [Working Draft #20 of Unicode® Technical Standard #18](https://unicode.org/L2/L2019/19249-wd-uts18-20-draft.pdf)
- [Presentation by Mark Davis](https://docs.google.com/presentation/d/1wM3L5f3HF8nkCkGY0SR_gT_HvmkDO0VLxcuhjQCJUtc/edit)
- [L2/20-056: UTS #18 regex ad hoc: `\p` vs. `\m` for properties of strings](https://unicode.org/L2/L2020/20056-regex-adhoc-rept.pdf)

## TC39 meeting notes

- [May 2018](https://tc39.es/tc39-notes/2018-05_may-22.html#11ia-sequence-properties-in-unicode-property-escapes)
- [September 2018](https://tc39.es/tc39-notes/2018-09_sept-26.html#sequence-properties-in-unicode-property-escapes-for-stage-2)
- [January 2019](https://github.com/tc39/tc39-notes/blob/def2ee0c04bc91612576237314a4f3b1fe2edaef/meetings/2019-01/jan-31.md#update-on-sequence-properties-in-unicode-property-escapes)
- October 2019

## Specification

* [Ecmarkup source](https://github.com/tc39/proposal-regexp-unicode-sequence-properties/blob/master/spec.html)
* [HTML version](https://tc39.es/proposal-regexp-unicode-sequence-properties/)

## Implementations

* none yet
