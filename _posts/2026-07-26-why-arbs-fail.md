---
title: "Why Most Architecture Reviews Fail"
date: 2026-07-26 07:30:00 -0400
categories: [blog]
tags: [architecture, governance]
excerpt: "Architecture review boards start with good intentions. Six months later, engineers see them as all red tape and no value."
image:
  path: /assets/images/banners/2026-07-26.jpg
  alt: header
header:
  og_image: /assets/images/banners/2026-07-26.jpg
---

[![leaves](/assets/images/banners/2026-07-26.jpg)](https://unsplash.com/photos/green-leaves-make-up-a-dense-dark-wall-TFV0HZXvL7Y)

Large engineering organizations love to create Architecture Review Boards (ARBs).

It starts with good intentions:

> "Let's ensure consistency."
>
> "Let's reduce technical debt."
>
> "Let's avoid teams solving the same problem ten different ways."

Six months later, engineers bemoan the architecture reviews and see it as unnecessary bureaucracy that provides little value. Why?

---

## The Original Goal Was Admirable

Architecture review boards should, in theory, help organizations to:

- share knowledge
- reduce risk
- enforce security
- improve consistency
- prevent expensive mistakes

These are worthwhile objectives. In my experience, the failure is often in the implementation and structure of these reviews.

---

## Failure Points

### Failure #1: Reviews Become Approval Meetings

Most review boards become gatekeepers. In and of itself, this is not necessarily a bad thing. If the goal is to create a quality gate, then you need some formal approval mechanism.

The issue is when the mindset of the ARB becomes "Can we approve this design?" instead of asking "How can we help this team succeed?". This approach immediately drives a wedge between engineering teams and the ARB. The preparation becomes performative. Design docs become templatized to satisfy the committee and it becomes tedious to explore new paths as an engineering team (or solutions architect). Sensing the tension, the ARB itself also feels as though its hands are tied (depending on its members' appetite for confrontation) — feedback gets swallowed because asking questions feels like pushing back.

> **<i class="fas fa-lightbulb"></i>key insight:**  
> When reviews feel like hurdles instead of handrails, engineers stop bringing problems early—and that's when risk actually increases.
> {: .notice--info}

---

### Failure #2: Everything Gets Reviewed

Not every decision needs committee oversight. It becomes a matter of scale. If everything is put through a review, review fatigue sets in for both sides. Architects spend their time rubber-stamping routine changes instead of focusing on decisions that genuinely need scrutiny. And teams waste time and effort for small decisions or under-communicate about big ones.

There's a closely related issue: _ambiguity in what gets reviewed_. I have often heard the phrase "_architecturally significant_ changes warrant a review" followed up with no idea how to formally define what meets those criteria. Ambiguity in the threshold for review sows frustration in engineering teams. I have been on both sides of this problem in my career.

---

### Failure #3: Standards Live in PowerPoint

Many organizations have well-crafted standards. But they're buried in:

- PDFs
- Confluence pages
- Slide decks
- Tribal knowledge

Engineers shouldn't need a meeting to discover best practices.

Standards should be discoverable where work happens—in the source code repositories, in the IDE, in CI pipelines, in pull request templates, in scaffolding tools that generate compliant code by default, etc. If an engineer has to search for the answer, they won't. They'll make the decision that unblocks them fastest and rationalize it later.

Standards that aren’t embedded into the workflow are just documentation (that may or may not be consulted).

### Failure #4: Architects Become Bottlenecks

Even if you decide on your version of "Architecturally Significant" and only review a subset of changes, a large organization with a dozen business units spanning 10k+ engineers can still generate dozens of reviews per week or more. The ARB can quickly become a bottleneck slowing down delivery. When your ARB becomes the thing preventing teams from meeting their KPIs, you strain the engineering/architecture relationship more, draw the ire of leadership, and become less willing to push back effectively, becoming a rubber stamp.

If your architects become the scarce resource, you’ve built the wrong system.

## A Better Model

Admittedly, this post has been a bit of a downer so far — just talking about all the ways ARBs fail. But I think it's important to understand the problems first before we can come up with a model that aims to fix the shortcomings. Now that we have the main issues enumerated, let's start to create a model that addresses them one by one.

I'm going to walk through a real-world example based on an API ARB that I designed and implemented. Though certain implementation details are specific to APIs, I believe this approach could be applied more broadly across architecture and software engineering.

### A Git-Based Model (How it gets Reviewed)

I realized something important:

The bulk of reviews shouldn’t happen in meetings. I really didn't want to have a formal review board meeting to review system architecture and API contracts where the board was seeing the material for the first time. I wanted the process to be asynchronous, and I wanted the process to take place where our engineers were already working.

For us, that was Git. It's distributed. The ecosystem around it allows for asynchronous (as well as transparent and permanent) feedback. And engineers within the organization were already comfortable with it.

At a high level, the idea was to require engineering teams to check in a small number of artifacts for review. This also had the benefit of enabling automation and tooling that would otherwise be impossible with ad hoc review board meetings.

The layout of our 'API Governance Repository' was very simple:

```
api-governance/
├── .github/            # copilot instructions and PR template files
├── apis/               # API artifacts (OpenAPI spec and diagrams) aligned with our CMDB
├── common/             # common domain objects, error responses, etc., to be reused in API contract files
├── linting/            # spectral rules for linting API contracts
├── templates/          # templates for various API types for teams to use when creating new APIs
└── tools/              # automation scripts
```

Engineers would add the required artifacts, and open a pull request, filling out some additional metadata used internally to assess risk of the overall application. This kicked off a forum of asynchronous feedback and automation. With GitHub we had Copilot integrated. So we moved our standards and guidelines into custom Copilot instructions. This was readable for engineers and Copilot — no PowerPoint standards here. We found Copilot could consistently review the artifacts even when they reached multiple thousands of lines in a single pull request. Our automated CI pipeline ran more deterministic checks using the spectral linting rules and SemVer checks on API changes. All of this allowed an ARB with 3 members to review APIs across an organization with 10k+ engineers.

### The Artifacts (What is Reviewed)

First, I wanted clear requirements on what teams needed to bring to a review. This helped the engineering teams because they knew what was expected — no gotcha moments during the process — and it helped the architects facilitating and implementing the review process. Having a normalized set of documents for review cut down on noise and reduced the cognitive overhead of context switching between multiple APIs. We settled on the following artifacts:

1. OpenAPI Specification-compliant document describing the API
2. [C4 Diagrams](https://c4model.com)
   - System Context (static model)
   - Container Diagram (static model)
3. Sequence Diagram (dynamic model) using the actors defined in the C4 container diagram

> **<i class="fas fa-lightbulb"></i>key insight:**  
> We chose these artifacts because of _where_ they were created in the development lifecycle. With contract first API development, having the OpenAPI Spec document and architecture diagrams could come before any code was written. We wanted the reviews to happen before engineers spent weeks or months writing code and becoming emotionally attached to their solutions.
> {: .notice--info}

### Defining Review Criteria (When to Review)

Because teams were required to check in their architecture diagrams and API specifications, technically all changes would trigger a review; from the smallest changes to an API contract to large architectural changes. However, reviews would be 100% automated with two clear exceptions:

1. The API was net new
2. New integrations were defined in the system context diagram or within the container diagram to external systems

If either criterion was met, a formal ARB meeting was scheduled to review the changes.

### The Formal ARB Meeting

In all honesty, this meeting was intentionally designed to be a rubber stamp meeting, which may sound odd. In reality, the purpose of this meeting was not to approve or deny the API (those discussions had already happened in the asynchronous conversations in GitHub). Instead, the idea with the formal review was to give engineers or solutions architects the opportunity to exercise their public speaking and technical communication skills — to give them a venue to confidently articulate the system that they had designed. This step is optional, but it worked well for us and in an age where LLMs are taking on more engineering and architectural work, I think it's incredibly important. The organization needs to know it can count on experts to describe its systems in times of triage and audit.

---

## Closing Thoughts

The most mature architecture organizations don't review more — they review less, because they've invested in standards, automation, and reusable patterns that make the right thing the easiest thing.

The goal isn't to eliminate governance. It's to shift from **approval** to **enablement**. Instead of being the people who say "no," architects become the people who build the rails so teams don't have to ask.

That's harder work upfront. But it scales, and it earns trust.

Architecture should be a platform, not a committee. Build automation and pipelines instead of setting up meeting schedules.
