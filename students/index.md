---
layout: page
title: "Student Pages"
description: "Each student maintains a project portfolio here. Documentation is part of the work."
permalink: /students/
breadcrumb: '<a href="/">Schedule</a> / Students'
---

## The FabAcademy Model

This course borrows its documentation model from [FabAcademy](https://fabacademy.org/), the global digital fabrication program run by Neil Gershenfeld at MIT's Center for Bits and Atoms.

In FabAcademy, every student maintains a personal website hosted within the course site. Each week, they document what they designed, what they built, what broke, and what they learned. The documentation *is* the portfolio. There are no essays, no slide decks, no final reports — just an accumulating record of real work.

We do the same thing here.

---

## How It Works

Every student gets a subdirectory under `/students/`. The structure looks like this:

```
/students/
  alice/
    index.md          ← Alice's main portfolio page
    week-01/
      index.md        ← Week 1 documentation
    week-02/
      index.md        ← Week 2 documentation
    ...
  bob/
    index.md          ← Bob's main portfolio page
    week-01/
      index.md
    ...
```

- **`/students/alice/`** — Alice's portfolio landing page. Overview of projects, links to weekly documentation, brief bio.
- **`/students/alice/week-01/`** — Alice's documentation for Week 1. What she built, what commands she ran, what failed, what she learned.

Each page is a Jekyll Markdown file with YAML front matter, just like every other page on this site.

---

## What to Document Each Week

Your weekly documentation should include:

1. **What you set out to do** — The goal for the week, in your own words.
2. **What you actually did** — Commands you ran, code you wrote, decisions you made. Include code snippets, screenshots, terminal output.
3. **What broke** — Errors you hit, dead ends you explored, things that did not work. This is not optional. If nothing broke, you were not pushing hard enough.
4. **What you learned** — The insight, the mental model shift, the thing you now understand that you did not before.
5. **Links to code** — Commit hashes, branches, pull requests. Your documentation should point to your actual work.

> **Documentation is part of the grade.** A working project with no documentation is incomplete. A failed project with excellent documentation of the failure is more valuable than a working project you cannot explain.

---

## Why Documentation Matters

Three reasons:

**1. It forces understanding.** Writing down what you did requires you to actually understand it. "I ran `make` and it worked" is not documentation. "I ran `make`, which invoked `arm-none-eabi-gcc` with `-mcpu=cortex-m0plus`, targeting the RP2040's Thumb instruction set" is documentation.

**2. It builds your portfolio.** By the end of this course, you will have 18+ weeks of documented project work. This is more compelling to employers and collaborators than any resume line item.

**3. It helps others.** Your documentation becomes a resource for future students. When someone hits the same error you hit, your write-up is the thing that unblocks them.

---

## Setting Up Your Student Page

1. **Create your directory:**
   ```
   mkdir -p students/yourname
   ```

2. **Create your index page** at `students/yourname/index.md`:
   ```yaml
   ---
   layout: page
   title: "Your Name"
   description: "Your one-line description"
   permalink: /students/yourname/
   breadcrumb: '<a href="/">Schedule</a> / <a href="/students/">Students</a> / Your Name'
   ---

   Brief bio and overview of your work.
   ```

3. **Create weekly documentation** at `students/yourname/week-01/index.md`:
   ```yaml
   ---
   layout: page
   title: "Week 1 — Environment & Agent Tooling"
   permalink: /students/yourname/week-01/
   breadcrumb: '<a href="/">Schedule</a> / <a href="/students/">Students</a> / <a href="/students/yourname/">Your Name</a> / Week 1'
   ---

   Your documentation here.
   ```

4. **Submit via pull request.** Fork the course repo, add your pages, and open a PR. Your instructor will review and merge.

---

## Student List

*No student pages yet. This section will be updated as students are added to the course.*

<!-- To add a student, add a list item here and create their directory under /students/ -->

---

<p style="font-size: 0.9rem;">
  <a href="/">← Back to Schedule</a>
</p>
