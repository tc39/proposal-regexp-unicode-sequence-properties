# ECMAScript proposal: support sequence properties in Unicode property escapes

## Status

This proposal is at stage 0 of [the TC39 process](https://tc39.github.io/process-document/).

## Motivation

The Unicode Standard assigns various properties and property values to every symbol. For example, to get the set of symbols that are used exclusively in the Greek script, search the Unicode database for symbols whose `Script` property is set to `Greek`.

[Unicode property escapes](https://github.com/tc39/proposal-regexp-unicode-sequence-properties) enable JavaScript developers to access these Unicode character properties natively in ECMAScript regular expressions.

```js
const regexGreekSymbol = /\p{Script=Greek}/u;
regexGreekSymbol.test('π');
// → true
```

The Unicode properties and values that are [currently](https://tc39.github.io/ecma262/#table-nonbinary-unicode-properties) [supported](https://tc39.github.io/ecma262/#table-binary-unicode-properties) in Unicode property escapes have something in common: they all expand to **a list of code points**. Such escapes can be transpiled as a character class containing the list of code points they match individually. For example, `\p{ASCII_Hex_Digit}` is equivalent to `[0-9A-Fa-f]`: it only ever matches a single Unicode symbol at a time.

However, the Unicode Standard defines properties that instead expand to **a list of _sequences_ of code points**. In regular expressions, such properties translate to a set of alternatives. To illustrate this, imagine a Unicode property that expands to the Unicode code point sequences `'a'`, `'mn'`, and `'xyz'`. This property translates to the following regular expression pattern: `a|mn|xyz`. Note how unlike existing Unicode property escapes, this pattern can match multiple Unicode symbols.

A minimal actual example of such a Unicode property is [`Emoji_Keycap_Sequence`](https://unicode.org/reports/tr51/). To represent this property in a regular expression, one could use the pattern `\x23\uFE0F\u20E3|\x2A\uFE0F\u20E3|\x30\uFE0F\u20E3|\x31\uFE0F\u20E3|\x32\uFE0F\u20E3|\x33\uFE0F\u20E3|\x34\uFE0F\u20E3|\x35\uFE0F\u20E3|\x36\uFE0F\u20E3|\x37\uFE0F\u20E3|\x38\uFE0F\u20E3|\x39\uFE0F\u20E3`. Regular expressions for these properties suffer from [the same issues that Unicode property escapes solve](https://github.com/tc39/proposal-regexp-unicode-sequence-properties#motivation): they’re hard to write or maintain manually, they tend to be large, and they’re unreadable. (The `Emoji_Keycap_Sequence` pattern in particular can be simplified as `[\x23\x2A0-9]\uFE0F\u20E3`, but even in that form it’s hard to decipher.)

## Proposed solution

We propose the addition of _Unicode sequence properties_ to the existing Unicode property escapes syntax.

With this feature, the above regular expression could be written as:

```js
const regexEmojiKeycap = /\p{Emoji_Keycap_Sequence}/u;
regexEmojiKeycap.test('4️⃣');
// → true
```

We propose to support the following Unicode sequence properties defined in [Unicode TR51](https://unicode.org/reports/tr51/):

- `Emoji_Combining_Sequence`
- `Emoji_Flag_Sequence`
- `Emoji_Keycap_Sequence`
- `Emoji_Modifier_Sequence`
- `Emoji_Tag_Sequence`
- `Emoji_ZWJ_Sequence`

## High-level API

Re-using the existing Unicode property escapes syntax for this new functionality seems appropriate:

<pre>\p{<b><i>UnicodeSequencePropertyName</i></b>}</pre>

As before, `\P{…}` is the negated form of `\p{…}`.

Given that `UnicodeSequencePropertyName` expands to a list of sequences of Unicode code points, the proposal includes a static restriction that bans such properties within character classes.

### FAQ

#### What about backwards compatibility?

Unicode property escapes for unsupported Unicode properties throw an early `SyntaxError`. As such, we can add support for new properties in a backwards-compatible way, as long as we re-use the existing syntax.

#### Why ban the use of these properties within character classes?

Currently, each property escape expand to a list of code points. As such, their meaning is clear and unambiguous, even within a character class. For example, the following regular expression matches either a Letter, a Number, or an underscore:

```js
const re = /[\p{Letter}\p{Number}_]u;
```

For the new properties introduced by this proposal, the expected behavior within character classes is unclear. A character class, when matched, always produces only a single character. Allowing sequence properties within character classes would change that, for no good reason.

```js
const re = /[\p{Emoji_Flag_Sequence}_a-z]u;
// 🤔 What should this do?

// If the goal is to match either `\p{Emoji_Flag_Sequence}` or `_` or
// `[a-z]`, one could still use `|`:
const re = /\p{Emoji_Flag_Sequence}|[a-z_]/u;
```

To avoid confusion, the proposal throws a `SyntaxError` exception when sequence properties are used within character classes.

## Illustrative examples

### Matching emoji sequences

Per [UTR51 ED-26](https://unicode.org/reports/tr51/proposed.html#def_rgi_sequences), the term “emoji sequences” refers to [emoji flag sequences](https://unicode.org/Public/emoji/latest/emoji-sequences.txt), [emoji tag sequences](https://unicode.org/Public/emoji/latest/emoji-sequences.txt), and [emoji ZWJ sequences](https://unicode.org/Public/emoji/latest/emoji-zwj-sequences.txt). With this proposal, emoji sequences can be represented as a RegExp pattern in JavaScript:

```js
const reEmojiSequence = /\p{Emoji_Flag_Sequence}|\p{Emoji_Tag_Sequence}|\p{Emoji_ZWJ_Sequence}/u;
```

### Matching all emoji (including emoji sequences)

This proposal makes it possible to match all emoji, regardless of whether they consist of sequences or not:

```js
const reEmoji = /\p{Emoji_Modifier_Base}\p{Emoji_Modifier}?|\p{Emoji_Presentation}|\p{Emoji}\uFE0F|\p{Emoji_Flag_Sequence}|\p{Emoji_Tag_Sequence}|\p{Emoji_ZWJ_Sequence}/gu;
```

This regular expression matches, from left to right:

1. emoji with optional modifiers (`\p{Emoji_Modifier_Base}\p{Emoji_Modifier}?` per [ED-13](https://unicode.org/reports/tr51/proposed.html#def_emoji_modifier_sequence));
2. any remaining symbols that render as emoji rather than text by default (`\p{Emoji_Presentation}` per [ED-6](https://unicode.org/reports/tr51/proposed.html#def_emoji_presentation));
3. symbols that render as text by default, but are forced to render as emoji using U+FE0F VARIATION SELECTOR-16 (`\p{Emoji}\uFE0F` per [ED-9a](https://unicode.org/reports/tr51/proposed.html#def_emoji_presentation_sequence));
4. emoji sequences (`\p{Emoji_Flag_Sequence}|\p{Emoji_Tag_Sequence}|\p{Emoji_ZWJ_Sequence}`, [as discussed above](#matching-emoji-sequences)).

[An equivalent regular expression](https://github.com/mathiasbynens/emoji-regex) without the use of property escapes is ~7 KB in size. With property escapes, but without sequence property support, the size is still ~4.5 KB. The abovementioned regular expression with sequence properties takes up 155 bytes.

### Matching hashtags

Unicode® Standard Annex #31 defines [hashtag identifiers](https://unicode.org/reports/tr31/#hashtag_identifiers) in two forms.

The _Default Hashtag Identifier Syntax (UAX31-D2)_ translates to the following JavaScript regular expression:

```js
const reHashtag = /[#\uFF03]\p{XID_Continue}+/u;
```

However, the _Extended Hashtag Identifier Syntax (UAX31-R8)_ currently cannot trivially be expressed as a JavaScript regular expression, as it includes emoji. An approximation without emoji sequence support would be:

```js
// This matches *some* emoji, but not those consisting of sequences.
const reHashtag = /[#\uFF03][\p{XID_Continue}_\p{Emoji}]+/u;
```

The above pattern matches *some* emoji, but not those consisting of sequences. With the proposed feature however, fully implementing the UAX31-R8 syntax becomes feasible:

```js
const reHashtag = /[#\uFF03](?:[\p{XID_Continue}_\p{Emoji}]|\p{Emoji_Flag_Sequence}|\p{Emoji_Tag_Sequence}|\p{Emoji_ZWJ_Sequence})+/u;
```

[An equivalent regular expression](https://github.com/mathiasbynens/hashtag-regex) without the use of property escapes is ~12 KB in size. With property escapes, but without sequence property support, the size is still ~3 KB. The abovementioned regular expression with sequence properties takes up 115 bytes.

## TC39 meeting notes

- TODO

## Specification

* [Ecmarkup source](https://github.com/mathiasbynens/proposal-regexp-unicode-sequence-properties/blob/master/spec.html)
* [HTML version](https://mathiasbynens.github.io/proposal-regexp-unicode-sequence-properties/)

## Implementations

* none yet
