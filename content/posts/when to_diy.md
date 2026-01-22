---
title: "AI-Assisted Coding: Knowing When to Type It Yourself"
date: 2025-01-21
draft: false
tags: ["ai", "productivity", "python", "workflow"]
description: "AI coding assistants are powerful. But the goal is efficiency, not AI usage for its own sake. Here's when I close the tab."
---

AI coding assistants are powerful. I use them daily. But the goal is to be efficient — not to maximize the percentage of AI-generated code in my projects.

Here are five situations where I've learned to close the chat and just write the damn code myself.

## 1. Code you know by heart

```python
# You don't need AI for this
users = [u for u in data if u["active"]]

with open("config.json") as f:
    config = json.load(f)
```

List comprehensions. Context managers. Basic dict manipulations. If you've written these patterns hundreds of times, the time spent typing a prompt exceeds the time spent typing the code.

AI shines when you're reaching for something unfamiliar. Your muscle memory doesn't need assistance.

## 2. Surgical edits in existing code

You need to change a condition. Rename a variable. Adjust three lines of logic.

The context you'd have to provide to the AI — the surrounding code, the intent, the constraints — is wildly disproportionate to the edit itself. By the time you've explained what you want, you could have made the change, tested it, and moved on.

## 3. When you're still exploring

AI needs a clear brief. If *you* don't know what you want yet, you'll generate ten useless variations and waste time evaluating them.

When I'm in exploration mode — trying different approaches, sketching out ideas — I prototype dirty and fast. AI enters the picture later, maybe for a refactor once the direction is clear.

## 4. "Political" code: team conventions and naming

Your team decided on a specific project structure. You have naming conventions. There are patterns that exist for historical reasons no AI could possibly know.

The AI will generate something reasonable-looking that violates three unwritten rules. You'll spend more time correcting it than you would have spent writing it from scratch.

## 5. Critical one-liners

A regex. A validation rule. A business logic calculation.

```python
# Would you trust AI-generated code here without reading every character?
is_valid = re.match(r"^[A-Z]{2}\d{6}$", invoice_id) is not None
```

The risk of a subtle error is too high for a minimal time gain. For these lines, you need to *understand* every character. And if you're going to read it that carefully anyway, you might as well write it yourself.

---

## A recent example: when "wasted" time isn't wasted

I was developing a SQL generation function for data versioning. The specs were convoluted — edge cases everywhere, conditional logic depending on schema metadata.

I spent more time explaining the context to Claude than it would have taken me to write the function myself.

But here's the thing: that conversation forced me to *actually conceptualize* the problem. I had to articulate every edge case, challenge my own assumptions, defend my design choices. The AI didn't write my code — it served as a sparring partner.

**The takeaway:** sometimes the "time lost" isn't lost. But you need to be aware of what you're doing. If your goal is to ship fast, close the chat. If your goal is to clarify your thinking, keep talking — but own that choice.

---

## The flip side: when AI is the right call

This isn't an anti-AI article. Here's when I *do* reach for it:

- **Unfamiliar territory**: a library I've never used, a language feature I'm rusty on
- **Boilerplate**: tests, docstrings, repetitive CRUD operations
- **Refactoring**: "make this more readable" with code I've already validated
- **Rubber ducking**: when I need to explain a problem to think it through (see above)

---

## Final thought

The best developers I know use AI tools *selectively*. They've developed an instinct for when it helps and when it's overhead.

That instinct comes from paying attention. Notice when you're faster with AI. Notice when you're slower. Adjust.

The goal isn't to be pro-AI or anti-AI. The goal is to ship quality code efficiently. Use whatever gets you there.