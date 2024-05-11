---
title: "Implementing a Constrained Decoding Engine for Structured Text Generation in Large Language Models"
date: 2024-05-05
---

# Implementing a Constrained Decoding Engine for Structured Text Generation in Large Language Models

{% assign post = site.posts | where: "title", "Designing a Constrained Decoding Engine for Structured Text Generation in Large Language Models" | first %}

I recommend checking out my previous post, [Designing a Constrained Decoding Engine for Structured Text Generation in Large Language Models]({{ post.url | relative_url }}), before reading this one if you haven't already.

## Choosing a programming language

By inspecting our design, we observe several key aspects on the implementation:

1. **Complex Implementation**: The design is complex and likely will lead to complex implementation.
2. **Performance Constraints**: The implementation is primarily CPU-bound or memory-bound, rather than IO-bound.
3. **Efficiency**: The implementation must be efficient.
4. **Integration Flexibility**: The program must be easily integratable into other applications or languages.
5. **Stability of Demand**: Changes in implementation due to shifting demands are extremely unlikely.

Considering these factors, C++ and Rust are the optimal choices. They offer zero-cost abstractions, allow low-level operations, can export a C ABI, and are heavily optimized for performance. While I personally prefer Rust for its near zero-cost safety features and modern toolchain, modern C++ is equally capable under these criteria.

If the requirements for efficiency and ease of embedding were lifted, virtually any language with sufficient expressive power could be used.

## Implement API

### `engine.initialize()`

Implementing this method requires us to initialize the grammar, tokenizer's vocabulary and configuration first. While initializing the vocabulary and configuration is straightforward, initializing the grammar involves constructing certain data structures from the EBNF text to facilitate a more efficient implementation later on. Specifically, we want to create an [abstract syntax tree(AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree) of the EBNF and then simplify the AST into what I term Loup's normal form (LNF)[^2].

#### Construct EBNF AST

Fortunately, in Rust we have [ebnf](https://github.com/ChAoSUnItY/ebnf) library that handles most of work for us. We only need to make minor modifications[^1] to include `any!` and `except!` special nonterminals and support comments.

#### Obtain LNF

This step is trickier. To make efficient implementation easier, we would like to do the following, sequentially:

1. **Remove unused rules**

    Unused rules are a subset of [useless productions](https://www.geeksforgeeks.org/simplifying-context-free-grammars/) in the sense that we allow productions that never terminate because they are not strictly "useless" in our context. For example:

    ```ebnf
    S ::= 'ab'S | 'ab'A | 'ab'B;
    A ::= 'c'd;
    B ::= 'a'B;
    C ::= 'dc';
    ```

    In this grammar, assuming `S` is our starting nonterminal, the rule `C ::= 'dc'`; is unused and will be removed. However, `B ::= 'a'B;`, a production that never terminates, will be retained. Allowing productions that never terminate enables expressing constraints on infinite sequences, thus enhancing expressiveness[^3].

2. **Flatten nested rules with nonterminals**

    Nested rules refer to rules containing groups, options and/or repetitions. We can view groups, options and/or repetitions are "anonymous nonterminals." For example,

    ```ebnf
    A ::= ("a"|"b")"c"*;
    ```

    is equivalent to:

    ```ebnf
    A ::= GroupA1 C;
    GroupA1 ::= "a"|"b";
    C ::= "c"*;
    ```

    To unify representation, we will replace all "anonymous nonterminals" with explicitly named nonterminals.

3. **Flatten operators with precedence**

    The AST of `filter ::= 'a' , 'b' , 'c' | 'd';` looks like this:

    ```yaml
    - - lhs: filter
        rhs:
        Operator:
            - Terminal: a
            - Concatenation
            - Operator:
                - Terminal: b
                - Concatenation
                - Operator:
                    - Terminal: c
                    - Alternation
                    - Terminal: d
    ```

    This representation makes perfect sense for an EBNF AST but is inefficient for manipulation due to its hierarchy's misalignment with operator precedence. Additionally, deep tree traversal can lead to inefficiencies such as pointer chasing and cache misses, as well as implementation challenges like stack overflow due to excessive recursion[^4].

    We prefer a tree-like representation that aligns with operator precedence and is relatively shallow, like this:

    ```yaml
    - - lhs: filter
        rhs:
        Operator:
            Altercations:
            - Concatenations:
                - Terminal: d
            - Concatenations:
                - Terminal: a
                - Terminal: b
                - Terminal: c
    ```

4. **Group rules with same left-hand side together**

    Well, most people already group rules with same left-hand side together. But maybe some translation layers(that convert Python class to EBNF schema) become lazy in certain cases. Anyway, we will turn `A::='a';A::='b;` into `A::='a'|'b';`.

5. **Remove unit rules**

    We will also flatten unnecessary nested nonterminals, like from `A::=B+;B::=C?;` to `A::=B*;`.

6. **Merge consecutive terminals**

7. **Eliminate null rules**

    a


[^1]: Well, sort of. You can definitely argue that we also need syntax highlight and user-friendly errors for our EBNF variant to enhance usability, and I think you are right. The problem is that the library does not have these features, and implementing these features are time-consuming, and hence I will omit them. At least for *now*.
[^2]: The rationale behind this naming is because I only see this form in [Loup's blog](https://loup-vaillant.fr/tutorials/earley-parsing/recogniser). This is definitely not an academic term but to the best of my knowledge, I do not know if there exists a formal name.
[^3]: Is this particularly useful? Maybe not. Does this affect implementation's efficiency? Definitely not. So if EBNF by definition allows it, why don't we allow it?
[^4]: That's why I write all the steps before with a while loop and a stack on the heap.
