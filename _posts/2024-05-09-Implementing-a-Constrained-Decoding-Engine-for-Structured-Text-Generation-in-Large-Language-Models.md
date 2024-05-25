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

## Implement core API

### `engine.initialize()`

Implementing this method requires us to initialize the grammar, tokenizer's vocabulary and configuration first. While initializing the vocabulary and configuration is straightforward, initializing the grammar involves constructing certain data structures from the EBNF text to facilitate a more efficient implementation later on. Specifically, we want to create an [abstract syntax tree(AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree) of the EBNF, validate the AST, and then simplify the AST into what I term Loup's normal form (LNF)[^2].

#### Construct EBNF AST

Fortunately, in Rust we have [ebnf](https://github.com/ChAoSUnItY/ebnf) library that handles most of work for us. We only need to make minor modifications[^1] to:

1. Support `any!` and `except!` special nonterminals
2. Support comments
3. Check regular expressions' syntactic correctness
4. Intern strings for regular expressions, nonterminals and terminals.

#### Validate the AST

It is possible for an EBNF snippet to be syntactically correct yet semantically incorrect. For example: `A::=B;`, where `B` is undefined. Hence, we will validate the AST against:

1. Undefined nonterminals
2. Invalid excepted nonterminals
    - All excepted nonterminals must directly contain nonempty terminals combined with concatneations and alternations[^6].

#### Obtain LNF

This step is trickier. To make efficient implementation easier, we would like to do the following:

- **Remove useless rules**

    Useless rules are rules that never contribute to text recognition. For example:

    ```ebnf
    S ::= 'ab'S | 'ab'A | 'ab'B;
    S ::= S;
    A ::= 'c'd;
    B ::= 'a'B;
    C ::= 'dc';
    ```

    Assuming `S` is the starting nonterminal, the rule `C ::= 'dc';` is unused and will be removed as it contributes nothing to the derivations starting from `S`. Similarly, the rule `S ::= S;` is considered useless because it recursively refers to itself indefinitely without leading to a terminal state. On the other hand, the rule `B ::= 'a'B;` will be retained. Although this production never terminates, it allows for the expression of constraints on infinite sequences, thus enhancing the grammar’s expressiveness.[^3].

- **Flatten nested rules with nonterminals**

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

- **Flatten operators with precedence**

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

- **Group rules with same left-hand side together**

    Well, most people already group rules with same left-hand side together. But maybe some translation layers(that convert Python class to EBNF schema) become lazy in certain cases. Anyway, we will turn `A::='a'; A::='b;` into `A::='a'|'b';` to ensure consistency.

- **Merge consecutive terminals**

    Obviously `A::='b''c''d';` can be simplified to `A::='bcd';`.

- **Deduplicate alternations**

    Obviously `A::='a'|'a';` can be simplified to `A::='a';`. Ensuring alternations of the same nonterminal are unique are helpful in later optimizations.

- **Remove unit rules**
    Rules like `A::=B; B::=C;` can be simplified to `A::=C;`. EBNF-unique recursion semantics should be perserved in this process: expressions like `A ::= B+; B ::= C?;` become `A ::= B*;`.

- **Expand repetitions and options**

    Note that:
  - `A::=B?;` is equivalent to `A::=""|B;`.
  - `A::=B*;` is equivalent to `A::=""|B|AB;`[^5].
  - `A::=B+;` is equivalent to `A::=B|AB;`.

    This simplification, along with flattening nested rules with nonterminals, remove EBNF-unique recursion semantics, allowing cleaner implementation.

- **Merge nonterminals with same productions**

    ```ebnf
    A ::= B|C;
    B ::= 'a';
    C ::= 'a';
    ```

    can be simplified to:

    ```ebnf
    A ::= B|B;
    B ::= 'a';
    ```

- **Eliminate nullable rules**

    Consider this grammar:

    ```ebnf
    A::=B? B? B? B? B?;
    B ::= 'b';
    ```

    And this string: `'bbbbb'`.
    If we do not remove nullable rules(nonterminals that accept empty string), then at the first `'b'`, we need to check:
  - whether `B? B? B? B? B?` accepts the character, since it is the direct derivation of `A`.
  - whether `B? B? B? B?` accepts the character, since `B?` is nullable, allowing us to skip it.
  - whether `B? B? B?` accepts the character, since `B? B?` is nullable, allowing us to skip it.
  - whether `B? B?` accepts the character, since `B? B? B?` is nullable, allowing us to skip it.
  - whether `B?` accepts the character, since `B? B? B? B?` is nullable, allowing us to skip it.

    When advancing to the next `'b'`, we must update our position in all five variations and repeat the above checks for each variation. A naive implementation may lead to exponential time complexity. While a more advanced approach might deduplicate the rules to reduce runtime costs, the most efficient strategy is to rewrite the grammar to avoid nullable rules entirely, so we will not have any runtime costs.

    **The Grammar Rewriting Procedure**

    1. Identify All Nullable Symbols: Nullable symbols are those that can:

        - Directly derive to an empty string (`A ::= '';`).
        - Be derived from concatenations of other nullable symbols (`A ::= B? B?;`).
    2. Modify Productions Involving Nullable Symbols: For each nullable symbol within a production, create two additional productions: one with the symbol and one without it. If removing the symbol results in an empty production, do not add it[^8].
        - For example, `A ::= B?;` is expanded to `A ::= B;`.
        - This procedure is similar to generating all possible combinations of nullable symbols in the production.

The code implementing all the procedures described above is available in [my forked ebnf repository](https://github.com/Dan-wanna-M/ebnf)[^9].

### `engine.modify_possible_logits()`

Unlike the AST transformations, which only require basic programming skills and straightforward tree traversal and construction algorithms[^7], implementing this function needs advanced algorithms and more aggressive optimizations.

#### Choosing an algorithm

Let's revisit our problem. After our engine is initialized with LNF, we need to

1. Determine if the current engine state can accept the input token.
2. Update the state if possible.
3. Identify all possible tokens that can be accepted by the updated state.
4. Update the logits accordingly.

The first three steps are similar to the operations of a [parser](https://en.wikipedia.org/wiki/Parsing) in streaming mode. Since our grammar is a proper superset of context free grammars, we should examine algorithms capable of parsing **[any context free language](https://en.wikipedia.org/wiki/Context-free_language).** This excludes [LL(k) parsers](https://en.wikipedia.org/wiki/LL_parser) and [LR(k) parsers](https://en.wikipedia.org/wiki/LR_parser)(such as [yacc](https://en.wikipedia.org/wiki/Yacc#:~:text=Yacc%20(Yet%20Another%20Compiler%2DCompiler,Johnson.))) which only handle a subset of context-free languages. The remaining parsers are [Earley](https://en.wikipedia.org/wiki/Earley_parser), [GLR](https://en.wikipedia.org/wiki/GLR_parser), [GLL](https://softwareengineering.stackexchange.com/questions/425562/how-does-the-gll-parsing-algorithm-work) and [parsing with derivatives](https://pdarragh.github.io/blog/2017/05/22/parsing-with-derivatives/). I prefer a modified Earley algorithm because:

- The way Earley works allows us to work on a contiguous buffer, while GLR, GLL and parsing with derivatives all need some graph-like or tree-like structures that lead to more pointer chasing and cache misses.
- Properly optimized, an Earley parser can match the linear time performance of LL(k)/LR(k) parsers on LL(k)/LR(k) grammar and can also parse many languages in linear time that LL(k)/LR(k) parsers cannot.
- The Earley algorithm can be easily adapted to incorporate our `except!` special nonterminal semantics without compromising its original time complexity.

#### Modifications to the original Earley algorithm

If you are unfamiliar with how the original Earley algorithm operates, I highly recommend reading [Vaillant Loup's blog](https://loup-vaillant.fr/tutorials/earley-parsing/) which remains one of the best resources on Earley parsing even after ten years.

- **Operate on byte level rather than token level**

    Traditional parsers operate on token sequences generated by a [lexer](https://en.wikipedia.org/wiki/Lexical_analysis). However, in our context, using a separate lexer is impractical as each LLM call generates only one logits and one sampled token. Requiring all terminals in KBNF to be composed from LLM tokens would make creating KBNF schema extremely tedious, rendering it infeasible. Encoding terminals into LLM tokens is infeasible since terminals could be tokenized in unexpected ways. Therefore, we will encode all LLM token strings and KBNF terminals in UTF-8, enabling our engine to operate at the byte level.

    This also means we need a way to indicate how far we are in a given terminal, since a terminal may consist of multiple bytes. We can achieve this goal by adding one extra field `stateID` to all Earley items.

- **Discard completed Earley items**

    The Earley parsing algorithm typically retains all completed items to construct a parsing forest. However, since our goal is only to recognize strings, we can discard completed Earley items (i.e., those for which completions have been executed) in an Earley set. This optimization allows us to compress the set of to-be-completed Earley items into a set of tuples `(<nonterminal>, <start_column>)`, e.g., compressing `(A -> a., 1)` into `(A, 1)`.

- **Deploy Leo's optimization**

    [Leo's optimization](https://www.sciencedirect.com/science/article/pii/030439759190180A), a modification on the original Earley algorithm, enables parsing all LR(k) grammars in linear time. Despite its effectiveness, it is often not explained in plain terms. I will restate Leo's optimization here. Consider this grammar with the string `'aaaaa'`:

    ```ebnf
    A::='a'A|'a';
    ```

    The corresponding Earley sets:

    ```txt
    ======0======
    (A ->.a A, 0)
    (A ->.a, 0)
    ======1======
    (A -> a.A, 0)
    (A -> a., 0)
    (A ->.a A, 1)
    (A ->.a, 1)
    ======2======
    (A -> a.A, 1)
    (A -> a., 1)
    (A ->.a A, 2)
    (A ->.a, 2)
    (A -> a A., 0)
    ======3======
    (A -> a.A, 2)
    (A -> a., 2)
    (A ->.a A, 3)
    (A ->.a, 3)
    (A -> a A., 1)
    (A -> a A., 0)
    ======4======
    (A -> a.A, 3)
    (A -> a., 3)
    (A ->.a A, 4)
    (A ->.a, 4)
    (A -> a A., 2)
    (A -> a A., 1)
    (A -> a A., 0)
    ======5======
    (A -> a.A, 4)
    (A -> a., 4)
    (A ->.a A, 5)
    (A ->.a, 5)
    (A -> a A., 3)
    (A -> a A., 2)
    (A -> a A., 1)
    (A -> a A., 0)
    ```

    Note that the repeated processing of `(A -> a A., x)` leads to quadratic time complexity. Also, note that the completed Earley items' chain

    `(A -> a., 4)=>(A -> a A., 3)=>(A -> a A., 2)=>(A -> a A., 1)=>(A -> a A., 0)`

    where the first completed item triggers the the second completed item, which triggers the third, and so on, is the **only** way `(A -> a.A, 0)` in set `0` becomes `(A -> aA., 0)` in set `4`. Similarly, this chain is **also** the **only** way `(A -> a.A, 0)` in set `0` becomes `(A -> aA., 0)` in set `3`, in set `2`, and so on. This motivates us to cache `(A -> aA., 0)` and `A` in some ways, since we know the completion of `A` will always lead to `(A -> aA., 0)` eventually. This can be more formally stated with the pseudocode:

  - **Leo optimization(Lazily evaluated)**

    ```python
    def find_leo_item(earley_set, nonterminal):
        filtered = list(filter(earley_set,
                    lambda item: item.current_symbol == nonterminal))
        if len(filtered) == 1: # there exists exact one parent
            temp = filtered[0].advance_dot() # create a new item
            if temp.dot_index == len(item.rule):
                # reaches the end of the rule after advancing
                return temp # This is the leo item
        return None

    def find_topmost_leo_item(earley_set, nonterminal,last_leo_item):
        if nonterminal in earley_set.leo_items:
            return earley_set.leo_items[nonterminal]
            # one nonterminal will only correspond to one chain in a set
            # otherwise it is no longer a chain.
        leo_item = find_leo_item(earley_set, nonterminal, last_leo_item)
        if leo_item is not None:
            leo_item = find_topmost_leo_item(
                leo_item.initial_earley_set,
                 leo_item.nonterminal, leo_item)
            earley_set.leo_items[leo_item.nonterminal] = leo_item
        return last_leo_item

    def leo_complete(earley_item, earley_set):
        leo_item = find_topmost_leo_item(earley_item.initial_earley_set,
         earley_item.nonterminal, None)
        if leo_item is not None:
            earley_set.leo_items[earley_item.nonterminal] = leo_item
            return True
        return False

    def complete(earley_item, earley_set):
        if not leo_complete(earley_item, earley_set):
            earley_complete(earley_item, earley_set)
    ```

  Finding the topmost Leo item is not too complex. It is essentially the chain of Earley items where all the intermediate items are thrown away. The definition of Leo item constraint, **however**, is **subtle**. If we use this constraint instead:

  ```python
  def find_leo_item(earley_set, nonterminal):
        filtered = list(filter(earley_set,
                    lambda item: item.advance_dot().dot_index == len(item.rule)))
        # find all rules one step away from completition first
        if len(filtered) == 1: # there exists exact one
            if filtered[0].current_symbol == nonterminal:
                # check the postdot symbol
                return temp # This will NOT work!
        return None
  ```

  **Then the code will NOT work!** [This post](https://cs.stackexchange.com/questions/101770/leos-deterministic-reduction-for-earley-parsing) shows a counterexample.

- **Use contiguous buffers for Earley sets and LNF**

    A naive implementation of Earley sets might utilize `Vec<Vec<T>>`, a 2D nested vector. However, this setup isn't optimal due to potential cache misses; the inner vectors’ buffers may not be allocated in one contiguous memory block.

    A more efficient approach is to use a "jagged vector," similar to a [jagged array](https://en.wikipedia.org/wiki/Jagged_array) but with the capability to append data to the last row and create new rows as needed. A jagged vector utilizes a contiguous buffer as its element storage and a separate dense array of indices to locate rows within the buffer.
    By employing multiple hierarchical index arrays, this structure also enables efficient access to flattened LNF grammar, where `struct[nonterminalID][production_id][rule_ID]` corresponds to a single dotted symbol in an Earley item.

    I implemented the jagged array in my [jaggedarray](https://crates.io/crates/jaggedarray) crate.

- **Support regular expressions**
    To support regular expressions, we will:
    1. **Define the representation of a regular expression**
        Various representations exist for regular expressions, including naive backtracking, densely represented deterministic finite automata (DFA), sparsely represented DFA, and Thompson nondeterministic finite automata (NFA). I prefer using the lazily built dense DFA as the default option because it maintains linear time complexity and is relatively fast. Other options like fully compiled DFA are configurable.

    2. **Extend the earley algorithm to handle regular expressions**
        We will integrate regular expressions into the Earley algorithm with a `stateID` in an Earley item to represent the current state within a DFA. When we process the regular expression during scanning stage of the Earley algorithm:
        - If the byte does not match, no action is taken.
        - If the byte matches and completes the DFA, the item is copied to the next Earley set, and the dot is advanced by one step.
        - If the byte matches but the DFA requires more bytes for a full match, the item is copied to the next Earley set with an updated `stateID`.
        - If the byte matches and the DFA can both end and accept more bytes, the item is duplicated in the next Earley set: one copy advances the dot, and the other updates the `stateID`.

        Since empty rules are already handled during the generation of LNF, there is no need to check if the regular expression can accept an empty string here.
- **Support `except!`**
    To support `except!`, we will:
    1. **Define the representation of a `except!`**
        The simplest and probably the most efficient way to represent the `except!` is to use a regular expression[^11] in unanchored search mode.
    2. **Extend the earley algorithm to handle `except!`**
        We will integrate regular expressions into the Earley algorithm with a `stateID` in an Earley item to represent the current state within a DFA. With bit manipulation, we can store the current maximum repetition in `stateID` as well.
        When we process the regular expression in `except!` during scanning stage of the Earley algorithm:
        - If the byte does not match, the item is duplicated in the next Earley set: one copy advances the dot, and the other updates the `stateID`.
        - If the byte matches, no action is taken.
- **Use regular expression for large sets of terminals**
    Representing large sets of terminals as individual LNF nodes leads to significant memory and runtime overhead. A more efficient approach is to compile these terminals into a DFA using regular expressions, which compresses common prefixes, infixes, and suffixes.
- **Compress Earley items and LNF nodes**
    An Earley item contains `(nonterminalID, dot_position, production_id, start_position, stateID)`. Some fields usually require a narrower range than others; for example, it's unlikely for there to be more than 256 items in a single production or more than 256 productions for a single nonterminal. This allows us to use smaller integers, such as `u16u8u8u16u16` or `u8u8u8u8u32`. With generics, the most optimal combination can be selected based on user configuration and specific grammars. This method also applies to Leo items, to-be-completed Earley items, and LNF nodes.
- **Filter tokens by possible first bytes**
    Once an Earley set enters the scanning stage (where there are no items left to be completed or predicted), we know precisely which bytes can potentially be accepted. Efficient prefix-search data structures (like `HashMap<u8, Box<[u8]>>`) can be utilized to filter out a potentially large number of tokens without running the Earley recognizer.

- **Cache Possible Tokens**

    **Theoretical Discussion**

    Unlike regular expressions, the computational model equivalent to context-free grammar is [pushdown automata(PDA)](https://en.wikipedia.org/wiki/Pushdown_automaton), which is not a finite state automaton. This implies that it is theoretically impossible to fully cache all states of a given context-free grammar. Moreover, even for LL(k) or LR(k) grammars, subsets of context-free grammar equivalent to [deterministic pushdown automata](https://en.wikipedia.org/wiki/Deterministic_pushdown_automaton),  finite state automata may still not simulate them due to the infinite capacity of the stack in PDA.

    So why bother caching possible tokens? There are two main reasons:

    1. **Simplicity of many grammars**: A significant number of grammars are not exceedingly complex. For example, regular grammars, left-recursive grammars, and right-recursive grammars can often be fully cached[^12], which is one reason tools like Outlines and Guidance initially supported only regular expressions.

    2. **Finite output length**: When the output length is finite, the number of states involved is also finite. Given that the same grammar is often reused to generate outputs of similar lengths, it is likely that the same states will be reached repeatedly, making caching effective.

    Indeed, there are scenarios where neither of these conditions is met. Hence, enabling cache support in our system will be optional.

    **The Algorithm**

    The actual caching algorithm is complex. Naively caching all Earley sets for a given input sequence is very inefficient. We need a method to compact Earley sets. For Earley sets `S(0), S(1), ..., S(m)`, we have the following observations:

    1. All Earley sets `S(i)` for which `max(start position in D) < i < m` do not influence accepted bytes, where `D` consists of items in `S(m)` not created at `m`.
    2. The start positions of items created at `S(m)` do not influence accepted bytes.
    3. For any item `(nonterminal, start_position, ...)` in `S(m)`, if `S(start_position)` contains a Leo item corresponding to `nonterminal`, then the `start_position` can be changed to the Leo item's start position without affecting accepted bytes.

    These observations lead to our compacting algorithm:

    1. Iterate over `S(m)` and apply observation 3 wherever possible.
    2. Apply observation 1 to shift `S(m)` to position `n`.
    3. Adjust the start positions of items referred to in observation 2 to `n`.

    This compacted Earley set can then serve as an index in the cache. What's more, even if caching is disabled, this compacting algorithm might still be useful as it reduces memory usage on long inputs and possibly cache misses.

    Additional heuristic caching strategies might involve using nonterminal IDs as cache indices and computing the union of possible tokens over all dotted nonterminals. I will leave the decision on whether to use these strategies to the user.

    **Cache mode**

    It is theoretically possible to cache all states of Earley recognizer up to `m` input bytes. However, its time complexity is equivalent to enumerate all possible ASTs with inputs up to `m` bytes. I suspect this will only work for simple grammars or short inputs. Lazy mode sounds like a more reasonable default option.

## Do we meet our goals?

Let's revisit our goals and check whether we have met them.

[^1]: Well, sort of. You can definitely argue that we also need user-friendly errors for our EBNF variant to enhance usability, and I think you are right. The problem is that the library does not have these features, and implementing these features are time-consuming, and hence I will omit them. At least for *now*.
[^2]: The rationale behind this naming is because I only see this form in [Loup's blog](https://loup-vaillant.fr/tutorials/earley-parsing/recogniser). This is definitely not an academic term but to the best of my knowledge, I do not know if there exists a formal name.
[^3]: Is this particularly useful? Maybe not. Does this affect implementation's efficiency? Definitely not. So if EBNF by definition allows it, why don't we allow it?
[^4]: That's why I code all the steps before with a while loop and a stack on the heap.
[^5]: This is the left recursion version. You may want the right recursion version. I prefer left recursion because I will use Earley Algorithm later.
[^6]: It is technically possible to support more complex nonterminals that only contain terminals *after substituting all nonterminals and grouping*. However, this feature is harder to implement so I will leave it for *now*.
[^7]: Well, not that *simple* if you, like me, do not want to use recursion and hate reference counting and unnecessary cloning.
[^8]: Note that accepting an empty string at the beginning is useless in our context.
[^9]: Maybe in the future I will publish it as a Rust crate.
[^11]: After typing this down I realized I could support full regular expression as negative lookaheads easily. Whether that is a good design choice and how much interference it has on other optimizations are unclear though.
[^12]: This is just a feeling. Maybe I am wrong.
