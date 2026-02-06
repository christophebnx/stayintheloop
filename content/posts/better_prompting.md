---
title: "Stop Writing Better Prompts. Start Reviewing AI Code"
date: 2026-02-07
draft: false
tags: ["python", "ai", "prompting", "patterns", "data-engineering"]
summary: "Stop chasing the perfect prompt. Start with a draft, review it like code, and iterate with constraints until the output matches your actual design."
---

Most people try to “get better at prompting” by hunting for the perfect wording.

That’s the wrong target.

When you use an AI model for Python development, the winning move isn’t a clever one-shot prompt. It’s a workflow: **draft → review → constrain → refine**.

If you already know how to collaborate with humans in a codebase, you already know how to collaborate with an AI. You just need to treat the first answer as a draft — not as a final solution.

## From “asking a question” to collaborating

A lot of AI-assisted coding frustration comes from a wrong mental model.

We tend to treat the model like search:

> “Write a function that flattens a JSON.”

The model answers. The code looks plausible. Sometimes it even works.

Then reality shows up:
- unclear structure
- hidden assumptions
- edge cases ignored
- design choices you didn’t agree to

That’s not the model “being bad”. It’s you and the model optimizing for different goals.

In real Python work — even something that sounds simple like flattening JSON — the model behaves more like a **very fast junior developer** than an oracle.

That means:
- it follows instructions literally
- it doesn’t know what constraints matter unless you state them
- it optimizes for “a solution”, not for *your* solution

Once you accept that, prompting stops being about asking the right question and becomes about **guiding a reasoning process**.

You don’t start with the perfect prompt.  
You start a loop where:
- the first answer is a draft
- your feedback is the signal
- constraints emerge progressively

This shift matters. Without it, you fight the tool instead of using it.

## Step one: the naïve prompt (and why it fails)

Let’s be honest: your first prompt is usually naïve.

That’s not a mistake — it’s a starting point.

A typical initial prompt for something like JSON flattening might be:

> “Write a Python function that flattens a nested JSON into a flat dictionary.”

The model will happily comply.

What you usually get:
- a recursive function
- a couple of `isinstance(...)` checks
- some key concatenation logic
- maybe a docstring

At first glance, it looks fine.

Then you notice the missing decisions:
- What about lists vs dicts?
- Are lists indexed or exploded?
- Are empty lists meaningful or ignored?
- What happens with root-level scalars?
- Is the output a single dict or multiple rows?

None of this was specified — so the model guessed.

**The model didn’t do something wrong. It filled the gaps you left.**

A naïve prompt produces code that is:
- reasonable
- generic
- loosely correct

And that’s exactly why it’s dangerous: it looks “done” while hiding design choices you didn’t make consciously.

At this stage, the goal is **not** to fix the code.  
The goal is to read it critically and figure out what actually matters.

## Feedback is the real prompt

Once you have a first draft, the most common mistake is to immediately try to “fix” the prompt.

That’s backwards.

The initial prompt has already done its job: it produced something concrete to react to. From now on, **feedback matters more than wording**.

Think about how you’d review a junior developer’s code. You wouldn’t say:

> “Rewrite this, but better.”

You’d point out specific problems.

Effective feedback looks like:
- “This assumes flattening means returning a single dict.”
- “List handling is underspecified.”
- “Edge cases like empty lists or root scalars aren’t defined.”
- “The output shape doesn’t match how the data will be consumed.”

None of this tells the model *how* to fix the code.  
It clarifies *what is wrong or ambiguous*.

Good feedback:
- reduces ambiguity
- makes assumptions explicit
- narrows the solution space

This is where prompting stops being about phrasing and starts being about design.

## Turning feedback into constraints

Once the issues are named, the next iteration becomes much more precise.

Instead of “make it better”, you translate feedback into **constraints**:

- Flattening must produce **rows**, not a single mapping
- Lists and tuples must **explode** into multiple rows
- Nested arrays must produce cartesian combinations
- Empty lists produce **no rows**, not empty values
- Root-level scalars must still produce a usable output

These aren’t implementation details.  
They’re rules.

When you feed rules back into the model, improvisation stops and refinement starts.

And you don’t need all constraints upfront. Most of them only become obvious *after* you’ve seen a concrete draft and reacted to it.

Iteration isn’t a weakness here.  
It’s the mechanism that reveals what actually matters.

## A concrete example: flattening JSON, iteratively

Here’s what this looks like in practice.

### The draft you’ll typically get

A model answering the naïve prompt will usually return something like this:

