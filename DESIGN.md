# C++ Project Builder — Design Doc (v0.1, first pass)

## 1. What this is

A CLI tool that teaches C++ by walking a learner through a sequence of small,
real projects instead of isolated exercises. An LLM (Claude, via API) acts as
the tutor: it explains concepts as they come up, reviews the learner's actual
code, gives hints instead of answers, and decides when a project is "done
enough" to move on.

The learner writes code in their own editor. The CLI's job is to hand them a
project brief, watch what they build, build/run/test it for them, and mediate
between their code and the LLM tutor.

## 2. Goals

- Teach C++ (language + toolchain + basic engineering habits: git, tests,
  debugging) through projects a learner would actually want to have built.
- Make the tutor feel like a patient pair-programmer, not a quiz app: it
  reacts to the learner's real code, not multiple-choice answers.
- Work entirely from the terminal, on the learner's own machine, with their
  own editor.
- Be usable by someone who has never installed a compiler.

## 3. Non-goals (for v1)

- Not a general-purpose IDE or in-browser editor.
- Not language-agnostic — C++-specific tooling (CMake, compiler diagnostics,
  sanitizers) is a feature, not something to abstract away.
- Not a fully autonomous "write the project for me" tool — the LLM reviews
  and hints, it doesn't write the learner's solution.
- No multiplayer/classroom features (cohorts, leaderboards) in v1.

## 4. The learning loop — what actually happens

This is the core UX question, so it gets spelled out step by step. Concretely,
a session looks like this:

```
$ cppt start          # first run: onboarding, detects/installs toolchain
$ cppt next           # "here's your next project"
$ cppt status         # "here's what's expected of your current project"
$ cppt check          # "build it, run tests, ask the tutor to review"
$ cppt ask "why did that segfault"
$ cppt hint           # ask for a nudge without giving up progress
```

This is explicitly a **homework model, not a live-tutoring model**: there's
no lecture and no synchronous session. Each project is a self-contained
assignment — primer, then the work, then a check-in for feedback — done
entirely on the learner's own schedule. This matters for someone with zero
C++ background: the tool cannot assume prior lecture material exists
anywhere else, so the primer in step 1 below is not optional polish, it's
the only place new concepts get taught before they're needed.

Step by step, for one project:

