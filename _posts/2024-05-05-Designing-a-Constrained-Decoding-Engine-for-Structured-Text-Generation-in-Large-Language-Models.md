---
title: "Designing a Constrained Decoding Engine for Structured Text Generation in Large Language Models"
date: 2024-05-05
---
## Motivation

Structured text generation means to create text in specific formats, such as JSON, markdown, HTML, or code. Typically, prompting a model to produce text in these formats works well *most of the time*. But what if you need it to work perfectly *every time*? That’s where constrained decoding comes into play. It prevents the generation of unwanted tokens by adjusting the model's output probabilities. Combined with prompt engineering, constrained decoding ensures the output strictly follows the desired format.

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

Todo.
