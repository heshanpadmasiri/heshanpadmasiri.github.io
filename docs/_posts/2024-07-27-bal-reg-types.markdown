---
layout: post
title:  "Regular expressions as set-theoretic types in Ballerina"
date:   2024-07-27 11:15:43 +0530
categories: Ballerina
---
_This article was original published on [Medium](https://medium.com/@hpheshan/regular-expressions-as-set-theoretic-types-a23bda54d01f)_

Given their widespread use, I shall assume readers are already familiar with regular expressions (basic knowledge of regular expression notations such as union and Kleene star is sufficient for our purposes) and start my explanation from set-theoretic types. Set-theoretic types treat types in programming languages as sets of values. For example, type `int` is the set of integers. Then subtyping operation (determining if a given type belongs to another type) is equivalent to testing the subset relation between two sets. Furthermore, once types have been represented as sets one can naturally represent new types by applying set-theoretic operations such as union and intersection on types.

While this makes understanding typing relations between various types quite natural to the programmer, implementing a typing system that can handle set-theoretic types is not a trivial task (in contrast to having a set of rules in the language specifications that determine typing relations). As a result, only a handful of languages such as [CDuce](https://www.cduce.org/) and [Ballerina](https://ballerina.io/) implement set-theoretic types.

In contrast, almost all the major programming languages implement regular expressions. However, in most such implementations regular expression related operations are handled by dedicated regular expression resolvers that use specialized algorithms for those operations. However if one can represent regular expressions as types in a language that supports set-theoretic types, then we can just use the type system of the language to perform regular expression operations. This is because the type corresponding to a given regular expression represents the set of all strings that are valid under the language of that regular expression. Then if we want to check if a given string matches that regular expression all we have to do is check if that string is an element in the set represented by the regular expression. This is same as checking the subset relation between the set of the regular expression and the set with just that string, which corresponds to the subtyping operation. Given most languages that support set-theoretic types treat each string as a set of just that string, this is quite straightforward.

Using this property of set-theoretic types some languages, especially once more focused on string manipulation such as CDuce directly allow [regular expression operations on types](https://www.cduce.org/manual_types_patterns.html#syntax). Given individual characters are also types one can straightforwardly represent any regular expression as a type using that.

However more general purpose languages such as Ballerina don’t allow regular expression operations on types. However, in this post, we shall see even in such languages one can still represent regular expressions as types as long as the typing system support recursion.

For demonstrations, I shall be using Ballerina. However, the examples used here should be intuitive enough for any reader who is not familiar with the Ballerina language. Readers interested in a deeper understanding of the ballerina typing system may find [this](https://www.youtube.com/watch?v=_4x5v4rGUOw) presentation by James Clark useful.

# Representing regular expressions as types

Similar to [Thompson’s constructions](https://en.wikipedia.org/wiki/Thompson%27s_construction) we shall represent regular expressions by recursively splitting regular expressions into their subexpressions. Since it is possible for a given regular expression to have more than one way to split it we shall formalize our splitting process as follows

1. Each regular expression consists of a concatenation of two subexpressions, where these subexpressions could be either empty, single character, concatenation, union, or Kleen star. We shall call these two subexpressions `head` and `tail`.
2. Since it is possible to have more than one way to split a regular expression this way we shall restrict `head` to be just a single character. Thus we can represent regular expression as follows

```typescript
type RegExp () | [string:Char, SubExp];
type SubExp Concat | Union | Star | ();
```

_We are using `()` (which represent `nil` in Ballerina) to represent the end of the regular expression._

# Concatenation
We shall start by assuming that `Union` and `Star` operations don't exist. Then concatenation is just a sequence of characters. Then we can represent concatenation as follows

```typescript
type Concat () | [string:Char, T];
```

As an example, we can represent “abc” as follows.

```typescript
type T ["a", T_bc];
type T_bc ["b", T_c];
type T_c ["c", ()];
```

# Union
Unlike other operations, union operation has the corresponding union set operation, which we will use. Let’s start by considering the case of a simple union of just two subexpressions. Then we can represent union type as follows

```typescript
type Union SubExp1|SubExp2;
```

As an example, we can represent “a|b” as follows (using the definition of concatenation)

```typescript
type T T_a | T_b;
type T_a ["a", ()];
type T_b ["b", ()];
```

Now to combine concatenation with the union we shall consider 2 possibilities.

1. Character sequence followed by union
2. Union followed by a character sequence

In the first case, all we have to do at the end of the character sequence is let `T` in the `Concat` type be the union type. As an example consider "ab(c|d)".

```typescript
type T ["a", TbU];
type TbU ["b", TU];
type TU TC | TD;
type TC ["c", ()];
type TD ["d", ()];
```

For the second case, one can think of a union operation to spread over the character sequences. As an example, if we think about “(a|b)cd”, this is the same as “acd|bcd” which we know how to represent.

Now we can allow concatenation to have unions by allowing the first union we see to spread over the rest. As an example consider “a(b|c)d(e|f)”

```typescript
type T  ["a", T1];
type T1 T2 | T3;
type T2 ["b", T4];
type T3 ["c", T4];
type T4 ["d", T5];
type T5 T6 | T7;
type T6 ["e", ()];
type T7 ["f", ()];
```

_For compactness, I have combined identical types into a single type but this is not necessary_

# Kleene star

Similar to union let’s start by considering the case where we only have a Kleene star. In this case, we can define our type as a union of `()` (to represent an empty string) and a type that terminates with recursion to the union itself as shown below

```typescript
type Star () | T1;
type T1 [string:Char, T2];
type T2 T1 | Star;
```

Here we are using `T1` to represent the sequence covered by the Kleene star (using the concatenation) and we terminate with the `Star` type. Note that this means `Star` is defined using `Star` itself and this is why we need to support recursive types in the type system. As an example consider "(ab)*"

```typescript
type T  () | T1;
type T1 ["a", T2];
type T2 ["b", T];
```

Then we can consider the following 2 cases

1. Subexpression sequence followed by star.
2. Star followed by a subexpression sequence.

Since we can deal with the first case the same as we did in the corresponding case for unions I shall focus only on the second case. This is the general case for what we have discussed so far. Let `T_rest` be the type representing the subexpression sequence after the star then we can define Star in a more general form as

```typescript
type Star T_rest | T1;
type T1 [string:Char, T2];
type T2 T1 | Star;
```

Finally as an example with everything we have seen so far let us consider “a(b|c)d(ef)*g”

```typescript
type T  ["a", T1];
type T1 T2 | T3;
type T2 ["b", T4];
type T3 ["c", T4];
type T4 ["d", T5];
type T5 T6 | T7;
type T6 ["g", ()];
type T7 ["e", T8];
type T8 ["f", T5];
```

# Using the nballerina regex program

For readers who like to experience the result of what we have discussed so far, we have created a simple command line program. To use it you can clone the [nballerina repository](https://github.com/ballerina-platform/nballerina) and build the regex program by running `make` in `./extra/regex`. This will produce a jar file in `./build/extra/bin` called `nballerina.regex`. Then you can run it as `java -jar nballerina.regex.jar regex1 regex2`. As an example running `java -jar nballerina.regex.jar "a" "a*"` will show the type relation between "a" and "a*". One can use the optional argument `balTypes` to get the ballerina type definitions for the regular expressions (example `java -jar nballerina.regex.jar --balTypes "a" "a*"`).

This program can show when a given regular expression (or a string) belongs to another regular expression (represented as `A < B` when A belongs to B), one regular expression is equal to another (represented as `A = B`), or when neither regular expressions are subsets of each other (represented as `A <> B`).
