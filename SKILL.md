---
name: book-to-skill
description: Converts a technical book (PDF or EPUB) into a structured Claude Code skill — extracting frameworks, mental models, principles, techniques, and anti-patterns the author crystallized. Use when the user wants to study a book through Claude, apply an author's frameworks while working, or build a reusable knowledge base from any PDF or EPUB.
when_to_use: Trigger phrases — "turn this book into a skill", "create a skill from this PDF", "create a skill from this EPUB", "I want to study X book", "add this book to my skills", "convert PDF to skill", "convert EPUB to skill", "analyze this book", "extract frameworks from this book". Accepts a path to a PDF or EPUB and optional skill name slug.
disable-model-invocation: true
context: fork
agent: general-purpose
allowed-tools: Bash(python3 *) Bash(pdftotext *) Bash(mkdir *) Bash(cp *) Bash(find *) Bash(wc *) Bash(echo *) Bash(cat *) Bash(date *) Read Write Glob Grep
argument-hint: <path-to-pdf-or-epub> [skill-name-slug]
arguments: [book_path, skill_name]
effort: high
---

# Book-to-Skill Converter

Transform written knowledge into actionable Claude Code skills by extracting structure — not producing summaries.

## Philosophy

Books contain crystallized expertise: frameworks, principles, and techniques that took years to develop. This skill extracts that knowledge into a format Claude can leverage repeatedly.

**Extract structure, not summaries.** A skill isn't a book report. It's a toolkit of:
- Named frameworks (mental models with clear application)
- Actionable principles (rules that guide decisions)
- Techniques (step-by-step methods)
- Anti-patterns (what to avoid and why)
- Voice calibration (how the author thinks and communicates)

**Preserve the author's precision.** Frameworks often have specific names for reasons. "The 5 Whys" isn't interchangeable with "ask why multiple times." Capture the exact formulation.

**Layer depth appropriately.** Simple books → simple skills. Complex books with 10+ frameworks → skills with reference files and on-demand chapters.

---

## Modes of Operation

Three paths available. Route based on what the user asks:

### 1. Full Conversion (Default)
**Trigger:** User provides a PDF path without special instructions
**Action:** Run all steps below (Steps 0–9)
**Output:** Complete skill with SKILL.md, chapters/, glossary, patterns, cheatsheet

### 2. Analyze Only
**Trigger:** User says "analyze", "just extract", or "I want to review before generating"
**Action:** Run Steps 0–3, then produce a structured extraction report (frameworks, principles, techniques found). Stop — do NOT generate skill files.
**Output:** Analysis report for user review

### 3. Generate from Prior Analysis
**Trigger:** User has existing analysis notes or previously ran analyze-only
**Action:** Skip Steps 0–3, use the provided analysis as input, run Steps 4–9
**Output:** Skill files from the provided analysis

---

## Step 0 — Out-of-scope check

If the argument is NOT a path to a PDF or EPUB file, stop and respond:
> "book-to-skill requires a PDF or EPUB path. Usage: `/book-to-skill /path/to/book.pdf [skill-name]` or `/book-to-skill /path/to/book.epub [skill-name]`"

---

## Step 1 — Validate input

```bash
test -f "$0" && echo "FILE_OK" || echo "FILE_NOT_FOUND: $0"
file "$0" | grep -iE "pdf|epub|zip" && echo "FORMAT_OK" || echo "FORMAT_UNKNOWN"
```

Check the file extension (`.pdf` or `.epub`) or magic bytes (`%PDF` or `PK` zip header).

**If the file is not found and the argument looks truncated at a space** (a common failure: the user passed an unquoted path like `/books/A Philosophy of Software Design.epub` and only `/books/A` came through), try to recover it:

```bash
find "$(dirname "$0")" -maxdepth 1 \( -iname "$(basename "$0")*.pdf" -o -iname "$(basename "$0")*.epub" \) 2>/dev/null
```

If exactly one match is found, tell the user which file you found and proceed with it (mentioning they can re-run with a quoted path if you guessed wrong). If multiple match, list them and ask. If none, stop.

If the file is not found or the format is not supported, stop with a clear error message listing supported formats.

---

## Step 1.5 — Identify book type

Before extracting, ask the user:

> "What kind of content does this book have? This helps me choose the best extraction method.
>
> 1. **Technical** — has code blocks, tables, formulas, diagrams (e.g. programming books, academic papers, architecture guides)
> 2. **Text-heavy** — mostly prose, few or no tables/code (e.g. management, productivity, narrative non-fiction)
> 3. **Not sure** — I'll use the fast method and warn you if quality seems limited"

