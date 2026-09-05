# Writing Skills: A Complete Learning Guide

A practical reference for understanding what skills are, how they work, and how
to write good ones. Read it top to bottom the first time; after that, jump to
the section you need.

---

## 1. What a skill is (the mental model)

A **skill** is a folder containing a `SKILL.md` file that gives Claude
specialized, reusable instructions for a particular kind of task. Think of it
as a playbook: instead of re-explaining "here's how I want you to write release
notes" every time, you write it down once, and Claude consults it whenever the
task comes up.

The key idea to internalize early: **a skill is not loaded all the time.** Claude
only pulls in the full instructions when it decides the skill is relevant. That
decision, and the way content is loaded in stages, drives almost every design
choice you'll make. Section 3 covers this in depth — it's the single most
important concept.

A useful contrast:
- A **prompt** is a one-time instruction for the current task.
- A **skill** is durable, reusable knowledge that activates automatically when
  the right kind of task appears.

---

## 2. Anatomy of a skill

The minimal skill is one file:

```
my-skill/
└── SKILL.md          ← required
```

`SKILL.md` has two parts: **YAML frontmatter** (metadata) at the top, then a
**Markdown body** (the instructions). That's it for most skills.

When a skill grows, you add supporting folders:

```
my-skill/
├── SKILL.md          ← frontmatter + instructions (required)
├── scripts/          ← executable code for deterministic, repetitive steps
├── references/       ← long docs loaded only when needed (specs, big tables)
└── assets/           ← files used in the output (templates, fonts, icons)
```

You only create these folders when the content genuinely warrants it (see
Section 7). Don't add structure for its own sake — a clean single file beats a
cluttered folder.

---

## 3. The three loading levels (progressive disclosure)

This is the heart of skill design. Content loads in **three stages**, and your
job is to put the right thing at the right level.

**Level 1 — Name + description.** *Always* in Claude's context, for every
conversation. This is roughly 100 words and it is the *only* thing Claude reads
when deciding whether to use the skill. Keep it tight and make it earn its place.

**Level 2 — The SKILL.md body.** Loaded *only when the skill triggers*. This is
where your real instructions live. You have room here — aim to stay under ~500
lines, but it's fine to be thorough.

**Level 3 — Bundled resources** (`references/`, `scripts/`, `assets/`). Loaded
*only when the body points to them*. This is where bulk goes: a 600-line API
spec, a sample document, a data table. It costs nothing until something actually
needs it, and scripts can even execute without being read into context.

Why this matters: context is finite and shared across everything Claude is
doing. A skill that dumps 2,000 lines into Level 1 or 2 is wasteful and makes
Claude worse at everything else. Progressive disclosure lets a skill be huge in
total while staying cheap until the moment it's needed.

**Design rule of thumb:**
- Triggering info → Level 1 (description).
- Core procedure → Level 2 (body).
- Reference material, examples, code → Level 3 (folders).

---

## 4. The frontmatter (and why `description` is everything)

```markdown
---
name: my-skill-name
description: What the skill does AND exactly when to trigger it.
---
```

**`name`** — a short, lowercase, hyphenated identifier (e.g. `release-notes`,
`csv-cleaner`). It usually matches the folder name.

**`description`** — the most important field in the entire skill. Claude decides
whether to use a skill based *solely* on the name + description. A perfect body
is useless if the description never triggers it.

Two things the description must do:
1. Say **what** the skill does.
2. Say **when** to use it — specific phrases, file types, and contexts.

**Be a little pushy.** In practice Claude tends to *under*-trigger skills — it
skips them when they'd actually help. To counteract this, write descriptions
that lean toward activation:

> *Weak:* "Builds a dashboard to display internal data."
>
> *Strong:* "Builds a dashboard to display internal data. Use this whenever the
> user mentions dashboards, metrics, data visualization, or wants to display any
> kind of company data — even if they don't explicitly say 'dashboard'."

Notice the strong version lists concrete trigger words and explicitly covers the
case where the user *doesn't* use the obvious keyword. That's the pattern to copy.

---

## 5. Writing the body

The body is plain Markdown. There's no required structure, but a reliable shape
that works for most skills:

```markdown
# Skill Name

## What it does
One or two sentences.

## Steps
1. First do this
2. Then this
3. The result looks like this

## Output Format
Default to <X> unless the user asks otherwise.

## Edge Cases
- Handle <X> like this
- Handle <Y> like that
```

### Writing patterns

**Use the imperative voice.** Write "Extract the endpoints, then list the
parameters" rather than "The skill should extract the endpoints." You're giving
instructions, so phrase them as instructions.

**Pin down output formats explicitly** when consistency matters:

```markdown
## Report structure
Always use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**Include examples** — they're often clearer than description. An input/output
pair communicates a format instantly:

```markdown
## Commit message format
Input:  Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

---

## 6. Writing style and principles

A few principles separate a mediocre skill from a good one.

**Explain *why*, don't just command.** Instead of stacking heavy-handed "MUST"
rules, give the reasoning. "Validate the date format before parsing, because
mixed formats are the most common cause of silent errors here" teaches better
than "MUST validate dates." When Claude understands the intent, it generalizes
correctly to cases you didn't anticipate.

**Write for generality, not for one example.** It's tempting to encode the exact
example in front of you. Resist it. Describe the *category* of task so the skill
works on inputs you haven't seen.

**Draft, then reread with fresh eyes.** Your first pass will have gaps and
awkward phrasing. Write it, step away, read it as if you'd never seen it, and
fix what's confusing.

