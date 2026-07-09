# The Vibe Coder's Quality Playbook: Making AI-Assisted Code Actually Ship

### Why This Exists

You're building with AI. Maybe you came from sysadmin work, maybe you're self-taught, maybe you have a CS degree but never wrote OOP for real. It doesn't matter. The AI generates plausible-looking code *fast*, and speed creates a false sense of quality. You ship confidently. Then it breaks in production, and you realize nobody—not you, not the AI—actually verified anything.

This playbook fixes that. Not by demanding you become a senior engineer first—by giving you the moves that *prevent* the failures instead of catching them after the fact. The cheapest move is the one you make before the AI generates anything. We'll start there.

**Companion read:** *The Hidden Rules of Clean, Scalable Code* covers the principles being enforced here. Read it first if you want the foundations; the references in this playbook will still make sense even if you don't.

---

### The AI-Assisted Failure Mode: Surface Compliance Without Semantic Compliance

Traditional coding is slow enough that you *notice* your own mistakes. You type a function, re-read it, spot the typo. AI-assisted coding removes that friction—and the natural review that came with it. What replaces it is one specific failure pattern:

> **The AI produces output that satisfies the SHAPE of what you asked for, without delivering the SUBSTANCE.**

You asked for "error handling." It produced try/except blocks. ✓ shape. Did anything get handled? ✗ substance.
You asked for "tests." It produced test functions that pass. ✓ shape. Do they test anything meaningful? ✗ substance.
You asked for "OOP." It produced a class with `__init__`. ✓ shape. Methods, invariants, behavior? ✗ substance.

This is one mechanism, not four. Once you can name it, you can spot it everywhere. The four most common instances:

| Instance | Looks like | Misses the substance of |
|---|---|---|
| **Plausible Nonsense** | Code that reads correctly | Actually computing the right thing |
| **Invisible Duplication** | Each block looks fine in isolation | Aggregate maintainability—the system's single source of truth |
| **Cargo-Cult Error Handling** | Try/except blocks scattered everywhere | Failures recovered or surfaced with an action |
| **Testing Theater** | Tests exist and pass green | Code correctness actually constrained by the tests |

The same pattern shows up beyond these four: `async`/`await` syntax without parallelism, type hints without type-driven design, docstrings that just restate the function signature. They're all the same failure.

**Detection skill (memorize this):**
> *"What's the SUBSTANTIVE property the practice exists for? Is the output delivering it, or just matching its shape?"*

This question is your single most valuable habit. Build it in. (The pattern has a name in the broader literature: it's a software-development instance of Goodhart's Law—"when a measure becomes a target, it ceases to be a good measure.")

---

### Principle 0 — Constrain Before You Generate

Most quality guidance is about *reviewing* what the AI produced. Reviews work, but they're expensive: by the time you're reviewing, the AI has already produced 200 lines, you have to read them, diagnose what's wrong, and correct.

**The cheaper move is upstream. Constrain the prompt before generation.**

A constraint in the prompt shifts the AI's default output more than any review checkpoint. A 50-token constraint at the start of session ("use SRP, max 40 lines per function, classes with methods and invariants — not bundling structs") changes the character of the next 2000 lines. No review checkpoint matches that leverage ratio.

This is the classical software-engineering rule "fix bugs at design time, not production"—applied to AI generation. The leverage decays as you move downstream:

```
Prompt constraint   >   Review during gen   >   Tests   >   Production debugging
  (cheapest)                                                       (most expensive)
```

Same logic shows up everywhere:
- Type system at compile time > runtime checks > test failures > prod errors
- Code review > merge > CI failure > customer report
- Architecture decisions > implementation refactor > rewrite > deprecation

**What to put in the prompt:**

| Goal | Add to the prompt |
|---|---|
| Real OOP, not bundling | "Use classes with methods that operate on the data; validate invariants in __init__" |
| SRP holds | "Max 40 lines per function; one responsibility each" |
| Real tests | "Write tests that fail if the function returns wrong output for these specific edge cases: ..." |
| Error handling that handles | "Catch specific exception types; messages must say what's wrong and how to fix it" |
| Stay consistent | "Use snake_case for Python / camelCase for JS. No mixing." |