```python
def flatten(data, parent_key="", sep="_"):
    out = {}
    for k, v in data.items():
        new_key = f"{parent_key}{sep}{k}" if parent_key else k
        if isinstance(v, dict):
            out.update(flatten(v, new_key, sep=sep))
        elif isinstance(v, list):
            for i, item in enumerate(v):
                out[f"{new_key}{sep}{i}"] = item
        else:
            out[new_key] = v
    return out
```
It’s not “wrong”. It’s a reasonable interpretation of “flatten JSON”.

It’s also not what many downstream workflows need.

Problems jump out immediately:
- it assumes the root is a dict
- lists are encoded as indexed keys
- lists of dicts are not handled
- the output shape prevents any notion of explosion or cartesian combinations

This is where iterative prompting matters: you don’t fix syntax, you fix semantics.

### The target behavior (repo-aligned)

In `json-flatten`, flattening is coupled with **explosion**:
- arrays generate multiple rows
- nested arrays produce cartesian combinations
- the return type is `List[Dict[str, Any]]`

Here is the implementation aligned with the actual project logic:

```python
from collections.abc import MutableMapping
from typing import Any, Union, List, Dict, Tuple


def flatten_and_explode(
    data: Union[Dict[str, Any], List[Any], Tuple[Any, ...], Any],
    parent_key: str = "",
    sep: str = "_"
) -> List[Dict[str, Any]]:
    """
    Recursively flattens and explodes a dictionary, list, or tuple
    into a list of flat dictionaries.

    Notes:
        - Empty dict -> returns `[{}]`
        - Empty list/tuple -> returns `[]`
        - Root-level scalars -> `[{"value": data}]`
          unless a parent_key is provided
    """
    if isinstance(data, (list, tuple)):
        result: List[Dict[str, Any]] = []
        for elem in data:
            result.extend(flatten_and_explode(elem, parent_key=parent_key, sep=sep))
        return result

    if isinstance(data, MutableMapping):
        parts: List[List[Dict[str, Any]]] = []
        for k, v in data.items():
            new_key = f"{parent_key}{sep}{k}" if parent_key else k

            if isinstance(v, MutableMapping):
                parts.append(flatten_and_explode(v, new_key, sep=sep))

            elif isinstance(v, (list, tuple)):
                exploded: List[Dict[str, Any]] = []
                for elem in v:
                    if isinstance(elem, MutableMapping):
                        exploded.extend(flatten_and_explode(elem, new_key, sep=sep))
                    else:
                        exploded.append({new_key: elem})
                parts.append(exploded)

            else:
                parts.append([{new_key: v}])

        rows: List[Dict[str, Any]] = [{}]
        for part in parts:
            combined: List[Dict[str, Any]] = []
            for base in rows:
                for fragment in part:
                    combined.append({**base, **fragment})
            rows = combined
        return rows

    return [{parent_key if parent_key else "value": data}]
```

The key difference isn’t recursion technique.  
It’s the **output shape**.

That single decision cascades into everything else.

## Understanding explosion with small examples

```python
flatten_and_explode({"a": {"b": 1}})
```
```python
[{"a_b": 1}]
```

```python
flatten_and_explode({"a": [1, 2]})
```
```python
[{"a": 1}, {"a": 2}]
```

```python
flatten_and_explode({"a": [1, 2], "b": ["x", "y"]})
```
```python
[
    {"a": 1, "b": "x"},
    {"a": 1, "b": "y"},
    {"a": 2, "b": "x"},
    {"a": 2, "b": "y"},
]
```

```python
flatten_and_explode(42)
```
```python
[{"value": 42}]
```

None of this behavior is implied by “flatten JSON”.  
It only emerges once constraints are explicit.

## Why this works especially well for Python

Python is expressive and permissive, which makes it easy to generate code that *looks* correct while being subtly wrong.

Iterative prompting helps by forcing explicit contracts:
- accepted input types
- output shape
- edge-case semantics
- failure modes

The moment the workflow shifts from:

> “generate code for me”

to:

> “here’s why this version doesn’t fit — adjust accordingly”

the tool becomes genuinely useful.

You’re no longer delegating thinking.  
You’re accelerating it.

## Four rules of thumb for iterative prompting

**Treat the first answer as a draft. Always.**  
If you expect production-ready code from the first prompt, you’ll either be disappointed — or worse, misled.

**Feedback beats clever wording.**  
Specific critique is far more powerful than a “better” prompt.

**Turn criticism into constraints.**  
Constraints reduce ambiguity. Ambiguity is where plausible-but-wrong code hides.

**Stop iterating once the design is clear.**  
The AI helps you explore quickly. You still own the final trade-offs and structure.
