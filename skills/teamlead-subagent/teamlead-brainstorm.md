# TeamLead Brainstorm Protocol

This file is the detailed protocol for the hard gate summarized in `SKILL.md`.

Use it whenever the user asks for a feature, new component, new functionality, behavior change, UX change, API change, data model change, architecture-shaping refactor, or infra flow design.

Do not use this for read-only inspection, simple status commands, summaries, or bug fixes with a concrete broken symptom. For those bug fixes, use the WF-3 Root Cause flow from `workflows/wf-3.md`.

---

## Core Rule

Brainstorming is a Lead responsibility. Do not delegate it to SA, BA, Dev, or QA.

The goal is to convert the user's request into an approved design that can become a shared spec artifact. The Lead may write orchestration artifacts, but must not edit product code, tests, migrations, or product config.

Hard stop: do not write an implementation plan, spawn Dev/QA, or edit product files until the proposed design is approved by the Human.

---

## Step 1 -- Classify the Request

Decide whether the task needs brainstorming before any workflow selection.

Needs brainstorming:

- New feature or new user/system behavior
- Adding a component, screen, endpoint, job, integration, workflow, or capability
- Changing UX, API contract, data model, permissions, or business behavior
- Refactor that changes architecture boundaries or could affect behavior
- Infra/Docker/CI work that needs a new flow, deployment shape, or policy
- "Small" changes that still introduce or alter behavior

Usually skip brainstorming:

- Read-only research or summary
- Code review without edits
- Status/report commands
- Bug fix where the broken symptom is concrete; use WF-3 Root Cause flow instead
- Mechanical docs/spec cleanup with no product behavior change

If classification is uncertain, default to a short brainstorm. A two-question brainstorm is better than implementing on a hidden assumption.

---

## Step 2 -- Ground in Real Research First

Before asking the user detailed questions, the Research Gate must have run (see `teamlead-research.md`): research agents spawned in the background covering the codebase (relevant files, routes, services, schemas, tests, configs, patterns, recent changes, constraints) and — when the work touches a framework, library, or external API — current external docs via Context7/web search.

The Lead does not do this research inline; the Lead reads the merged `.state/{TASK_ID}/research.md` and works from it. If `research.md` does not exist yet for brainstorm-required work, stop and run the Research Gate before continuing — questions and designs built on guesses are exactly what this protocol exists to prevent.

External PRDs, Notion pages, issue descriptions, and design docs are useful inputs, but they are not canonical for TeamLead execution until their relevant requirements are converted into `.state/{TASK_ID}/spec.md` and approved by the Human.

Then tell the user, based on `research.md`:

- What the research found
- What seems clear
- What is still ambiguous

Keep this summary short — the depth lives in `research.md`; the message to the Human is just the digest that sets up better questions.

### Scope Decomposition Gate

Before refining details, check whether the request spans multiple independent subsystems.

If it does, stop and help the Human split it into sub-project specs. Each sub-project should be independently understandable, plannable, testable, and shippable. Then run the brainstorm loop for the first sub-project only.

Examples that usually need decomposition:

- Billing + onboarding + analytics + permissions in one request
- New API + admin UI + background jobs + third-party integration
- Cross-repo change where provider and consumer can be delivered separately

Do not spend many questions polishing a spec that is too broad to execute safely.

---

## Step 3 -- Ask One Question at a Time

Ask one focused question per message unless the user explicitly asked for a batch.

Order questions by risk: if `research.md` surfaced something that could invalidate the request's premise (a referenced model/feature that doesn't exist, a conflicting contract), ask about that first.

Good questions clarify:

- User or system goal
- Primary workflow
- Success criteria
- Edge cases and failure behavior
- Permission or data visibility rules
- Compatibility with existing behavior
- Non-goals

Prefer multiple-choice questions when it reduces effort for the Human, but include an "other/custom" path in plain language when needed.

Avoid:

- Long questionnaires
- Asking what can be learned by reading the repo
- Asking anything `research.md` already answers
- Asking implementation details the Lead can decide from conventions
- Bundling unrelated product, design, and technical questions together

---

## Step 4 -- Propose Approaches

After enough context is clear, propose 2-3 approaches.

For each approach include:

- What it does
- Main trade-off
- When it is the right choice

Then recommend one approach and explain why. Recommendation is part of the Lead role; do not just list options and make the Human design everything.

Keep the alternatives meaningful. Do not invent weak options just to reach three.

---

## Step 5 -- Present Proposed Design

Present a proposed design scaled to the task. Small changes can be a compact design; larger changes need sections.

Cover the relevant parts:

- Goal
- User/system behavior
- Important flows
- Components/modules affected
- Isolation boundaries and clear interfaces between units
- API/data/contract impact
- Error and empty states
- Testing approach
- Risks and open questions

Design units so each one has one clear responsibility, clear dependencies, and a small public interface. Agents should be able to understand what a unit does without reading every implementation detail.

End by asking for explicit approval or requested changes.

Approval language can be simple:

> Does this design look right? Once you approve it, I will turn it into the TeamLead spec artifact before planning implementation.

If the user changes direction, revise the design and ask again.

---

## Anti-Patterns

Do not skip brainstorming because the task sounds simple. Words like "just", "quick", "small", or "tiny" often hide product assumptions.

Do not start by spawning SA/BA to "figure it out" when the Human intent is still unclear. SA/BA can review or enrich the approved direction, but the Lead owns the first alignment loop.

Do not treat an existing spec as automatically approved. External specs, PRDs, and Notion docs still need gap/ambiguity review, conversion into the local `.state/{TASK_ID}/spec.md` artifact, and Human confirmation before implementation planning.

Do not ask the Human to choose file names, folder structure, or implementation details unless those choices affect product behavior or constraints.

---

## Output Before Spec

Before moving to `teamlead-spec.md`, the conversation must contain:

- Approved proposed design
- Resolved or accepted open questions
- Clear in-scope and out-of-scope boundaries
- Enough acceptance criteria to make QA useful
- Known API/data/contract impact, even if the answer is "none"

If any of these are missing, continue the brainstorm loop.