Some people call these "system prompts," "constitution prompts," or "specification prompts." The label doesn't matter. The habit does: write the constraint *before* the request, every time.

---

### Principle — Persistence Belongs to the Substrate

The AI has no memory of yesterday's session. Whatever you discussed, debugged, agreed on—gone. The model that opens your next chat is the same model that opened yesterday's, with no knowledge of what happened in between.

> **The AI is volatile. Only the substrate (code, tests, docs, comments, lint configs, type hints) persists.**

This sounds obvious until you watch its consequences:

- You found a bug yesterday and explained the fix to the AI. Today, prompted similarly, it regenerates the same bug.
- You spent 30 minutes designing a clean architecture together. Three sessions later, the AI suggests an approach that violates the architecture, because it never saw the design.
- You wrote "always use the existing User class" five times in the chat. New session: AI creates a parallel User class.

The mechanism is structural, not a model flaw. The AI is stateless across sessions by design. The fix is also structural: **any knowledge you want to survive the next session has to be written into the substrate**, not retained in the chat.

| Knowledge type | Substrate location |
|---|---|
| "This bug used to happen" | Regression test |
| "Why we did X" | Code comment / commit message |
| "Use this convention" | Lint config / type signature |
| "This is the architecture" | ADR (architecture decision record) |
| "Domain vocabulary" | Project glossary in `docs/` |
| "Discovered constraint" | Explicit assertion / type / docstring |

Practical corollary: **Bugs that aren't in tests will be regenerated.** Regression testing isn't just defensive in AI-assisted code—it's the *only* memory of what didn't work.

---

### Principle — Read AI Output Skeptically

Bainbridge's *Ironies of Automation* (1983) noticed something counterintuitive: **automation degrades the operator's skill at monitoring the automation.** Pilots who let autopilot fly become less able to spot autopilot mistakes. Reviewers who let the AI write become less able to spot the AI's mistakes.

The fix is to actively build the skill the AI is trying to take from you: **reading code as if you didn't write it.**

**A structured protocol for reading AI-generated code:**

For each function or block:
1. **What does it claim to do?** State the intended behavior in one sentence.
2. **Does the code actually do that?** Read the body. Trace one input through it mentally.
3. **What does it not handle?** Empty input. Wrong type. Missing field. Concurrent modification.
4. **Can I name the failure mode?** Is this *Plausible Nonsense*? *Testing Theater*? Naming the failure makes it concrete; concrete failures are fixable.

This is a skill, not a checklist. Like reading music or proofreading, it gets faster with practice. The Surface Compliance detection question—*"is this delivering substance or just shape?"*—is the muscle to train.

**Push-back move:** Sometimes the AI confidently produces wrong code, you correct it, and the next version is wrong in a similar way. That's a signal—the AI is anchored on its prior output. Don't argue line-by-line. Either restate the problem from scratch in one new prompt, or throw away the session (next principle).

---

### Principle — Throw Away When You're Stuck

LLMs are trained to produce coherent continuations of context. That's mostly a feature. But when the prior context contains wrong output—either yours or the AI's—the coherence pressure pulls the next output toward *more variations of the wrong pattern*.

**Coherence pressure anchors errors.** You'll see it as:
- Re-prompting "fix the bug" produces a similar bug
- "No, do it this way" produces a cosmetically-changed version of the same wrong approach
- A long debugging session keeps generating outputs aligned with whatever you tried first

Recognize this as a known failure mode, and recognize the countermove: **context hygiene.** Throw away the chat. Start a fresh session. Write a tighter prompt that locks down what you want and what you *don't* want.

**When to throw away vs. when to keep iterating:**

| Stay in the session | Throw away and restart |
|---|---|
| The AI's response is close but a small thing is wrong | You've corrected the same kind of thing twice and it keeps coming back |
| You're refining details on a working approach | The whole approach is wrong but the AI keeps tweaking surface details |
| The context has working code you'd lose | The context is mostly your frustration + wrong AI attempts |

The block isn't a technical one—it's sunk cost. You've invested 20 minutes; throwing away feels like loss. **It's not.** A clean restart with what you've learned costs 5 minutes and produces better output than 30 more minutes of correction.

