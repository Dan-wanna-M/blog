---
title: "Designing a Constrained Decoding Engine for Structured Text Generation in Large Language Models"
date: 2024-05-05
---

# Designing a Constrained Decoding Engine for Structured Text Generation in Large Language Models

## Motivation

Structured text generation means to create text in specific formats, such as JSON, markdown, HTML, or code. Typically, prompting a model to produce text in these formats works well *most of the time*. But what if you need it to work perfectly *every time*? That’s where constrained decoding comes into play. It prevents the generation of unwanted tokens by adjusting the model's output probabilities. Combined with prompt engineering, constrained decoding ensures the output strictly follows the desired format. Moreover, if constrained decoding limits the possible outputs to a single token, there's no need to call the model again, which can [significantly speed up the generation process](https://blog.dottxt.co/coalescence.html#orga014f18).

### Why do we need an engine? Can't we just handcode it?

While it might seem straightforward, implementing constrained decoding correctly is difficult. Consider a *simple* scenario where we need the model to output in the format `XXXXX*[YYYY]*`. Here, `XXXXX` should not include `*[` and `YYYY` should not include both `*[` and `]*`. It is tempting to quickly write:

```python
# some logic
in_y = False
if last_token == "*[":
    in_y = True
if in_y:
    logits[find_token_id("*[")] = float("-inf") # tweak the probability
if last_token == "]*":
    # logic to end generation
# some logic
```

Unfortunately, this code fails for several reasons:

- `*[` may not exist in the model's vocabulary at all.
- The model may decide to split `*[` into two tokens(`*` and `[`) and bypass our constraint.
- The model may decide to output `a*[` or `*[a` or whatever token containing `*[` and bypass our constraint.

The correct version:

```python
# some logic

for token in vocabulary:
    if '*[' in token and token.index('*]') != len(token)-1:
            logits[find_token_id(token)] = float("-inf")
in_y = False
if output_text.endswith("]*"):
    # logic to end generation
if "*[" in output_text:
    in_y = True
if in_y:
    for token in vocabulary:
        if '*[' in token:
            logits[find_token_id(token)] = float("-inf")
        if output_text.endswith("*") and '[' in token:
            logits[find_token_id(token)] = float("-inf")
# some logic
```

Even handcoding a simple format requires handling many nuances. Now, imagine handcoding constrained decoding for full JSON. This example should demonstrate why handcoding constrained decoding in general is *not* a good idea.

### Why not just use out-of-the-box solutions like Guidance, Outline, Jsonformer, etc.?

Out-of-the-box solutions such as Guidance, Outline, Jsonformer, and others, can be very useful and might perfectly meet your needs. However, they might not be the best fit if you:

- Prefer not to incorporate a Python runtime into your application.
- Need a faster engine.
- Require very fine-grained control over the process.
- Would like to write something much more interesting that day-to-day application code.

## Designing the engine

### Goals

- Users should be able to easily use the engine to specify commonly used formats or patterns.
- The engine should implement formats in the most efficient way. In other words, if a user does not use extra expressiveness, then they should not incur unnecessary overhead.
- Users should have the flexibility to select their preferred options when implementation choices involve compromises.

### Choosing a metasyntax notation

To enable users to specify a format, we need a method to describe that format. A long-established approach is to use a metasyntax notation, a format designed to describe other formats. To select a metasyntax notation, we need to ask ourselves: "[How much expressiveness do we need?](https://en.wikipedia.org/wiki/Chomsky_hierarchy)" Considering our goal to support all commonly used formats, we should examine the most complex yet practical formats—programming languages. [They turn out to be mostly context-free.](https://cs.stackexchange.com/questions/140078/are-modern-programming-languages-context-free) Then, a variant of [Extended Backus-Naur Form(EBNF)](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form) is a good choice:

- It is a wided used(and arguably easy to use) metasyntax notation to define format even before LLM era.
- It is independent from any specific programming languages.
- Popular libraries like [Outlines](https://github.com/outlines-dev/outlines?tab=readme-ov-file#using-context-free-grammars-to-guide-generation), [Guidances](https://github.com/guidance-ai/guidance?tab=readme-ov-file#context-free-grammars) and [Llama.cpp](https://github.com/ggerganov/llama.cpp/blob/master/grammars/README.md) all use some variants of EBNF.
- It is extensible and hence allows us to define some context-sensitive stuffs for ease of use.

Besides all the standard features of EBNF(except exceptions[^3]), we also want:

#### Standard regular expression[^1] support

```ebnf
digits ::= #'[0-9]'; (*A nonterminal that accepts one digit*)
(*Yes, this weird thing is a comment in ebnf*)
```

Adding regular expression improves usability. It also enables potential optimizations since regular expression can be compiled into [determistic finite automata(DFA)](https://en.wikipedia.org/wiki/Deterministic_finite_automaton).

#### Regex-like operators extension [^2]

```ebnf
digits ::= #'[0-9]';
digits ::= digits+; (*A nonterminal that accepts one or more digit*)
```

```ebnf
aaa ::= 'a'*; (*A nonterminal that accepts zero or more 'a'*)
```

```ebnf
aaa ::= 'a'?; (*A nonterminal that accepts zero or one 'a'*)
```

Many EBNF variants already use this extension, enhancing usability.

#### any! special nonterminal

```ebnf
start ::= any!;(*Accept any token*)
```

This is essentially wildcards, but for tokens rather than characters.

#### except!(\<strings\>,[n]) special nonterminal

```ebnf
start ::= except!('\n\n')'\n\n';(*Accept a text sequence
 where the only '\n\n' is at the end*)
(*'114514\n\n' will be accepted*)
(*'114514' will not be accepted*)
(*'114\n\n514\n\n' will not be accepted*)
```

```ebnf
start ::= except!('\n\n', 10)'\n\n';(*Accept a text sequence
 that at most consists of 10 tokens
  where the only '\n\n' is at the end*)
(*'114514\n\n' will be accepted*)
(*'114514' will not be accepted*)
(*'114\n\n514\n\n' will not be accepted*)
```

```ebnf
end ::= '\n\n'; (*The nonterminal in except!,
 after full expansion,
 must only contain alternation of strings.*)
start ::= except!(end)end;(*Accept a text sequence
 where the only '\n\n' is at the end*)
(*'114514\n\n' will be accepted*)
(*'114514' will not be accepted*)
(*'114\n\n514\n\n' will not be accepted*)
```

I know it looks ugly, but it's necessary. Without this extension, we still can't easily express the format `XXXXX*[YYYY]*` where `XXXXX` should not include `*[` and `YYYY` should not include both `*[` and `]*`.

**Can't we just augment regular expression with negative lookahead?**

Most regular expression engines either do not support lookahead at all or support the arbitrary version. Modifying these engines to extend or limit their capabilities would introduce significant complexities.

Furthermore, augmenting regular expressions in this way would require a way to embed nonterminals within the regular expressions, leading to added complexity. Such changes could also break compatibility with standard regular expression syntax.

**Can't we just use \#except!(\<strings\>) to represent one token and treat it like a normal nonterminal?**

It won't work. Consider this hypothetical syntax:

```ebnf
X ::= except!('*[')|except!('*[')X;
```

- Defining its semantics to exclude all tokens containing `'*['` in the middle does not prevent the model from outputting `'*'` and `[` tokens separately.
- If its semantics is defined as to reject any token that creates `'*['` in the generated text, then the semantics will be confusing. For instance, theoretically `'*'X` should allow `*[` at the beginning (where the first `'*'` comes from the terminal and the second from `X`, which only bans `*[` and not `[`); however, this proposed definition would reject `*[` at the beginning.
- Defining special handling for cases like `X` would essentially recreate the original `except!` syntax but with a more convoluted representation and implementation.

The fundamental issue is that to implement `except!` semantics, the nonterminal must be **stateful** to track the text it has accepted. However, nonterminal recursion in EBNF or context-free languages is **not** designed to be stateful.

**Can't we just use more natural syntax like except!(\<strings\>)\*?**

This extended semantics is not natural at all in the context of context-free languages. It is context-sensitive. As discussed above, we need an integrated method to handle its repetition, but `except!(<strings>)*` makes it appear as a simple combination of `except!(<strings>)` and `*`. In other words, a nonterminal repeated 0 or more times. As the discussion above indicates, this is **not** the correct interpretation. An integrated, though less elegant syntax, will clearly indicate that its semantics differ significantly from other parts of the grammar.

### Core API Design

Fortunately, considering the use cases, our theoretical API can be very simple:

1. `engine.initialize()`: Initializes the engine with grammar, vocabulary, configuration, etc.
2. `engine.modify_possible_logits()`: A high-level method that takes a new token and a mutable logits slice to:
    - Modify the input logits and return a flag indicating some next tokens can be accepted.
    - Leave the input logits unchanged and return a flag indicating the grammar is exhausted, and no further inputs are allowed. Exhaustion is defined as the scenario where previous tokens plus the new token form a string fully recognized according to the grammar.
    - Leave the input logits unchanged and return a flag indicating the input token is invalid according to the grammar.
3. `engine.advance()`: A low-level method that takes a new token, updates internal states, and returns flags to indicate:
    - New inputs can be accepted.
    - The grammar is exhausted.
    - The input token is invalid according to the grammar.
4. `engine.current_possible_tokens()`: Returns current possible tokens based on the engine's state.
5. `update_logits_from_tokens()`: Modifies logits based on the current possible tokens. These three low-level methods support scenarios where the engine's states need updating during prefill stages.
6. `engine.reset()`: Resets the engine's internal states to their initial conditions.
7. `engine.clone()`: Clones the engine’s internal states but does not clone vocabulary and grammar.
    - Assuming vocabulary and grammar are immutable once loaded, sharing their immutable references can prevent unnecessary allocations.

## Conclusion

Designing a practical engine for constrained decoding is an exciting experience, revealing how many intricacies are involved in creating a library that, at first glance, appears straightforward. Most challenges stem from the our requirement that the engine must be *practical* for real-world applications. In the future, I plan to implement this design and will write a similar post on its implementation.

[^1]: Regular expression in this post refers to the regular expression as it is defined in wikipedia. Many programming languages have extended regular expression to include features such as arbitrary lookarounds and recursion, which essentially turns it into context-free or context-sensitive languages and is almost impossible to implement efficiently.
[^2]: Optional operators are already supported in the EBNF standard.
[^3]: The standard exception symbol, per ISO standard, is a weird thing and has too much limitations to be useful.