Store the answer as `BOOK_TYPE`:
- Option 1 → `BOOK_TYPE=technical`
- Option 2 → `BOOK_TYPE=text`
- Option 3 → `BOOK_TYPE=text`

**If `BOOK_TYPE=technical`**, inform the user before proceeding:
> "📐 Technical mode selected — using Docling for structure-aware extraction (tables, code blocks, formulas preserved as markdown). This takes ~1.5s per page, so expect a few minutes for longer books. Starting now…"

**If `BOOK_TYPE=text`**, inform:
> "📄 Text mode selected — using fast extraction (pdftotext). Ready in seconds."

---

## Step 2 — Extract text from PDF or EPUB

Run the extraction script, passing the book type:

```bash
python3 ~/.claude/skills/book-to-skill/scripts/extract.py "$0" --mode <BOOK_TYPE>
```

- `--mode technical` → uses Docling (layout-aware, preserves tables and code blocks as markdown)
- `--mode text` → uses pdftotext → PyPDF2 → pdfminer fallback chain (fast, plain text)

This creates:
- `/tmp/book_skill_work/full_text.txt` — full extracted text
- `/tmp/book_skill_work/metadata.json` — title, estimated pages, token count, size, extraction_mode

Read `/tmp/book_skill_work/metadata.json` to understand what was extracted.

---

## Step 2.5 — Pre-flight cost estimate

Read `/tmp/book_skill_work/metadata.json` and present the user with an estimate **before doing any generation**:

```
📖 Book detected: <filename> (<format: PDF or EPUB>)
📄 Pages/Spine items: ~<N> | Chapters: ~<N> | Words: ~<N> | Source tokens: ~<N>K

📊 Estimated token usage (Full Conversion):
   Input  (book reading + prompts): ~<N>K tokens
   Output (skill files generated):  ~<N>K tokens
   Total:                           ~<N>K tokens

   ⏱  Estimated time: ~<N> minutes

📁 Files to be generated:
   SKILL.md + <N> chapter files + glossary + patterns + cheatsheet

➡  Proceed with Full Conversion? (or type "analyze only" to preview first)
```

**How to estimate:**
- Input tokens ≈ `estimated_tokens` from metadata × 1.3 (prompts overhead per chapter pass)
- Output tokens ≈ chapters × 1,000 + 4,000 (SKILL.md) + 4,500 (glossary + patterns + cheatsheet)

**On cost:** do NOT present a dollar figure by default. Users on subscription plans (Pro/Max) are not billed per token — the work only draws against their usage limits, and a dollar estimate reads as a bill. Only if the user asks what it would cost, or says they use the pay-per-token API, translate using current API prices (as of 2025: Sonnet in=$3/MTok out=$15/MTok, Haiku in=$0.80/MTok out=$4/MTok) and label it clearly as "API-equivalent — not billed on subscription plans".

Wait for the user to confirm before proceeding. If they say "analyze only", switch to Mode 2.

**Reduce round-trips:** present this confirmation together with the Step 4 (purpose) and Step 5 (skill name) questions as a single multi-question prompt, so the user answers everything at once and the rest of the conversion runs uninterrupted.

---

## Step 3 — Analyze book structure

Read the first 8,000 characters of `/tmp/book_skill_work/full_text.txt` to identify:
- Book **title** and **author(s)**
- **Chapter structure** (look for "Chapter N", "PART I", numbered headings, table of contents)
- **Core themes** and subject domain
- Approximate number of chapters

Then read the Table of Contents section if present to map all chapters.

**Verify the chapter count independently — `chapters_detected` in metadata.json is a heuristic.** Cross-check against the ToC, and map the actual body chapter boundaries with something like:

```bash
grep -nE "^Chapter [0-9]+$" /tmp/book_skill_work/full_text.txt
```

Beware two traps: (a) ToC entries and in-prose cross-references ("Chapter 18 discusses...") also mention chapter numbers — a true heading is a line containing *only* the marker, followed shortly by a title line, and body headings appear *after* the front matter; (b) line-wrapped prose can put "Chapter N" at the start of a line. When in doubt, read a few lines after each candidate heading to confirm it starts a chapter rather than referencing one. Record the line offsets of each confirmed chapter start — Step 7 uses them.

