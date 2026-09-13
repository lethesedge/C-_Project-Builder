# Foundations Track — Project Sketch (v0.1)

Goal: by the end of this track, a learner can read/write plain C++ (types,
control flow, functions, arrays, structs, basic classes, pointers/references,
file I/O) without yet touching the STL containers or templates. Track 2
("Data Structures & Memory") opens by having the learner *rewrite* a couple
of these projects with `std::vector`/`std::map`, so the pain of fixed-size
arrays and manual struct arrays here is deliberate, not an oversight.

Hard constraint across the whole track unless a project says otherwise:
**no `<vector>`, `<map>`, `<string>` algorithms beyond basics, or `new`/`delete`.**
Raw arrays, `std::string` for text (banning it too would make I/O painful for
no pedagogical benefit), and stack-allocated structs only.

## Track overview

| # | Project | Introduces | Builds on |
|---|---------|------------|-----------|
| 01 | Unit Converter | variables, types, arithmetic, `if`/`else`, functions | — |
| 02 | Mad Libs Generator | `std::string`, `getline`, string concatenation | 01 |
| 03 | Number Guessing Game | loops, `rand`, functions returning values, constants | 01 |
| 04 | Text Adventure (single path) | enums, `switch`, program state across turns | 01–03 |
| 05 | Grade Book | fixed C-arrays, structs, array params (pointer decay) | 01–04 |
| 06 | Calculator | error handling, separating logic from I/O | 01–05 |
| 07 | Contact Book | references, pointers (`nullptr`), file I/O | 05, 06 |
| 08 | Grid Game (Tic-Tac-Toe / Life) | 2D arrays, nested loops, pointer/array relationship | 05, 07 |
| 09 | Inventory System | classes, encapsulation, constructors, invariants | 05–08 |
| 10 | **Capstone**: Library Catalog | integrates all of the above | 01–09 |

Each project is small enough to finish in one sitting (roughly 1-3 hours for
a first-time learner); the capstone is sized for a weekend.

## Project sketches

### 01 — Unit Converter
- **Pitch**: convert between at least three unit pairs (e.g. °C/°F,
  km/miles, kg/lb) from a simple text menu.
- **Concepts**: `int`/`double`, arithmetic and casting, `cin`/`cout`,
  `if`/`else` branching, writing your first non-`main` functions.
- **Constraints**: no arrays, no loops required — a menu that runs once per
  program invocation is fine.
- **Done when**: at least 3 conversions are correct to a reasonable
  precision, invalid menu input doesn't crash the program, at least 2
  user-defined functions exist (not everything crammed into `main`).
- **Stretch**: loop the menu until the user chooses to quit.

### 02 — Mad Libs Generator
- **Pitch**: ask for ~5 words (noun, verb, adjective, place, number) and
  splice them into a short story template.
- **Concepts**: `std::string`, `getline` vs `cin >>` (and why mixing them
  bites you), string concatenation/`+=`.
- **Constraints**: no arrays/vectors of words — named variables are fine at
  this size.
