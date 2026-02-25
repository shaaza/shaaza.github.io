---
layout: page
title: "Week 5: User Testing & Feedback"
description: "Find a real person. Have them try your OS. Listen."
permalink: /level-1/week-05/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-1/">Level 1</a> / Week 5'
---

## Objective

You have been building in isolation for three weeks. This week, you stop building and start **listening**. Find a real person — a classmate, a friend, a stranger at a coffee shop — and have them try your OS. Watch what happens. Write down what you learn.

This is not a nice-to-have step. It is the most humbling and educational part of the entire project.

## Why User Testing Matters

You know how your OS works because you built it (or rather, you directed its construction). You know which commands exist, what the limitations are, where the bugs hide. A real user knows none of this.

> "Everyone has a plan until they get punched in the face." — Mike Tyson
>
> Everyone has a working OS until a real user tries to use it.

User testing reveals:

- **Assumptions you did not know you made** — "Of course you type `ls` to list files... wait, you tried `dir`?"
- **Error messages that make no sense** — Clear to you, cryptic to everyone else
- **Missing features you thought were optional** — "How do I go back? There is no undo?"
- **The difference between "works" and "usable"** — Technically correct is not enough

## How to Run a User Test

### Step 1: Find Your Tester

You need **one real person** who is not you. Ideal candidates:

| Tester Type | What They Reveal |
|---|---|
| **Fellow student in this course** | Whether your OS makes sense to someone with similar context |
| **CS student not in this course** | Whether your system is learnable without your specific background |
| **Non-technical friend** | Whether your shell and error messages are comprehensible to humans |
| **Experienced developer** | What a practiced eye notices about design and robustness |

Any of these will work. The important thing is that it is **not you**.

### Step 2: Set Up the Test

Prepare the environment before your tester arrives:

```bash
# Make sure the OS builds and boots cleanly
make clean && make
./os

# Prepare a clean environment
# Reset any state from your development sessions
# Have the OS ready to boot with a fresh start
```

**Do not give a tutorial.** Give your tester the OS and say:

> "This is a minimal operating system I built. It has a shell where you can type commands. Try to figure out what it can do. I'll watch and take notes but I won't help unless you're completely stuck."

### Step 3: Observe

While your tester uses the OS, **take notes** on:

| Observation | Example |
|---|---|
| **What they try first** | Did they type `help`? `ls`? `exit`? Something unexpected? |
| **Where they get confused** | "What does this error mean?" "How do I...?" |
| **What they expect to work but doesn't** | Tried `cd` but you only have a flat filesystem |
| **What delights them** | "Oh cool, it has colors!" "That error message is funny!" |
| **Where they give up** | The moment they stop trying is your most important data point |
| **Questions they ask** | Every question is a missing feature or unclear interface |

### Step 4: The Debrief

After 15–20 minutes (do not torture your tester for longer), ask:

1. "What was the first thing you tried to do?"
2. "What was the most confusing moment?"
3. "What did you expect to work that did not?"
4. "If you could change one thing, what would it be?"
5. "Would you use this again? Why or why not?"

**Write down their answers verbatim.** Do not paraphrase, do not editorialize, do not defend your design choices. Just listen.

## Processing Feedback

### The Feedback Taxonomy

Categorize every piece of feedback:

| Category | Description | Example | Priority |
|---|---|---|---|
| **Critical** | The OS crashed or became unusable | "It froze when I typed a long command" | Fix in Week 6 |
| **Usability** | User could not figure out how to do something | "I didn't know `help` existed" | Fix in Week 6 |
| **Missing** | User expected a feature that does not exist | "There's no way to see file contents" | Evaluate — add if feasible |
| **Polish** | Rough edges that do not block usage | "The prompt is ugly" | Nice to have |
| **Delight** | Something the user enjoyed | "I love the boot messages" | Keep and amplify |

### Prioritization

You cannot fix everything. Prioritize ruthlessly:

1. **Crashes and freezes** — these are unacceptable in a shipped product
2. **First 30 seconds** — if the user cannot figure out what to do in 30 seconds, nothing else matters
3. **Your differentiation feature** — if users do not notice or use your unique feature, it needs work
4. **Everything else** — nice to have, but not blocking

### Using Your Agent to Process Feedback

```bash
claude

> I user-tested my OS and got the following feedback:
> [paste your notes]
>
> Here is my current shell code:
> [paste shell.c]
>
> Based on this feedback, suggest specific changes to improve
> the first-time user experience. Focus on:
> 1. What should happen when the OS boots (first thing the user sees)
> 2. How to make the help system more discoverable
> 3. Better error messages for the top 3 errors encountered
```

## What This Week Teaches

User testing teaches skills that cannot be learned any other way:

| Skill | What You Learn |
|---|---|
| **Humility** | Your beautiful architecture means nothing if people cannot use it |
| **Empathy** | Other people think differently than you do |
| **Observation** | Watching someone struggle teaches more than any code review |
| **Prioritization** | You cannot fix everything; you must choose what matters most |
| **Feedback loops** | Build, test, learn, repeat — this is how good products are made |

> "The purpose of user testing is not to validate your design. It is to discover your blind spots."

## Deliverables

By the end of Week 5:

- [ ] At least one user test conducted with a real person
- [ ] Raw observation notes from the testing session
- [ ] Debrief answers recorded verbatim
- [ ] Feedback categorized and prioritized
- [ ] Plan for Week 6 polish based on feedback
- [ ] Brief reflection: What surprised you most about watching someone use your OS?

---

<p style="text-align: center;">
  <a href="/level-1/week-04/">&larr; Week 4: Iterate & Differentiate</a> · <a href="/level-1/week-06/">Week 6: Polish & Ship &rarr;</a>
</p>