**If mode is "Analyze Only":** produce the extraction report now and stop. Structure:
```
## Extraction Report — <Title>

### Author's Core Frameworks
- **<Framework Name>**: <what it is and when to apply>

### Key Principles
- <Principle>: <actionable rule>

### Techniques & Methods
- <Technique>: <step-by-step or how-to>

### Anti-patterns
- <What to avoid>: <why>

### Suggested Skill Name
`{author-lastname}-{core-concept}` — e.g. `cialdini-influence`

### Chapters Detected
| # | Title | Main Frameworks |
```

---

## Step 4 — Ask purpose (Full Conversion only)

Before generating, ask the user:

> "What should this skill help you do? (Pick one or more)
> 1. Apply the author's frameworks while working
> 2. Think with the author's mental models
> 3. Reference specific chapters and concepts
> 4. All of the above"

Use the answer to weight what gets highlighted in the SKILL.md Core section.

---

## Step 5 — Determine skill name

If `$1` was provided, use it as the skill slug.
Otherwise, propose two options and let the user choose:
- **By author-concept**: `{author-lastname}-{core-concept}` (e.g. `cialdini-influence`, `meadows-systems`)
- **By title**: lowercase hyphens from book title (e.g. `designing-data-intensive-apps`)

Default to author-concept format if the book has a strong methodological identity.

Check that `~/.claude/skills/<skill_name>/` does NOT already exist.
If it does, append `-2` or ask the user before overwriting.

---

## Step 6 — Create skill directory structure

```bash
mkdir -p ~/.claude/skills/<skill_name>/chapters
```

---

## Step 7 — Generate chapter summaries

**TOKEN BUDGET RULE — CRITICAL:**
- Each chapter summary file: **800–1,200 tokens** (dense, not verbose)
- Files are loaded on-demand — they are NOT capped per se, but keep them useful and tight

For EACH chapter/major section identified in Step 3:

Read the corresponding section of `/tmp/book_skill_work/full_text.txt` (use the line offsets recorded in Step 3).

**Pipeline the work to cut wall-clock time:** in each response, WRITE the chapter files for sections you have already read while READING the next batch of chapters — Write and Read calls are independent, so issue them in parallel in the same message. Read 3–5 chapters per batch rather than one at a time.

**Mine the back matter first if it exists:** many books include author-written summaries (e.g. "Summary of Design Principles", key-takeaways appendices, glossaries). Read those early — they are the author's own distillation and should anchor the cheatsheet and SKILL.md rather than your reconstruction.

Create `~/.claude/skills/<skill_name>/chapters/ch<NN>-<slug>.md` using the structure below.

**Adapt emphasis based on `BOOK_TYPE`:**
- `technical` → prioritize "Code Examples", "Reference Tables", and "Commands & APIs" sections; preserve exact syntax
- `text` → prioritize "Frameworks Introduced", "Mental Models", and "Key Takeaways"; skip empty technical sections

```markdown
# Chapter N: <Full Title>

## Core Idea
<1–2 sentences: the single most important thing this chapter teaches>

## Frameworks Introduced
- **<Framework Name>**: <exact formulation — preserve the author's naming>
  - When to use: <specific situation>
  - How: <steps or criteria>

## Key Concepts
- **<Term>**: <precise definition in 1 sentence>
(5–10 most important terms from this chapter)

## Mental Models
<2–4 frameworks or thinking tools. Write as "Use X when Y" or "Think of X as Y">

## Anti-patterns
- **<What to avoid>**: <why it fails>

## Code Examples *(technical books only — omit if BOOK_TYPE=text)*
<!-- Copy the most instructive snippet from the chapter. Preserve indentation exactly. -->
```<language>
<key code example from this chapter>
```
- **What it demonstrates**: <one line>

## Reference Tables *(technical books only — omit if BOOK_TYPE=text)*
<!-- Reproduce any comparison matrix, parameter table, or decision table from the chapter in markdown. -->

## Key Takeaways
1. <Actionable insight>
2. <Actionable insight>
3. <Actionable insight>
(3–7 takeaways a practitioner must remember)

## Connects To
- **Ch N**: <why this chapter relates>
- **<Concept>**: <external concept or standard it connects with>
```

---

## Step 8 — Generate supporting files

### glossary.md
Create `~/.claude/skills/<skill_name>/glossary.md`:
- Every significant term from the book, alphabetically sorted
- Format: `**Term** — definition (Ch N)`
- Max 1,500 tokens

### patterns.md
Create `~/.claude/skills/<skill_name>/patterns.md`:
- All concrete techniques, design patterns, algorithms from the book
- Format: `## Pattern Name\n**When to use**: ...\n**How**: ...\n**Trade-offs**: ...`
- Max 2,000 tokens

