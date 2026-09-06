---
title: "CBOR Pointer: Selecting Elements of Concise Binary Object Representation (CBOR) Documents"
abbrev: "CBOR Pointer"
category: info

docname: draft-mahy-cbor-pointer-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "Concise Binary Object Representation Maintenance and Extensions"
keyword:
 - CBOR Pointer
 - pathspec
 - JSON Pointer
 - XPointer
venue:
  group: "Concise Binary Object Representation Maintenance and Extensions"
  type: "Working Group"
  mail: "cbor@ietf.org"
  arch: "https://www.ietf.org/mail-archive/web/cbor/current/maillist.html"
  github: "rohanmahy/cbor-pointer"
  latest: "https://rohanmahy.github.io/cbor-pointer/draft-mahy-cbor-pointer.html"

author:
 -
    fullname: Rohan Mahy
    organization:
    email: rohan.ietf@gmail.com

normative:

informative:

...

--- abstract

CBOR Pointer is a syntax to identify a single CBOR value from a CBOR document with an arbitrarily complex nested structure.
It is analogous to JSON Pointer.


--- middle

# Introduction

CBOR Pointer is a syntax for identifying a single arbitrary subtree or element of a CBOR {{!RFC8949}} Document or a CBOR sequence.
It provides functionality analogous to JSON Pointer {{?RFC6901}} but supporting the full range of CBOR types.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Definition

A CBOR Pointer is an array consisting of pathspecs.
The entire array can be implicitly typed or explicitly typed.
The first pathspec operates on the root of the CBOR or CBOR sequence document as the parent element.
Pathspecs MUST be evaluated in array order, with the item selected by each pathspec becoming the parent element for the next pathspec.
A pointer with no pathspecs selects the entire CBOR document; this applies to both `[]` and `TBD1([])`.

Successful evaluation of a CBOR Pointer returns the selected CBOR item directly.
If evaluation does not identify an item, resolution fails.
Resolution failure is an error condition, not a CBOR value; selecting `null` or `undefined` is a successful evaluation.
As with JSON Pointer (Section 7 of {{RFC6901}}), applications define how resolution failure is handled.
This document does not prescribe a CBOR encoding or an API representation for evaluation outcomes.

If a pathspec cannot select an item according to the rules below, evaluation MUST fail and subsequent pathspecs MUST NOT be evaluated.
This includes a missing map key, an out-of-range array index, a tag number or explicit parent-type mismatch, or a parent element for which the pathspec has no defined operation.
An integer, floating-point number, text string, or simple value can be selected as the final result, but applying a further pathspec to it causes resolution failure.

## Pointer Validity

A CBOR Pointer MUST be either an untagged array of implicit pathspecs or an array wrapped in tag `TBD1` containing explicit pathspecs.
An implicit pathspec can be any CBOR data item, since it can identify a map key.
An explicit pathspec MUST be one of the forms defined in {{explicit-pathspecs}}, with the specified tag content type.
Any other explicit pathspec, including an unknown pathspec tag, is invalid.
These restrictions apply to the pathspec itself, not to tags or other items inside a map key.

Pointer validity is independent of the document being evaluated and of whether evaluation reaches a particular pathspec.
Invalid pointer syntax is an error condition distinct from resolution failure, as in Section 7 of {{RFC6901}}.
Applications define how these error conditions are reported.
An evaluator MAY stop at the first detected error; it is not required to validate the entire pointer before evaluation or to discover every error.

For example, `TBD1([TBD3("missing"), TBD2("first")])` is invalid because `"first"` is not an integer index, whether or not the first lookup succeeds.
In contrast, `TBD1([TBD2(100)])` is syntactically valid but fails to resolve when applied to an array with fewer than 101 elements.
The implicit pointer `["first"]` is also syntactically valid: it can select a map entry, but fails to resolve when applied to an array.

## Implicit Pathspecs

The semantics of an implicit pathspec depend on the type of the parent element.

- If the parent element is an array, it returns the appropriate element:
  - if the pathspec is an unsigned integer, it matches the element at that zero-based position from the start of the array;
  - if the pathspec is a CBOR negative integer `i` and the array has `n` elements, it matches the element at zero-based position `n + i`, provided `0 <= n + i < n`; thus, `-1` matches the last element, `-2` the second-to-last element, and so on.
- If the parent element is a map, the pathspec matches if it matches one of the map keys of the map. It returns the value of the map key.
- If the parent element is a tag, the pathspec matches if it matches the tag number. It returns the value inside the tag.
- If the parent element is a byte string, one layer of embedded CBOR is decoded as described in {{embedded-cbor}}. The same pathspec is then evaluated against the decoded item if it is an array, map, or tag; otherwise, resolution fails.
- If the root element is a CBOR sequence, the pathspec is evaluated as if the entire sequence were wrapped in an array.

## Examples with Implicit Pathspecs

Given the following source document, the table below gives the corresponding result.
In the table, *failure* indicates resolution failure rather than a CBOR value.

~~~ cbor-diag
777([
  [
    [1, "two", 3],
    [4, "five", 6]
  ],
  {
    1: "abc",
    -18: h'1234',
    "x": null,
    35: 1(1760686166),
    "y": [ "l", "m"]
  },
  <<{
    2: 45,
    "pdq": false
  }>>,
  27,
  h'abcdef'
])
~~~


