---
title: "Solutions Looking for Problems: A Field Guide to Fake Disruption"
date: 2025-01-22
draft: false
summary: "Project Elephant didn't exist. But you've probably worked on it."
tags: ["data-engineering", "project-management", "anti-bullshit", "cloud-migration", "disruption"]
---

Let me introduce you to Project Elephant. It doesn't exist — but you've probably worked on it.

A decade-old on-premises data warehouse. Slow queries that everyone complains about but nobody fixes. Cryptic business logic scattered across hundreds of stored procedures that nobody dares/wants to touch. Source data quality issues that get "fixed" with ridiculously creative WHERE clauses, joins on nested sub-queries and some "mapping tables" manually fed by who knows who. Documentation that was "planned for Q2" three years ago.

Then one day, leadership announces the solution: cloud migration. Modern stack. Scalability. AI-ready infrastructure. Digital transformation. The budget is approved for more than the value of my dream house and a team of good looking consultants is hired in to make it happen.

Six months later, the migration is complete. There's champagne. The project manager gives a nice speech that is both emotional and funny. There's a company-wide email celebrating the team. LinkedIn posts with "proud to announce" and rocket emojis. Project Elephant is declared a success.

Except nothing has changed.

The data contracts that didn't exist on-prem? Still don't exist on Databricks. The business logic hidden in stored procedures? Now hidden in unversioned and undocumented notebooks. The dirty source data? Still dirty, just sitting on S3 instead of Windows Sharedrive. The slow queries? Maybe a bit faster, nobody really knows, now burning through cloud compute credits at an alarming rate.

They moved the furniture. They didn't fix the plumbing.

This is what happens when "disruption" means changing technology instead of solving problems. And after fifteen years in data engineering, I've seen this pattern destroy more budgets and sanity than any technical debt ever could.

## The Word That Means Nothing

"Disruption" has become the corporate equivalent of "synergy" — a word so overused it now signals the opposite of what it intends. When a consultant says "we need to be disruptive," what they usually mean is: "I have a solution looking for a problem, and your budget looks fat enough."

And that's the fundamental issue with most "disruptive" projects: they start backwards. Someone sees a demo at a conference, reads a Gartner report, or gets pressured by a vendor with a target to hit. Suddenly there's momentum for a solution — Databricks, Snowflake, a shiny new LLM integration — before anyone has clearly articulated what problem needs solving. The technology comes first. The why comes later.

This is how you end up with a €500k cloud migration that nobody asked for, solving problems that weren't priorities, while the actual pain points — the ones users complain about every day — remain untouched.

Here's a quick translation guide for your next strategy meeting:

| What they say | What they actually mean | What it costs when it fails |
|---------------|------------------------|----------------------------|
| "We need to disrupt our data infrastructure" | We want to buy new tools | 6 months of dev, nothing in prod |
| "AI-driven transformation" | We'll add a chatbot somewhere | €200k and a disillusioned team |
| "Modernize our stack" | Replace old problems with shinier ones | A migration that migrates the problems too |
| "Become data-driven" | We want dashboards | €150k in BI tools nobody uses |
| "Build an AI-ready infrastructure" | We have no idea what we need yet | A platform waiting for a use case that never comes |

The tragedy isn't that these projects fail. It's that they succeed — technically. The migration completes. The new platform goes live. The boxes get checked. But six months later, the same people are having the same frustrations, just with different error messages.

Real disruption isn't about technology. It's about solving problems that were previously unsolvable, or solving them so much better that the improvement is undeniable. And undeniable means one thing: measurable.

## Disruption, Defined (For Real This Time)

I'm going to propose a working definition. Not because the world needs another framework, but because having a concrete definition lets you call bullshit when you see it.

**Disruption (n.):** A change that delivers measurable ROI within six months, integrates with existing systems incrementally, and can be easily reversed if it fails.

That's it. Three criteria. If a project doesn't meet all three, it's not disruption — it's gambling with someone else's money.

Let's break it down:

**Measurable** means you can put a number on the improvement before you start. Not "better user experience" — that's a feeling. I mean: "Reduce report generation time from 4 hours to 20 minutes" or "Eliminate 15 hours per week of manual data validation." If you can't quantify the current pain, you can't prove you've fixed it.

**Incremental** means it works alongside what exists today. The moment someone says "we need to rebuild from scratch," your bullshit detector should start screaming. Real disruption slots into the current workflow, proves its value, then expands. Revolution sounds exciting; evolution actually ships.

**Reversible** means you have a rollback plan that takes hours, not months. If the new system fails and you're stuck with it anyway, that's not innovation — that's a hostage situation.

Here's a formula I propose to evaluate projects:

```
Disruption Value = (Hours Saved × Frequency × Hourly Cost) - Implementation Cost
```

If that number isn't positive within six months, the project isn't disruptive. It's expensive.

## The Autopsy of Project Elephant

Let's go back to our fictional-but-familiar migration project and examine what actually went wrong.

### What Was Promised

The pitch deck was beautiful. Cloud-native architecture. Infinite scalability. 40% cost reduction within two years. Self-service analytics for business users. "AI-ready" infrastructure (whatever that means). The consulting firm had case studies. The vendor had a slick demo. Leadership was convinced. What could possibily go wrong ?

Budget: A lot - Timeline: 6 months. Success criteria: "Successful migration to cloud platform."