1. **Primer.** Before the brief, `cppt next` prints a short (5-10 minute
   read) explanation of *only the new concepts this project needs* —
   e.g. project 05's primer covers arrays and structs, assuming everything
   from projects 01-04 is already known. This is written content (LLM-
   generated from a per-project outline in the spec, so it can be
   regenerated with fresh examples/analogies if the first phrasing doesn't
   land — worth a `cppt explain --again` or `cppt ask` follow-up right
   there if it doesn't click), not a link out to a textbook.
2. **Brief.** Immediately after, `cppt next` prints the project brief and
   writes a `PROJECT.md` + starter files into a new directory (e.g.
   `projects/03-text-adventure/`). The brief states the goal, constraints
   (e.g. "no `<vector>` yet, you haven't learned it"), and what "done" means
   (a checklist, not just "it compiles").
3. **Learner writes code** in their own editor, at their own pace — this is
   the "homework" itself. The CLI isn't watching keystrokes; it only cares
   when invoked. No deadlines, since this is self-paced by design.
4. **`cppt check`** compiles the project (via CMake/a configured build),
   runs any provided tests, and captures compiler output, warnings, and test
   results. This is fast, deterministic, and free — no LLM call for a plain
   syntax error the compiler already explained clearly. This is effectively
   "turning in" the assignment, and it's meant to be run as many times as
   needed — resubmission is free and expected, not penalized.
5. **Tutor review.** If it builds and passes basic checks, the CLI sends a
   diff (or the full small project) plus the check results to the LLM tutor
   with a system prompt describing this project's learning goals and the
   learner's history (concepts already taught, common mistakes so far). The
   tutor responds with: what's good, what's fragile or unidiomatic, and 1-3
   targeted questions or hints — never a rewritten solution unless asked.
   This is the "grading," but formative, not punitive: no score, just
   feedback and (per step 7) a completion gate.
6. **Conversation.** The learner can `cppt ask <question>` at any point —
   about a compiler error, a concept, or "why is my code slow." The tutor
   has the project context and recent check output loaded automatically.
7. **Hints are opt-in and tiered.** `cppt hint` gives progressively more
   specific nudges (concept → pointer to the relevant section of their code
   → pseudocode), so asking for help doesn't skip straight to the answer.
8. **Completion.** The tutor (plus objective checks: builds clean, tests
   pass, meets the checklist) marks the project complete. The CLI records
   this in local progress state and unlocks `cppt next`.
9. **Between projects**, a short recap: what concepts this project actually
   exercised, what's coming next and why (e.g. "your text adventure hardcoded
   rooms in if/else — next project introduces `std::map` so you can do this
   properly").

So the answer to "how do I learn": you write real code against a spec, the
compiler and test runner give you fast objective feedback, and the LLM gives
you the slower, contextual feedback a human mentor would (this is idiomatic
but fragile, you're about to hit a lifetime bug, here's a question to make
you think about ownership) — without ever just handing you the fix.

## 5. Curriculum structure

- **Tracks**: ordered sequences of projects (e.g. "Foundations" →
  "Data Structures & Memory" → "Systems & Performance" → "A Real App").
- **Project spec** (data, not code): id, prerequisites, learning objectives,
  a **primer outline** (the list of new concepts the tutor must explain
  before the learner starts — see §4 step 1; this is what makes the tool
  usable with zero prior C++ knowledge instead of assuming a lecture already
  happened), constraints (allowed/disallowed features, since a project early
  in "Foundations" shouldn't quietly let the learner reach for `std::vector`
  before it's taught), starter files, a checklist of observable
  done-criteria, and hint tiers.
- Specs are plain files (YAML/JSON + markdown brief) shipped with the tool,
  versioned like content, not code — this makes it possible to add/edit
  projects without touching the CLI itself.
- Projects escalate in size: single-file exercises early on, small multi-file
  CMake projects by the middle track, one longer capstone at the end of each
  track that forces integration of everything learned so far.
- See [`curriculum/foundations.md`](curriculum/foundations.md) for a sketch
  of the first track: 10 projects, a worked example of the project spec
  format, and open questions specific to sequencing it.

## 6. Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌───────────────────┐
│   Learner    │────▶│   cppt CLI       │────▶│  Build/Test runner │
│ (editor, sh) │     │ (Rust or Python) │     │ (CMake/compiler,   │
└─────────────┘     │                  │     │  sandboxed exec)   │
                     │  - project state │     └───────────────────┘
                     │  - progress db   │
                     │  - prompt builder│────▶┌───────────────────┐
                     └──────────────────┘     │   LLM Tutor        │
                                               │ (Claude API)       │
                                               └───────────────────┘
```

- **CLI**: single static binary or `pipx`-installable package. Owns project
  scaffolding, invoking the build, and orchestrating calls to the LLM.
- **Build/test runner**: shells out to CMake + the system compiler (or a
  bundled toolchain via a container/devcontainer for a zero-install path).
  Captures stdout/stderr, exit codes, and (later) sanitizer output.
- **Sandboxed execution**: learner code actually runs, so running it (not
  just compiling) needs a boundary — resource/time limits at minimum,
  container or seccomp sandbox if this ever runs code that isn't the
  learner's own machine's problem (see §8).
- **LLM tutor**: stateless per call; the CLI is responsible for assembling
  context (project spec, learning history, recent diff + check output) into
  the prompt. No fine-tuning — steering is via system prompt + retrieved
  curriculum state.
- **Progress store**: local file (SQLite or JSON) tracking per-project
  status, concepts taught, mistakes seen repeatedly (so the tutor can call
  back to them), and timestamps. Local-only for v1 — no account/server.

## 7. Tech stack (proposed)

- **CLI language**: Rust (single binary, no runtime dependency to install —
  important since the whole point is lowering setup friction for C++
  beginners who may not have Python/Node either) or Python if shipping speed
  matters more than zero-dependency install. Recommend **Rust** for v1 given
  the audience already needs a systems toolchain installed.
- **Build system for learner projects**: CMake, since it's the ecosystem
  standard and worth teaching alongside the language.
- **LLM**: Claude via the Messages API. Tutor prompt is a single system
  prompt + curriculum context; no agentic tool use needed on the LLM side
  for v1 (the CLI does the "tool use" of compiling/running, not the model).
- **Progress storage**: SQLite (simple, inspectable, no server).

## 8. Open questions / risks

- **Toolchain setup friction.** Getting a working C++ compiler + CMake on a
  beginner's machine (especially Windows) is itself a common dropout point.
  Needs either a strong `cppt doctor`/setup wizard, or a devcontainer/Docker
  path as the default recommended install.
- **Sandboxing learner-run code.** Early projects will have infinite loops,
  bad pointer math, etc. Running with timeouts and (on Linux/Mac) basic
  resource limits is a v1 requirement, not a nice-to-have.
- **Cost/latency of LLM review on every `check`.** May want to only call the
  tutor when the build/tests pass, and cache/skip review on trivial re-runs.
- **Preventing the tutor from just writing the solution** when asked
  directly ("just give me the code") — needs explicit system-prompt policy
  and probably a tiered-refusal behavior, not a hard block (sometimes seeing
  one line of idiomatic code is the right teaching move).
- **Evaluating "done."** Compiler success + tests passing is necessary but
  not sufficient (code can be ugly-but-working). The tutor's qualitative
  judgment is part of "done" — needs a clear rubric per project so this is
  consistent rather than vibes-based.

## 9. Phase plan for building this tool

1. **Spike**: one hardcoded project, CLI does scaffold → build → single LLM
   review call. Validate the core loop feels good before building curriculum
   breadth.
2. **Curriculum v1**: 8-12 projects covering a "Foundations" track end to
   end; spec format finalized.
3. **Robustness**: sandboxed execution, progress persistence, `cppt doctor`
   setup flow.
4. **Curriculum breadth**: remaining tracks, capstones.
5. **Polish**: recap summaries, hint tiering, revisiting past mistakes.

## 10. Explicitly deferred to a later version

- Web/GUI front end.
- Multi-user/classroom features.
- Fine-tuned or locally-run model option.
- Auto-grading beyond what's described in §4/§8.