- **Done when**: at least 5 substitutions, multi-word input (e.g. "the red
  dog") works via `getline`, output reads as a coherent story.

### 03 — Number Guessing Game
- **Pitch**: computer picks a random number 1-100, player guesses, gets
  higher/lower feedback, program reports how many guesses it took.
- **Concepts**: `while`/`for` loops, `rand`/seeding, functions that return
  `bool`/`int`, `const` for fixed bounds.
- **Done when**: loop terminates on a correct guess or a max-attempts cap,
  higher/lower feedback is accurate, attempt count is tracked and reported.
- **Stretch**: let the *player* pick the number and the program guess
  (introduces the idea of a search strategy — binary search, informally).

### 04 — Text Adventure (single path)
- **Pitch**: a small branching story, 5+ connected rooms, with at least one
  flag/item that changes what choices are available later (e.g. you need
  the key from room 2 to unlock the door in room 4).
- **Concepts**: `enum`/`enum class` for room identifiers, `switch`
  statements, carrying state (current room, inventory flags) across turns
  in a loop.
- **Constraints**: rooms are `enum` cases handled in a `switch`/`if` chain —
  no array/container of room objects yet (that refactor is track 2's job,
  and the doc's §4 example calls this out explicitly as the motivating
  "why do I need `std::map`" moment).
- **Done when**: 5+ rooms are reachable, at least one gate depends on
  earlier-collected state, there's a clean win/lose exit (not just `return
  0` from the middle of a function).

### 05 — Grade Book
- **Pitch**: a fixed roster (say, 30 students max) with name + scores;
  compute class average, min, max, and apply a curve.
- **Concepts**: fixed-size C-arrays, a `Student` `struct`, looping over
  arrays, passing arrays to functions with an explicit size parameter (this
  is the first brush with pointer decay — worth the tutor calling out
  explicitly when it comes up in review).
- **Constraints**: array size is a compile-time constant (`const int
  MAX_STUDENTS = 30;`); no dynamic sizing.
- **Done when**: average/min/max are correct, a formatted table prints
  cleanly, a `curve(Student roster[], int count, double amount)`-shaped
  function exists and is actually used from `main`.

### 06 — Calculator
- **Pitch**: a two-operand calculator (`+ - * /`, integer or float) that
  never crashes on bad input.
- **Concepts**: `switch` on operator, explicit error handling (bad
  operator, divide-by-zero) via return codes or (optionally, as a first
  taste) exceptions, and — the real point of this project — separating
  *pure logic* (`double apply(double a, double b, char op)`) from *I/O*
  (`main`'s prompt/read/print loop), so the logic functions could be
  tested without a terminal.
- **Done when**: all four operators work, divide-by-zero and garbage input
  are handled without crashing or UB, and the arithmetic itself lives in
  functions that don't call `cin`/`cout` directly.
- **Stretch**: parse a full expression with precedence (`2 + 3 * 4`).

### 07 — Contact Book
- **Pitch**: add/search/list/delete contacts (name, phone, email), and
  persist them to a file so they survive between runs.
- **Concepts**: array-of-`struct`, passing structs *by reference* to avoid
  accidental copies, `nullptr`-returning search (`Contact* find(...)`) as
  the first real use of a raw pointer, `<fstream>` for reading/writing a
  simple text format.
- **Constraints**: fixed-size array (e.g. 100 contacts) — no `std::vector`.
- **Done when**: add/search/list/delete all work, data round-trips through
  a file correctly (quit and relaunch, data is still there), the caller of
  `find()` correctly checks for `nullptr` before dereferencing (this is a
  good project for the tutor to deliberately probe with "what happens if
  you search for someone who isn't there?").
- See the fully worked spec below for this one as the format example.

### 08 — Grid Game (Tic-Tac-Toe or Conway's Game of Life)
- **Pitch**: learner picks one — a 2-player Tic-Tac-Toe with win detection,
  or a Game of Life on a fixed NxN grid that steps generations.
- **Concepts**: 2D arrays, nested loops, the array/pointer relationship
  (passing a 2D array to a function), a basic render/update loop.
- **Constraints**: compile-time fixed grid size (3x3 or, say, 20x20 for
  Life) — no dynamic grid resizing.
- **Done when**: Tic-Tac-Toe — correct win/draw detection in all directions,
  no out-of-bounds moves accepted. Life — correct birth/death rules, no
  out-of-bounds reads at grid edges (a natural place to hit and discuss
  off-by-one/boundary bugs).

### 09 — Inventory System (first class)
- **Pitch**: rebuild something like the Contact Book's data model, but as a
  proper `Item` class instead of a bare struct — this is the track's first
  real class.
- **Concepts**: encapsulation (`private` data, `public` methods),
  constructors (including validating input at construction), maintaining a
  class invariant (e.g. quantity can never go negative — every mutating
  method enforces this, not just the call sites), `const` member functions.
- **Constraints**: still a fixed-size array of `Item` objects — deliberately
  so the "I wish this resized itself" itch is still there heading into
  track 2.
- **Done when**: all fields are private with accessors where needed, the
  invariant genuinely cannot be violated through the public interface
  (the tutor should try to break it during review), at least one `const`
  method exists and is used somewhere that needs it (e.g. printing).

### 10 — Capstone: Library Catalog
- **Pitch**: a book catalog with add/find/checkout/return/list, backed by a
  file, built using a proper class (not a bare struct) for `Book`.
- **Concepts**: this project doesn't introduce anything new — it's the
  integration checkpoint. Structs/classes, arrays, references, pointers,
  file I/O, and separating logic from I/O (from project 06) all have to
  work together.
- **Done when**: full CRUD works and persists across runs, `Book` is a
  class with enforced invariants (e.g. can't check out an already-checked-
  out book), and the tutor's review finds the I/O and logic reasonably
  separated (not a graded metric, a judgment call — see design doc §8 on
  "done" being partly qualitative).

## Worked example: full project spec (project 07, Contact Book)

This shows the concrete shape of the `PROJECT.md` + spec file described in
the main design doc (§5–6). One file like this per project ships with the
CLI.

```yaml
id: 07-contact-book
title: Contact Book
track: foundations
prerequisites: [05-grade-book, 06-calculator]
estimated_time: 2h
learning_objectives:
  - pass-by-reference for structs
  - raw pointers and nullptr as a "not found" result
  - basic file I/O with <fstream>
  - text-based serialization of struct data
primer_outline:
  # what cppt next explains BEFORE showing the brief — assumes 05/06 already landed
  - "references (&) vs pointers (*): when to use which, and why passing a struct by reference avoids a silent copy"
  - "what a raw pointer is, and why nullptr means 'points at nothing' rather than being a normal address"
  - "reading and writing plain text files with ifstream/ofstream: open, check success, read line by line, close"
  - "a simple serialization format for this project: one contact per line, fields separated by a delimiter"
constraints:
  disallowed: [std::vector, std::map, "new/delete"]
  allowed_includes: [iostream, string, fstream, cstring]
  max_contacts: 100
starter_files:
  - contact_book.cpp   # main() with a menu loop stub, Contact struct predefined
  - contacts.txt        # empty; created on first save
done_checklist:
  - "add, search, list, and delete all work from the menu"
  - "data written on exit is correctly read back in on the next run"
  - "find() returns a pointer, and every call site checks for nullptr before use"
  - "no out-of-bounds writes even when the contact list is full"
hints:
  tier_1: "What does it mean for search to fail? What could the function return to represent that?"
  tier_2: "Look at how you're returning from find() when no match exists — is there a sentinel value that already means 'nothing here' for a pointer?"
  tier_3: "Return nullptr from find() when no contact matches, and have every caller check `if (result != nullptr)` before dereferencing it."
```

## Open questions specific to this track

- Should project 04 (Text Adventure) and project 09 (Inventory) explicitly
  get **revisited** in track 2's opening projects (rewritten with
  `std::map`/`std::vector`)? Leaning yes — recommend track 2's project 01
  literally be "refactor your project 04 text adventure to use
  `std::map<RoomId, Room>`", so the payoff of the new tool is felt
  immediately against code the learner already knows intimately.
- Is `rand()`/`srand()` (project 03) worth teaching at all given
  `<random>` is the modern-C++ answer? Leaning toward starting with `rand()`
  for simplicity (fewer new concepts in project 03) and having the tutor
  flag it as "fine for now, we'll do this properly with `<random>` later"
  rather than front-loading `<random>`'s more verbose API in project 3.
- Project 06's exception-handling stretch goal might be too early (before
  any exceptions have been taught) — could instead defer exceptions
  entirely to track 2 and have 06 use return-code error handling only.
