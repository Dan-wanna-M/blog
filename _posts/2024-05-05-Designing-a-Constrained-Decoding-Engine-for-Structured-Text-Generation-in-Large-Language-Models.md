---
title: "Designing a Constrained Decoding Engine for Structured Text Generation in Large Language Models"
date: 2024-05-05
---
## Motivation

Structured text generation means to create text in specific formats, such as JSON, markdown, HTML, or code. Typically, prompting a model to produce text in these formats works well *most of the time*. But what if you need it to work perfectly *every time*? That’s where constrained decoding comes into play. It prevents the generation of unwanted tokens by adjusting the model's output probabilities. Combined with prompt engineering, constrained decoding ensures the output strictly follows the desired format. Moreover, if constrained decoding limits the possible outputs to a single token, there's no need to call the model again, which can [significantly speed up the generation process](https://blog.dottxt.co/coalescence.html#orga014f18).

### Why do we need an engine? Can't we just handcode it?

While it might seem straightforward, implementing constrained decoding correctly is challenging. Consider a *simple* scenario where we need the model to output in the format `XXXXX*[YYYY]*`. Here, `XXXXX` should not include `*[` and `YYYY` should exclude both `*[` and `]*`. It is tempting to quickly write:

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

We can see even handcoding a simple format leads to many nuances. Now, imagine handcoding constrained decoding for full JSON. This example should demonstrate why handcoding constrained decoding in general is *not* a good idea.

### Why Not Just Use Out-of-the-Box Solutions Like Guidance, Outline, Jsonformer, etc.?

Out-of-the-box solutions such as Guidance, Outline, Jsonformer, and others, can be incredibly useful and might perfectly meet your needs. However, they might not be the best fit if you:

- Prefer not to incorporate a Python runtime into your application.
- Need a faster engine.
- Require very fine-grained control over the process.
- Want to revive and apply the arcane knowledge you gained in high-level computer science courses, such as compiler design, from 10 years ago.

## Designing the engine

### Goals

- Users should be able to easily use the engine to specify commonly used formats or patterns.
- The engine should implement formats in the most efficient way. In other words, if a user does not use extra expressiveness, then they should not incur unnecessary overhead.
- Users should have the flexibility to select their preferred options when implementation choices involve compromises.

### Format notation language

Before selecting a format notation language, we should ask ourselves: "[How much expressiveness do we need?](https://en.wikipedia.org/wiki/Chomsky_hierarchy)" Programming languages, the most complex yet practical formats in reality, can provide an insight into the maximum expressiveness needed. [They turn out to be mostly context-free.](https://cs.stackexchange.com/questions/140078/are-modern-programming-languages-context-free) Then, a variant based on [Extended Backus-Naur Form(EBNF)](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form) is a good choice:

- It is a wided used(and arguably easy to use) metaformat to define format even before LLM era.
- It is independent from any specific programming languages.
- Popular libraries like [Outlines](https://github.com/outlines-dev/outlines?tab=readme-ov-file#using-context-free-grammars-to-guide-generation), [Guidances](https://github.com/guidance-ai/guidance?tab=readme-ov-file#context-free-grammars) and [Llama.cpp](https://github.com/ggerganov/llama.cpp/blob/master/grammars/README.md) all use some variants of EBNF.
- It is extensible and hence allows us to define some context-sensitive stuffs for ease of use.

Specifically, besides all the standard features of EBNF, we also want:

#### Standard regular expression[^1] support

```ebnf
digits ::= #'[0-9]'; (*A nonterminal that accepts one digit*)
(*Yes, this weird thing is a comment in ebnf*)
```

Adding regular expression improves usability, since they are already commonly used in programming. It also enables potential optimizations since regular expression can be compiled into [determistic finite automata(DFA)](https://en.wikipedia.org/wiki/Deterministic_finite_automaton).

#### Regex-like operators extension [^2]

```ebnf
digits ::= #'[0-9]';
digits ::= digits+; (*A nonterminal that accepts one or more digit*)
```

```ebnf
aaa ::= 'a'*; (*A nonterminal that accepts zero or more 'a'*)
```

Many EBNF variants already use this extension, enhancing usability.

#### except!([]) special support

Todo.

[^1]: Regular expression in this post refers to the regular expression as it is defined in wikipedia. Many programming languages have extended regular expression to include features such as arbitrary lookarounds and recursion, which essentially turns it into context-free or context-sensitive languages and is almost impossible to implement efficiently.
[^2]: Optional operators are already supported in the EBNF standard.