### cheatsheet.md
Create `~/.claude/skills/<skill_name>/cheatsheet.md`:
- Decision tables, comparison matrices, quick-reference rules
- The content you'd want on a single printed page
- Max 1,000 tokens

---

## Step 9 — Generate the master SKILL.md

**CRITICAL TOKEN BUDGET: Keep SKILL.md body under 4,000 tokens.**
Compaction truncates from the END — put the most important content FIRST.

Create `~/.claude/skills/<skill_name>/SKILL.md`:

```markdown
---
name: <skill_name>
description: Knowledge base from "<Full Title>" by <Author(s)>. Use when applying <author>'s frameworks for <key topics, 3–6 terms>.
when_to_use: <10–15 trigger phrases based on book topics and terms. Comma-separated.>
allowed-tools: Read Grep
argument-hint: [topic, framework name, or chapter number]
---

# <Full Title>
**Author**: <Author(s)> | **Pages**: ~<N> | **Chapters**: <N> | **Generated**: <YYYY-MM-DD>

## How to Use This Skill

- **Without arguments** — `/skill-name` loads core frameworks for reference
- **With a topic** — `/skill-name replication` → I find and read the relevant chapter
- **With chapter** — `/skill-name ch05` → I load that specific chapter
- **Browse** — ask "what chapters do you have?" to see the full index

When you ask about a topic not covered in Core Frameworks below, I will read
the relevant chapter file before answering.

---

## Core Frameworks & Mental Models
<!-- ~2,000 tokens: the author's most important named frameworks and principles.
     Preserve exact names. Write as "Use X when Y", "Prefer X over Y because Z".
     This is a toolkit, not a summary. -->

<generate 2,000 tokens of the most critical frameworks and insights here>

---

## Chapter Index

| # | Title | Key Frameworks |
|---|-------|----------------|
| [ch01](chapters/ch01-<slug>.md) | <Title> | <framework1>, <framework2> |
| [ch02](chapters/ch02-<slug>.md) | <Title> | <framework1>, <framework2> |
...

## Topic Index

<!-- Alphabetical. Major terms/frameworks → chapter(s) that cover them. -->
- **<Term>** → ch<N>[, ch<N>]
- **<Term>** → ch<N>

## Supporting Files

- [glossary.md](glossary.md) — all key terms with definitions
- [patterns.md](patterns.md) — all techniques and design patterns
- [cheatsheet.md](cheatsheet.md) — quick reference tables and decision guides

---

## Scope & Limits

<!-- Make this section carry real information, not boilerplate. Include:
     the publication year and anything dated about it (example languages,
     tooling); and any positions that are the author's contrarian opinion
     rather than field consensus, so Claude presents them as the author's
     view instead of established fact. -->
This skill covers the book content only. For hands-on implementation in your codebase,
combine with project-specific tools. For topics beyond this book, check related skills
or ask Claude directly.
```

---

## Step 10 — Cleanup and report

```bash
rm -rf /tmp/book_skill_work
```

Then report to the user:

```
✅ Skill created: ~/.claude/skills/<skill_name>/

📚 Book: <Full Title> — <Author>
📄 Pages: ~<N> | Chapters: <N>

Files generated:
  SKILL.md         — core frameworks + index   (~X tokens)
  chapters/        — <N> chapter summaries     (~X tokens each, ~X total)
  glossary.md      — key terms                 (~X tokens)
  patterns.md      — techniques & patterns     (~X tokens)
  cheatsheet.md    — quick reference           (~X tokens)
  ─────────────────────────────────────────────────────
  Total skill size: ~X tokens (loaded on-demand, not all at once)

💡 Tip: run /cost in Claude Code to see the actual token usage for this session.

Usage:
  /<skill_name>                    → load core frameworks
  /<skill_name> <topic>            → find and explain a topic
  /<skill_name> ch<N>              → dive into a specific chapter
```

---

## Quality Rules

1. **Extract structure, not summaries** — capture named frameworks, exact formulations, anti-patterns; not chapter recaps
2. **Preserve the author's precision** — "The 5 Whys" ≠ "ask why multiple times"; keep exact naming
3. **Density over completeness** — a 1,000-token summary beats a 10,000-token excerpt
4. **Practitioner voice** — write "Use X when Y", not "The book explains X"
5. **Front-load SKILL.md** — compaction keeps the first 5,000 tokens; most important content comes first
6. **Chapter files are on-demand** — they don't count against skill budget until loaded
7. **Never copy raw book text** — always synthesize, summarize, extract signal
8. **Topic index is critical** — it's how Claude navigates to the right chapter file