| CBOR Pointer | Result |
|-----------+--------|
| `[77]` | *failure* |
| `[777, 3]` | `27` |
| `[777, 3, 0]` | *failure* |
| `[777, 9]` | *failure* |
| `[777, 9, 0]` | *failure* |
| `[777, null]` | *failure* |
| `[777, 0]` | `[[1,"two",3],[4,"five",6]]` |
| `[777, 0, 1]` | `[4, "five",6]` |
| `[777, 0, 1, 1]` | `"five"` |
| `[777, 1, 1]` | `"abc"` |
| `[777, 1, -18]` | `h'1234'` |
| `[777, 1, -18, 1]` | *failure* |
| `[777, 1, "x"]` | `null` |
| `[777, 1, "x", 0]` | *failure* |
| `[777, 1, 35]` | `1(1760686166)` |
| `[777, 1, 35, 1]` | `1760686166` |
| `[777, 1, "y"]` | `["l","m"]` |
| `[777, 1, "y", 1]` | `"m"` |
| `[777, 1, "z"]` | *failure* |
| `[777, 2]` | `h'a202182d63706471f4'` |
| `[777, 2, 2]` | `45` |
| `[777, 2, "pdq"]` | `false` |
| `[777, 2, 0]` | *failure* |


## Explicit Pathspecs {#explicit-pathspecs}

Explicit CBOR Pointers use tags to match a specific type of element for each pathspec.
If the type of the parent element matches the expected type, the matching rules and return values are the same as for implicit pathspecs, except that in explicit pathspecs, CBOR data items encoded in byte strings are unwrapped in a separate pathspec.
Explicit CBOR Pointers are always wrapped in the tag `<TBD1>`.
Each element in an explicit CBOR Pointer is either the simple value `<TBD0>` for byte string encoded elements, or a pathspec tagged with one of the following tags:


| Data Type | Tag  | Tag Content |
|-----------+------+-------------|
| array     | TBD2 | Unsigned or negative integer (major type 0 or 1) |
| map       | TBD3 | Any CBOR data item identifying the map key |
| tag       | TBD4 | Unsigned integer (major type 0) identifying the tag number |
| sequence  | TBD5 | Unsigned or negative integer (major type 0 or 1) |


Converting one of our implicit pathspec examples (`[777, 1, "y", 1]`) into explicit pathspecs, gives us:

~~~ cbor-diag
TBD1([          # Explicit pathspecs
    TBD4(777),  # Tag 777
    TBD2(1),    # 2nd Array element
    TBD3("y"),  # Map key "y"
    TBD2(1)     # 2nd Array element
])
~~~

Both the implicit and explicit version return the value `"m"`.
However, an explicit pathspec tag referring to a different type causes resolution failure.
Consequently explicit pathspecs are useful where different types could be in the same location and the distinction is semantically meaningful.

Explicit pathspecs involving embedded byte strings require an additional pathspec element. For example, the equivalent of the implicit pointer `[777, 2, 2]` (which returns `45`) is the following:

~~~ cbor-diag
TBD1([            # Explicit Pathspecs
    TBD4(777),    # Tag 777
    TBD2(2),      # 3rd Array element
    simple(TBD0), # decode byte string
    TBD3(2)       # Map key 2
])
~~~

This property of explicit pathspecs makes it possible to return the entire decoded value of an encoded byte string.
For example, the following explicit pointer applied to our original example:

~~~ cbor-diag
TBD1([            # Explicit Pathspecs
    TBD4(777),    # Tag 777
    TBD2(2),      # 3rd Array element
    simple(TBD0)  # decode byte string
])
~~~

returns `{2:45, "pdq":false}` as its result.

## Embedded CBOR {#embedded-cbor}

When a pathspec decodes a byte string as embedded CBOR, the byte string MUST contain exactly one complete, valid CBOR data item, and decoding MUST consume all of its contents.
Empty contents, invalid or incomplete CBOR, or any bytes following the encoded item cause resolution failure.
This operation does not decode a CBOR sequence containing multiple items.

The explicit pathspec `simple(TBD0)` requires a byte string parent and selects the decoded item, regardless of its type.
Applying it to any other parent type causes resolution failure.
Each decoding operation unwraps exactly one byte string layer.
If the decoded item is itself a byte string, another `simple(TBD0)` pathspec is needed to decode its contents.

An implicit pathspec decodes one layer and then applies that same pathspec to the decoded array, map, or tag.
It does not recursively decode a second byte string layer or select a decoded scalar item.
A pointer that ends at a byte string selects that byte string unchanged; its contents are not decoded or validated as embedded CBOR.

The following examples use each source item as the root document.
As above, *failure* indicates resolution failure.

| Source Item | CBOR Pointer | Result |
|-------------+--------------+--------|
| `h'181b'` | `[]` | `h'181b'` |
| `h'181b'` | `TBD1([simple(TBD0)])` | `27` |
| `h'181b'` | `[0]` | *failure* |
| `h'81181b'` | `[0]` | `27` |
| `h'42181b'` | `TBD1([simple(TBD0)])` | `h'181b'` |
| `h'42181b'` | `TBD1([simple(TBD0), simple(TBD0)])` | `27` |
| `h'42181b'` | `[0]` | *failure* |
| `h''` | `TBD1([simple(TBD0)])` | *failure* |
| `h'18'` | `TBD1([simple(TBD0)])` | *failure* |
| `h'0001'` | `TBD1([simple(TBD0)])` | *failure* |
| `27` | `TBD1([simple(TBD0)])` | *failure* |


# Security Considerations

TODO Security


# IANA Considerations

TO DO register 5 tags (TBD1 through TBD5) and 1 simple value (TBD0).


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