Notice anything missing? There's no mention of the actual problems users face daily. No baseline metrics. No definition of what "better" looks like beyond "it's in the cloud now."

### What Was Delivered

Technically, the project succeeded. Every table was migrated. Every pipeline was rebuilt in Spark. The old servers were decommissioned. The go-live happened only one month late — practically on time.

### What Actually Migrated

Here's what nobody talked about in the success presentation:

**The data quality issues came along for the ride.** The source systems still sent garbage data. The difference is that garbage now flows faster and costs more to store. The business users who complained about bad reports? Still complaining. Now they just blame Databricks instead of Oracle.

**The undocumented business logic found a new home.** Those mysterious stored procedures with names like `sp_fix_customer_data_v3_FINAL_Jan2002_Quentin`? Their SQL code wrapped in a `spark.sql()` call, copy-pasted into notebooks that aren't version-controlled nor documented. Not even a rewrite — just the same spaghetti with a new plate. The tribal knowledge is still tribal. It's just running on more expensive hardware.

**The performance problems evolved.** Queries that took 10 minutes on-prem now take... 10 minutes. Sometimes less, if the query run on a bigger cluster. The old system was slow because of bad indexing. The new system is slow because of bad partitioning. Nobody analysed why it was so slow before the migration, hoping that more CPU would solve the problem.

**The organizational dysfunction remained intact.** The team that didn't write documentation before? Still doesn't. The stakeholders who changed requirements mid-sprint? Still do. The data producers who don't care about data quality? Still don't.

**And here's the dirty secret nobody mentions:** the business users never asked for this migration in the first place. The project was driven by IT leadership, maybe a new CTO wanting to make a mark, or a vendor relationship that needed justifying. The actual users — the analysts running reports, the finance team waiting for their dashboards — were handed a vague "we're modernizing, it'll be better" and told to make do. 

When the technical team asked for requirements, they got "just make it work like before, but faster." No specifications. No priorities. No acceptance criteria. Just "do your best." And six months later, when the new system doesn't match expectations that were never articulated, guess who gets blamed?

The platform changed. The problems didn't.

### What Should Have Happened

Imagine an alternate timeline. Same starting point, different approach.

**Month 1-2: Fix the actual problems on-premises.**

Start with the top three user complaints. In Project Elephant's case: slow reports, bad data quality, and inability to trace data lineage. 

For the slow reports, analyze the actual queries. Find the five worst offenders. Add proper indexing and partitioning to the existing system. Cost: maybe €20,000 in consulting time. Impact: 70% improvement in query performance, measurable within two weeks.

For data quality, implement validation at ingestion. Use Pydantic or Great Expectations to catch garbage before it enters the warehouse. Cost: one or two sprints of development. Impact: 80% reduction in "bad data" support tickets within a month.

For lineage, add basic documentation in the initial requirements and implement column-level tracking with existing tools. Cost: two weeks of work. Impact: audit questions now take hours instead of days.

**Month 3: Measure the improvement.**

Now you have baselines. Now you can prove that problems were solved. Now you have credibility.

**Month 4-6: Migrate if still necessary.**

Maybe the cloud migration still makes sense for scalability. But now you're migrating a system that works, with documented patterns, proven practices, and clear metrics. The migration becomes a platform change, not a Hail Mary.

Problems actually solved: yes. Team confidence: high. Rollback plan: trivial, because the on-prem system is not broken.

The disruption wasn't the cloud. The disruption was finally fixing the problems everyone had learned to live with.

## The Quick Checker: 10 Questions Before You Say Yes

Before you approve, join, or pitch a "disruptive" project, run it through these questions. Be honest. If you can't answer with specifics, that's your answer.

**The Problem (you need all three)**

Can you describe the problem in one sentence without using the words "modernize," "transform," or "innovate"? Can you quantify what this problem costs today in hours or euros? Have the people who suffer from this problem actually asked for this solution?

**The Solution (you need all three)**

Can you explain the solution to a non-technical stakeholder in under a minute? Can you deploy a working "minimum viable solution" in less than four weeks? Is there a simpler, cheaper alternative you've seriously considered?

**The Execution (you need all four)**

Do you have baseline metrics documented before starting? Do you have a rollback plan that takes less than one day? Is the team who will maintain this involved in building it? Will you measure success with business outcomes, not technical milestones?

**Scoring:** If you answered "no" or "I don't know" to more than three questions, stop. The project isn't ready. It might be a good idea, but good ideas with bad execution become expensive failures.

**Automatic disqualifiers (one is enough to walk away):**

The ROI timeline is longer than twelve months. There's no rollback plan. The primary justification is "our competitors are doing it." The people affected by the change weren't consulted.

## The Real Disruption

Project Elephant failed because it confused motion with progress. A lot of money was spent. A lot of work was done. Dashboards were rebuilt, pipelines were rewritten, infrastructure was modernized. But the users who struggled before the migration struggled after it, just with different tools.

True disruption is boring. It's fixing the slow query that's been annoying people for three years. It's adding validation that catches bad data before it corrupts reports. It's writing the documentation that lets someone else understand the system. It's small, measurable, reversible improvements that piles over time.

The next time someone pitches you on disruption, ask them one question: "What metric will be different in three months, and by how much?"

If they can't answer immediately and specifically, you're not looking at disruption. You're looking at Project Elephant, wearing a different name tag.

