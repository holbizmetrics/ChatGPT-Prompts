# The Vibe Coder's Series

Three companion guides for shipping software with AI as a collaborator. Audience: vibe coders with little or no formal CS background who keep noticing their code gets messier the faster they ship. Experienced developers will also find the principles hold true — none of this is dumbed down to falsehood.

## Reading order

1. **[Clean Code & OOP Principles](Clean_Code_And_OOP_Principles.md)** — *What to build.*
   Nine principles for code that doesn't rot. Real OOP (with methods + invariants, not just bundled data), composition over inheritance, when NOT to apply each rule, the legacy-refactor triage protocol. Read this first.

2. **[Vibe Coder's Quality Playbook](Vibe_Coding_Guide_Extended.md)** — *How to enforce it with AI.*
   Constrain before you generate. Read AI output skeptically. Throw away when you're stuck (don't argue the anchor). Two tiers — bare discipline (no tools needed) and full CI. Read this once you've internalized the principles.

3. **[Vibe Coder's Security Playbook](Vibe_Coder_Security_Playbook.md)** — *What will break your app in production.*
   Eight failures AI generates confidently (hardcoded secrets, SQL injection, XSS via `innerHTML`, missing validation, open CORS, insecure defaults, crypto misuse, trust boundary confusion). Pre-Ship Security Checklist. CRA + SSDF awareness without panic. Read this before you ship.

## The graduation path

When you've outgrown the security guide — paying customers, EU users, federal contracts, or enterprise B2B asking for SSDF attestation — the [conformity-pipeline](https://github.com/holbizmetrics/conformity-pipeline) project is the production-grade compliance tool. It unifies CRA Annex I + EO 14028/SSDF + license compliance into one evidence pipeline with traffic-light verdicts and signed evidence bundles.

## Why this series exists

Most clean-code guidance was written for human-paced typing, where you naturally re-read what you wrote. AI removes that friction. The principles haven't changed; the enforcement has. These guides cover the same craft from three angles: *what good code is*, *how to get AI to produce it*, and *what to check before shipping it* where someone else's data is at stake.

The unifying mechanism named across all three: **Surface Compliance Without Semantic Compliance** — AI generates output that satisfies the shape of a requested practice without delivering its substance. *"I added auth"* (cookie set) ≠ *actual auth* (password verified, session secure). *"I used OOP"* (`class X: __init__`) ≠ *actual OOP* (methods + invariants). *"I added tests"* (functions exist) ≠ *actual tests* (real coverage). Spotting that mismatch is the master skill.