(Aider (open-source) has a `/clear` command; ChatGPT and Claude both let you start a fresh chat. The countermove is common enough to be built in.)

---

### The Five Quality Checkpoints

These are *what to check after the code exists.* If Principle 0 (prompt constraints) is the prevention, these are the inspection. They aren't strictly ordered—but Checkpoint 1 enables the others, and Checkpoint 5 (tools) automates the rest.

#### Checkpoint 1: Code Structure and Readability

**Duplication Detection (DRY).** Across files and sessions, the AI doesn't remember what it already wrote. Flag any block that appears substantially similar in more than one place — *flag for review, not automatic extraction.* Apply the Rule of Three before you abstract (see *Clean Code & OOP Principles* → "When DRY is wrong"); premature extraction is its own failure mode.

*What to look for:*
- Functions with identical logic but different names
- Copy-pasted validation patterns with minor variations
- HTML/template strings repeated across components

**Function Size Enforcement (SRP).** A function over 40 lines is almost certainly doing more than one thing. *Rule of thumb:* If you can't describe what it does in one sentence without using "and," split it.

**Naming Consistency.** AI mirrors whatever convention it sees first in your context. Mixed `camelCase`/`snake_case` will replicate. Pick one per language and enforce it.

**Comment and Docstring Presence.** AI rarely includes meaningful comments unless asked. Minimum bar: every public function gets a one-line docstring; every non-obvious block gets an inline comment explaining *why*, not *what*.

#### Checkpoint 2: Error Handling and Input Validation

**Validate at the boundary.** Every external input—file uploads, API responses, user form data—gets validated before it touches your logic. Don't trust the AI to add this; it usually won't unless you explicitly ask.

```
load → validate → process → output
         ↓
    helpful error message with specific fix instructions
```

**No bare `except:` blocks.** Every `except` names a specific exception type and either recovers meaningfully or fails with a clear message.

**Error messages should be actionable.** *"Error processing file"* tells the user nothing. *"This looks like a Grype report, not a CycloneDX SBOM. Use: `grype -o cyclonedx-json`"* tells them exactly what happened and how to fix it.

#### Checkpoint 3: Testing as Substrate Memory

(Rationale already covered under "Persistence Belongs to the Substrate.")

**Unit tests for the logic the AI generated.** Test the *decisions*—parsing heuristics, edge cases, missing-field paths. AI-generated code is weakest exactly where decisions live.

**Integration tests for the seams.** AI generates modules that *look* like they connect. Prove they actually do. Pass real data through the pipeline.

**Regression tests for bugs you've fixed.** Every bug becomes a test case. Without this, the AI regenerates the same bug.

**Automated test runs on every commit.** Manual testing is what you do once. Automated testing is what protects you forever.

#### Checkpoint 4: Design and UX (When Your AI Builds UI)

Same "plausible but wrong" problem applies to design. The AI produces something that *looks* like a UI but hasn't been thought through as an *experience.*

- **Navigation coherence:** Does the user always know where they are and how to get back?
- **Consistency enforcement:** Same action, same visual treatment everywhere. If "delete" is red on one screen and blue on another, the AI didn't track its own decisions.
- **Accessibility baseline:** Contrast ratios, keyboard navigation, font sizes. Run Lighthouse or axe.
- **Responsive verification:** AI generates for the viewport you tested in. Verify at mobile (375px), tablet (768px), desktop (1440px).

#### Checkpoint 5: Tool Integration and Enforcement

When tools exist, leverage them. Embed quality checks into the generation loop, not after it. After-generation checks become suggestions you can ignore; during-generation checks become constraints the AI works within.

| Check | Trigger | Action |
|-------|---------|--------|
| Duplicate code detection | On every generation | Flag and suggest extraction |
| Function size > 40 lines | On every generation | Split suggestion with proposed boundaries |
| Missing type hints | On save | Auto-suggest signatures |
| No tests for new functions | On commit | Block merge or warn |
| Bare `except:` blocks | On save | Highlight with specific exception suggestion |
| Naming convention drift | On save | Auto-correct or warn |
| Missing docstrings | On commit | Generate draft from function signature |
| Accessibility violations | On UI preview | Flag with WCAG reference |
| Unused imports/variables | On save | Auto-remove or warn |

