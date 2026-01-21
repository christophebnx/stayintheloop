---
title: "Why I'm Writing About Clean Python and AI-Assisted Coding"
date: 2025-01-19
draft: false
tags: ["python", "ai", "clean-code", "data-engineering"]
description: "From 10 years of SAP BODS to full Python — why I care about code ownership and using AI tools thoughtfully."
---

I spent 10 years as a SAP BODS consultant. It taught me a lot — especially about the frustration of working inside a black box. When something broke, I often couldn't dig into the internals. I had to work around limitations rather than through them. The tool did the job, but I never felt like I truly understood my own pipelines end to end. And let's just say that debugging a data flow through a GUI with cryptic error messages at 2am — while the documentation loads slower than your will to live — builds character. A lot of character. If you've ever opened a support ticket hoping for answers and received "working as designed," you know exactly what I mean.

Three years ago, I switched to full Python. The difference wasn't just the language — it was the relationship with my code.

With Python, I can read every line of my pipeline. I can test it, version it, refactor it, deploy it wherever I want. When something breaks, I open the code and figure it out. No abstraction I can't pierce. That depth of understanding changed how I approach data engineering. It's not just about moving data from A to B. It's about building systems you know well enough to trust — and to fix at 2am when needed.

Now AI coding assistants are everywhere. And I see a risk: people generating code faster than they can understand it. Different tools, same problem — building things you don't fully own because you can't fully explain them.

**This blog exists because I believe two things:**

1. **Clean code is about ownership.** When you write code you can read, test, and explain, you own it. When you blindly accept generated code, you're back to working with a black box. The bottleneck was never typing speed — it's understanding.

2. **AI-assisted coding is powerful — when you stay in the driver's seat.** I use Claude and Copilot daily. They're tools, not oracles. Sometimes they save me hours. Sometimes they confidently hand me garbage. Knowing the difference is the skill.

## What you'll find here

Honest experiments with AI coding tools — what actually helps, what wastes time, where I got burned. Python patterns that make pipelines readable and maintainable. And the thread connecting both: how to use powerful tools without losing sight of what you're building.

I'll share the failures too. The times I trusted generated code I didn't fully understand and paid for it later.

## Who this is for

Devs who want to use AI thoughtfully, not blindly. Data engineers who care about understanding their systems deeply. Anyone who thinks "it works" is the starting point, not the finish line.

## Who this is NOT for

The "10x productivity with one prompt" crowd. If you want hype, there's plenty of that elsewhere.