**No surprises (and no malice).** A skill's behavior should match what its
description implies — nothing hidden. And skills must never contain malware,
exploit code, or anything designed to compromise security, exfiltrate data, or
enable unauthorized access. (Benign creative framing like "roleplay as a
pirate" is fine; deception and harm are not.)

---

## 7. When to use scripts, references, and assets

Default to putting everything in `SKILL.md`. Reach for folders only when:

- **`scripts/`** — the task has a deterministic, repetitive step better done by
  code than by prose reasoning (e.g. reformatting a file, running a fixed
  calculation). Scripts are faster and more reliable, and can run without being
  loaded into context.
- **`references/`** — you have a large body of documentation (a full spec, a long
  lookup table, extended examples) that's only needed sometimes. Move it out and
  point to it from the body: *"For the full field list, read
  `references/fields.md`."* For any reference file over ~300 lines, put a short
  table of contents at the top.
- **`assets/`** — files that go *into* the output, like a `.docx` template, a
  font, or an icon set.

**Multi-domain skills.** If one skill covers several variants (say, AWS vs GCP vs
Azure), keep the shared workflow in `SKILL.md` and put one reference file per
variant. Claude then reads only the relevant one:

```
cloud-deploy/
├── SKILL.md          ← shared workflow + how to pick the variant
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

**The 500-line signal.** If `SKILL.md` creeps past ~500 lines, that's your cue to
add a layer of hierarchy: move sections into `references/` and leave clear
pointers about where to go next.

---

## 8. How triggering actually works

Understanding this helps you write descriptions *and* set realistic
expectations.

Skills appear to Claude as a list of names + descriptions. When a task arrives,
Claude scans that list and decides whether any skill is worth consulting. Two
consequences:

1. **The description carries all the weight.** This is why Section 4 harps on it.
2. **Trivial, one-step tasks often won't trigger a skill at all** — and that's
   correct behavior. "Read this PDF" is something Claude just does; it won't
   reach for a skill no matter how well the description matches. Skills fire for
   tasks complex or specialized enough to benefit from a playbook.

Practical implication: if you're testing a skill, use *substantive* prompts that
a real task would involve. "Read file X" is a poor test case because it won't
trigger any skill regardless of quality.

---

## 9. The iteration loop

Skills are rarely right on the first draft. The proven workflow:

1. **Draft** the skill from your understanding of the task.
2. **Test** it against 2–3 realistic prompts — the kind of thing a real user
   would actually type, not toy examples.
3. **Review** the outputs honestly. Where did it go wrong? Vague? Missed an edge
   case? Wrong format?
4. **Refine** the skill to fix what you saw.
5. **Repeat** until it's solid, then widen the test set and run again at larger
   scale.

When you write test prompts, sanity-check them first: would a real person say
this, and is it substantive enough to actually trigger the skill? Then run them
and look at the results before changing anything — resist the urge to "fix"
based on a guess instead of an observed failure.

---

## 10. Common mistakes to avoid

- **Burying the trigger.** Putting "when to use this" in the body instead of the
  description. Claude doesn't see the body until *after* it decides to trigger —
  so all the "when" info has to be in the description.
- **A polite, vague description.** Politeness reads as low confidence and leads
  to under-triggering. Be specific and a little pushy.
- **Dumping everything into Level 2.** A 1,500-line body that should have been a
  tight workflow plus a couple of reference files.
- **Over-specifying to one example.** The skill works on the sample you tested
  and nothing else.
- **Commands without reasons.** Walls of "MUST" with no explanation, so Claude
  can't generalize to new cases.
- **Skipping the test loop.** Shipping the first draft without running it against
  realistic prompts.

---

## 11. A worked example

```markdown
---
name: api-docs
description: Generate API documentation from code. Use this whenever the user
  wants to document endpoints, write an API reference, or produce an OpenAPI
  spec — even if they just say "document my routes" or "write docs for this."
---

# API Documentation Skill

## What it does
Reads source code or a schema and produces clean API reference documentation.

## Steps
1. Read the provided code or schema.
2. Extract every endpoint, its parameters, and its return types.
3. Output in the requested format (Markdown by default, OpenAPI YAML if asked).

## Output Format
Default to Markdown. Switch to OpenAPI YAML only when the user requests it.

## Edge Cases
- Undocumented params: note them as "undocumented" rather than omitting them.
- Multiple versions: document the latest unless the user names a version.
```

Trace it against the principles: the **description** names concrete triggers and
covers the casual phrasing ("document my routes"). The **body** uses imperative
steps, states an explicit **default** output format, and handles two realistic
**edge cases**. It's short because the task doesn't need references or scripts —
which is exactly right.

---

## Quick reference card

| Thing | Where it goes | Rule |
|---|---|---|
| When to trigger | `description` (Level 1) | Specific + a little pushy |
| What it does | `description` + `## What it does` | One or two sentences |
| The procedure | Body `## Steps` (Level 2) | Imperative voice, under ~500 lines |
| Output shape | Body `## Output Format` | State a default |
| Tricky inputs | Body `## Edge Cases` | Explain *why*, not just *what* |
| Big docs / tables | `references/` (Level 3) | TOC if over ~300 lines |
| Repetitive code steps | `scripts/` (Level 3) | Deterministic work |
| Output templates/fonts | `assets/` (Level 3) | Files used in the result |

**The one-sentence summary:** put the trigger in a specific, slightly-pushy
description; keep the body a tight imperative workflow; push bulk into folders;
and refine by testing against realistic prompts.