**Real-time nudges over batch reviews.** A linting report with 47 warnings that arrives after you've written 500 lines is depressing and ignored. A single inline suggestion mid-generation gets acted on.

**Testing status as a merge gate.** Don't let code reach `main` without passing tests.

**Feedback loop: track what breaks.** Log which AI-generated patterns cause the most bugs. Feed that data back into your prompts or tool configuration.

---

### Tiers: With Tools and Without

Most quality guidance assumes you have a CI pipeline, linters, hooks, and a test runner. If you don't, the checklist above is aspirational, not actionable. Here's what to actually do at each tier:

**Tier 0 — Bare Discipline (no tools, just you and the AI):**

1. Always start with a prompt-side constraint (Principle 0). Cheapest, highest leverage.
2. After generation, scan for the four Surface Compliance instances. Just notice if any apply.
3. For anything you'd hate to break, write one test. Even a manual checklist file counts.
4. When stuck for 10+ minutes, throw away and restart.

That's enough. Tier 0 vibe coders ship clean code routinely; many never use tooling.

**Tier 1 — Editor + Linter:**

Add: pylint/eslint/etc. configured for your language. Type hints. Auto-format on save. Pre-commit hooks for the obvious stuff (no bare `except:`, no `print()` left in code).

**Tier 2 — Full CI:**

Add: automated tests on every commit. Merge gates. Test coverage thresholds where they matter. The full Checkpoint 5 matrix.

Move up tiers when the lower tier is no longer enough. Don't skip tiers—each one teaches you what to enforce at the next level.

---

### How Do You Know It's Working?

Quality practices that nobody measures get abandoned. Track this:

- **Fewer hours debugging week-over-week** (subjective, but you'll feel it)
- **Regressions caught before deploy** (count them; each one is a win for tests)
- **Ability to refactor without fear** (clean signal: you'll start doing it more often)
- **AI takes fewer rounds to produce shippable output** (signal: your prompts are better)

**Anti-metric: don't chase "code quality scores" alone.** They optimize for what's measurable, not what's load-bearing.

---

### The Cost of Over-Applying

More tests, more reviews, more checkpoints can also slow you to a stop. The right amount of discipline is *appropriate*, not maximum.

Some signs you've over-applied:
- You spend more time setting up tests than building features
- Every PR cycles 5+ times through review
- Your linter rules are arguments people work around
- Shipping a single bug fix takes a week

When this happens, *relax* a checkpoint. The principles aren't moralistic—they're tools. Use the tools that pay off for *your* project at *its* current stage.

The Quick Fixer is correct for one-off scripts. The Architect is correct for systems that need to last five years. Most code is in between.

---

### The Vibe Coder's Pre-Ship Checklist

Before you ship anything the AI helped build:

1. **Did I constrain in the prompt?** If the output is wrong, was it because the prompt was loose?
2. **Did I scan for the four Surface Compliance instances?** Specifically: is the output delivering substance, or just shape?
3. **Did I validate the inputs?** Not "does it work with good data" but "does it fail helpfully with bad data?"
4. **Can I explain what each function does?** If you can't, you can't maintain it.
5. **Is there a test for the thing that would hurt most if it broke?** One meaningful test beats twenty trivial ones.
6. **Did I check it at more than one input?** AI optimizes for your test case. Reality is wider.
7. **Would I be comfortable if another developer opened this file tomorrow?** If not, clean it now.

---

### Closing

Vibecoding is a superpower. It lets you build in hours what used to take days. But speed without structure is just faster debt accumulation.

The principles haven't changed—the enforcement has. The cheapest enforcement isn't more tools. It's a better prompt, a more skeptical read, and the willingness to throw away the session and start over when you're stuck.

Build the habits before you build the tools. The tools amplify the habits. They can't replace them.

**Companion reads:**
- *The Hidden Rules of Clean, Scalable Code* — the underlying principles: SRP, DRY, OOP that's actually OOP, when not to apply the rules, the legacy-refactor triage protocol. *What to build* to this playbook's *how to enforce*.
- *The Vibe Coder's Security Playbook* — what will break your app even if the code is clean and the tests pass. The eight failures AI generates confidently that ship vulnerabilities to production.
