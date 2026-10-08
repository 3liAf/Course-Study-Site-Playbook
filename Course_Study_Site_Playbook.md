# Course Study Site — Build Playbook

A reusable recipe for turning a messy folder of course files into a single
self-contained study site: notes + diagrams + flashcards + quiz + exercises + search.

Built from doing this for HCI (FAU). Every gotcha below cost real time the first
time round — the order of the phases matters more than it looks.

**What you need:** an agent with file access and a shell (Claude Code, Cursor, etc.).
Phases 0–2 need file access. Phase 7 needs a browser or a way to run the page.
A chat-only LLM can do Phases 3–6 if you paste content in manually.

---

## The one idea that matters

**Never let the model invent an answer key.** Almost all the value in this build came
from ranking sources by trustworthiness and saying, per question, which one was used:

| Tier | Source | Trust |
|---|---|---|
| 1 | Official solutions / marked answer keys | Definitive |
| 2 | Self-test banks that state answers in prose | Definitive |
| 3 | Lecture / handout text | High — use to derive |
| 4 | Student transcriptions with checkbox marks | **Do not trust** |

In the HCI build, tier-4 checkbox marks were provably wrong twice, and one exam
transcript had two answers literally swapped. If you skip this step you get a site
that confidently teaches wrong answers — worse than no site.

---

## Phase 0 — Inventory and organise

> **Prompt 1**
>
> Here is my course folder: `<PATH>`. Before changing anything, give me a complete
> inventory: every file with its size, page count, and whether its text is
> extractable or it's a scan. Identify exact duplicates by hash and near-duplicates
> by comparing extracted text. Tell me what each ambiguously-named file actually is
> by reading its first page. Don't move anything yet — just report.

> **Prompt 2**
>
> Now reorganise into: `01_Lectures/`, `02_Past_Exams/<YEAR>/`, `03_Study_Aids/`,
> `04_Papers/`, `99_Duplicates_Safe_To_Delete/`.
> Rules: rename files so the name says what it is; mark scans vs searchable text in
> the filename (`_SCANNED` / `_TEXT`); **move, never delete** — duplicates go to `99_`
> so I decide. Verify the file count is unchanged afterwards and show me the tree.

**Watch for:** lecture files hiding in the exam folder under a different naming
convention. In HCI, two naming schemes made the lecture set look incomplete when it
wasn't, and one lecture was misfiled among the exams — it later turned out to be the
only source for a topic that was examined.

---

## Phase 1 — Find out what's actually tested

> **Prompt 3**
>
> Do the past papers carry topic tags, question IDs, or point values? Extract them and
> build a frequency table of topics across all papers. I want to know where the marks
> actually are, not where the page count is.

> **Prompt 4**
>
> For every past paper, tell me **how you would learn the correct answer**:
> official solution marks / stated answers / derivable from lectures / not determinable.
> Be explicit where you cannot determine the answer. Do not guess.

**Scanned exams:** do NOT reach for OCR. OCR gives you the words but loses **which box
is ticked** — which is the entire answer. Read the pages visually instead.

> **Prompt 5 — reading scans cheaply**
>
> Render the scanned PDFs to images (`pdftoppm -png -r 120 -gray`), auto-crop the white
> margins, and stack 3 pages per image with a page label on each. Then read the stacked
> images. That's ~3× fewer round trips at the same legibility.

**Derive the key, don't read it off.** Marked-up exam reviews usually show the
*student's* answer plus a per-option right/wrong verdict. The key is the combination:

```
ticked  + ✓  →  option is CORRECT
ticked  + ✗  →  option is WRONG
unticked+ ✓  →  option is WRONG
unticked+ ✗  →  option is CORRECT   ← the one people miss
```

Confirm this rule on a question you already know before trusting it on 30 you don't.

---

## Phase 2 — Build the skeleton

> **Prompt 6**
>
> Build a single self-contained HTML study site. Non-negotiable technical contract:
>
> - **One file**, no build step, works offline by double-clicking.
> - **ASCII-only on disk.** Write every non-ASCII character as an HTML entity
>   (`&#8212;`, `&middot;`). This survives being served without a charset header.
>   **Exception:** strings assigned via `.textContent` must use JS escapes (`✓`),
>   because textContent does not decode HTML entities — it would print the literal text.
> - **Theme-aware via CSS custom properties.** Define the full light palette on bare
>   `:root`; redefine only the tokens inside `@media (prefers-color-scheme: dark)`
>   guarded as `:root:not([data-theme="light"])`; redefine again under
>   `:root[data-theme="dark"]`. Never define a colour only inside a media block.
>   Give `body` an explicit token background.
> - **Data-driven.** One `DATA` array of chapter objects; renderers read from it.
>   Adding content must never mean editing a renderer.
> - Modes: Notes / Flashcards / Quiz. Progress in `localStorage`, wrapped in try/catch.

**Chapter object schema** (keep this stable — everything else keys off it):

```js
{
  id: "c1", num: "01", title: "Chapter title", weight: 7,   // weight = exam frequency
  notes: [ {h: "Heading", body: "<p>HTML…</p>"} ],
  cards: [ {q: "Question", a: "<b>Answer</b> HTML"} ],
  quiz:  [ {q, multi: true, opts: [...], correct: [0,2], why: "explanation"} ],
  ex:    [ /* see Phase 6 */ ]
}
```

---

## Phase 3 — Add content in additive rounds

Once the file is a few thousand lines, editing `DATA` in place gets fragile. Use this
pattern instead — it is the single biggest time-saver in the whole build:

> **Prompt 7**
>
> Add new content as a **separate block appended before the state section**, merged in
> programmatically. Never edit the original `DATA` array:
>
> ```js
> var ROUND_N = { c3: [ …items… ], c5: [ …items… ] };
> for (var i = 0; i < DATA.length; i++) {
>   if (ROUND_N[DATA[i].id]) DATA[i].quiz = DATA[i].quiz.concat(ROUND_N[DATA[i].id]);
> }
> ```
>
> Apply it with a Python script that asserts its anchor string exists and asserts the
> round hasn't already been applied. If an anchor is missing, fail loudly — don't
> "find something close".

Each round = one source (one exam year, one paper, one set of exercises).

> **Prompt 8 — the rule for every explanation**
>
> Every quiz explanation must (a) say why the right answer is right, (b) say why the
> most tempting wrong answer is wrong, and (c) carry a small source line naming where
> the answer came from. Explain the *distinction being tested*, not just the fact.

That third part is what makes it a learning tool rather than a memory test.

---

## Phase 4 — Search

> **Prompt 9**
>
> Add a global search. Build one flat index over every note, flashcard, quiz option and
> explanation (strip HTML to plain text once at load). Multi-term AND matching,
> rank title hits above body hits, group results by type, highlight matches with
> `<mark>`, and make each result **jump to the exact item** — the specific flashcard,
> not just its chapter. `/` focuses, `Esc` clears. Debounce ~110 ms.

---

## Phase 5 — Diagrams

> **Prompt 10**
>
> Add inline SVG diagrams for the figures that are hard to learn from prose. Use CSS
> custom properties for every fill and stroke so they work in both themes — no
> hard-coded colours. Give each a `viewBox` and `aria-label`, wrap in an
> `overflow-x:auto` container.
>
> Prioritise diagrams where the *spatial* arrangement IS the content — a law's geometry,
> a 2×2 error matrix, a continuum, anything with labelled axes. Skip decorative ones.

The highest-value diagram in the HCI build was the Gestalt laws drawn as actual dots
and curves. The lowest-value ones were boxes with arrows that restated a list.

---

## Phase 6 — Exercises (the highest-yield mode)

If the course has practice exercises with **no published answers**, this is where the
most value is — nobody else has worked them out.

> **Prompt 11**
>
> Add an Exercises mode with four item types, all self-checking:
>
> - `mc` — multiple choice, per-option feedback
> - `match` — dropdown per row; on check, wrong rows show `→ correct answer` inline
> - `order` — assign positions; wrong rows show the correct position
> - `open` / `calc` — think first, then reveal a model answer or worked solution
>
> For `open` and `calc`, write a **real** answer: the reasoning, the trade-offs, the
> numbers worked through step by step. Not a hint.

> **Prompt 12**
>
> Cross-check the exercises against the past papers. Flag any exercise that appears on
> an exam nearly verbatim — tell me explicitly which ones, because those are the ones
> to memorise.

In HCI, one exercise turned out to be a past exam question **word for word**. That is
worth knowing before the exam, not after.

---

## Phase 7 — Test it properly

Do not eyeball it. Run the page and drive it.

> **Prompt 13**
>
> Serve the folder locally and open the page. Then sweep **every chapter × every mode**,
> asserting: no console errors, no panel renders empty, no `undefined` / `NaN` /
> `[object Object]` in output, and no raw HTML entities leaking into rendered text.
> Report the totals per mode. Then exercise one item of each interactive type end to end,
> deliberately answering partly wrong, and confirm all feedback states render.

```js
// paste into the console — the sweep that caught every real bug
const errs=[]; let counts={};
const modes=[...document.querySelectorAll('.modes button')];
const chaps=[...document.querySelectorAll('#chlist button')];
for (let i=0;i<chaps.length;i++) for (const m of ['notes','cards','quiz','ex']) {
  modes.find(b=>b.dataset.mode===m).click(); chaps[i].click();
  const t = document.querySelector('#p-'+m).innerText.trim();
  if (!t) errs.push(i+'/'+m+' EMPTY');
  if (/undefined|NaN|\[object|&#\d+;|&[a-z]+;/.test(t)) errs.push(i+'/'+m+' BAD TOKEN');
  counts[m]=(counts[m]||0)+1;
}
({errors: errs.length?errs:'none', counts});
```

Also check both themes by reading **computed** styles, not screenshots:

```js
document.documentElement.setAttribute('data-theme','light');
getComputedStyle(document.querySelector('svg text')).fill;  // must contrast with body bg
```

---

## Phase 8 — Host it

> **Prompt 14**
>
> Host it so I can reach it from any of my devices. Check what's already running first
> and **don't disturb it** — mount on a sub-path rather than taking over the root.
> Show me how to update it and how to remove it.

Tailscale, if you have it — private to your devices, no public exposure:

```bash
scp study_site.html server:/opt/course-study/index.html
ssh server 'tailscale serve --bg --yes --set-path=/course /opt/course-study'
# update: just scp again.   remove: tailscale serve --https=443 --set-path=/course off
```

`tailscale serve` = your devices only. `tailscale funnel` = the public internet — don't,
unless you mean it. Serving a directory needs no web server process at all.

---

## Variants — when the course isn't shaped like this one

The playbook above leans on past papers for two separate things: **what gets weighted**
and **what the correct answer is**. Lose those and you need different inputs, not a
watered-down version of the same build.

### Variant A — no past exams available

The failure mode here is specific and dangerous: with nothing to mine, a model will
cheerfully generate plausible-looking questions, and three weeks later you cannot
remember which ones came from the course and which the machine made up. You will
revise invented material with the same confidence as real material.

**So the hard rule is separation, enforced in the UI:**

> **Prompt 15**
>
> We have no past exams. Every question you write is therefore **generated practice**,
> not sourced. Tag each item `origin: "generated"` or `origin: "sourced"` and render
> generated ones with a visible marker — a different badge colour and the label
> "generated practice, not from a past paper". I want to be able to filter to
> sourced-only at any time. Never let the two look alike.

**Find the real answer keys anyway.** They usually exist, just not as past papers:

> **Prompt 16**
>
> Search the course materials for anything with answers already attached: tutorial or
> problem sheets with model solutions, textbook chapter-end questions (often answered
> in an appendix or a published solutions manual), worked examples in the slides,
> self-test quizzes in the VLE, and any lecture recording where the lecturer works a
> problem through. Rank what you find by the same trust tiers. Build from these first
> and only generate questions for topics with no such source.

**Replace tag-frequency weighting with emphasis proxies:**

> **Prompt 17**
>
> Without past papers I have no frequency data, so infer emphasis instead and show me
> the evidence per topic:
> - explicit **learning objectives** — these are the exam blueprint in disguise, since
>   the examiner usually writes the paper from their own stated outcomes
> - slide count and lecture minutes per topic
> - how many **worked examples** a topic gets — heavy worked-example topics are almost
>   always calculation questions later
> - anything the lecturer flags verbally or in writing ("you should be able to…",
>   "this is examinable", recap and summary slides)
> - topics that recur across more than one lecture
>
> Give me a weighting table with the reasoning, and say which signals conflict.

**Then predict the paper before writing any questions:**

> **Prompt 18**
>
> Using the learning objectives and the emphasis table, write me a one-page **exam
> blueprint**: likely topics, likely question formats per topic (recall / apply /
> calculate / compare), and rough mark allocation. Show it to me for sanity-checking
> **before** you write a single question — then generate questions against the approved
> blueprint, matching the verb in each objective ("calculate" → a calculation,
> "compare" → a comparison, "describe" → recall).

Two more sources worth the search: **past papers from other universities** teaching the
same textbook (the questions transfer even when the course doesn't), and the
**lecturer's own publications** — people examine what they research, and a lecturer
whose papers you've read is far more predictable.

### Variant B — lab / project / coursework courses

Here the assessment isn't recall at all. You're marked on artifacts: code, reports,
demos, a viva. A flashcard site is close to useless. Build a different thing.

**The marking rubric is your tier-1 source.** It is the answer key — it literally
enumerates what earns marks. If you have one, everything else is secondary.

> **Prompt 19**
>
> This is a lab/coursework course. Build the site around the **assessment artifacts**,
> not recall. Modes I want:
>
> - **Spec → checklist.** Parse each assignment brief and the marking rubric into a
>   hard checklist of deliverables with mark values attached, so I can see what each
>   unchecked box costs me. Flag requirements stated only in passing in the brief —
>   those are the ones people lose marks on.
> - **Rubric self-check.** For each criterion, what distinguishes a top answer from a
>   middling one, in concrete terms, with an example of each where possible.
> - **Error catalogue.** Error message or symptom → cause → fix. Searchable.
> - **Environment runbook.** The exact setup that actually worked: versions, commands,
>   config, and the workarounds. Written so I can reproduce it on a clean machine.
> - **Viva prep.** Questions about *my own* submission: why this design, what's the
>   complexity, what would break at scale, what would you change.

> **Prompt 20 — the error catalogue is the highest-value part**
>
> As I work, append every error I hit to the catalogue: the exact message, what actually
> caused it, the fix that worked, and how long it cost me. Include the ones that turned
> out to be my own mistake — those recur most. This is the artifact I'll reuse next
> term and lend to other people; treat it as the primary deliverable, not a side note.

> **Prompt 21 — build it during, not after**
>
> Unlike an exam site, this gets built **incrementally as the course runs**. After each
> lab session, update the site: new checklist items, new catalogue entries, any design
> decision I made and why. Keep a dated decisions log — I will need it for the report
> and the viva, and I will not remember the reasoning by then.

**What carries over unchanged:** the single-file HTML contract, theme tokens, the
additive-round pattern, search (more valuable here — an error catalogue is useless if
you can't grep it at 2am), and Phase 7 testing.

**What to drop:** exam-frequency weighting, flashcards for definitions nobody will ask
you to recite, and past-paper mining.

**Hybrid courses** — exam plus coursework — are common. Build both, keep them in
separate modes, and weight your time by the actual mark split in the module handbook,
not by which one feels more urgent.

---

## Gotchas, in the order they will bite you

1. **Mojibake.** `·` and `—` render as `Â·` when a local file is served without a
   charset header. Convert every non-ASCII char to an HTML entity. But entities inside
   `.textContent` print literally — use `\uXXXX` there.
2. **`<span>` doesn't take `margin-bottom`.** Search results ran together on one line
   until the title and snippet got `display:block`.
3. **Invisible feedback.** A "you missed this correct answer" tick inherited
   `color:transparent` from the unselected state. Check every state's *computed* colour.
4. **Anchor drift.** Once you entity-encode, your edit anchors change (`&ndash;` becomes
   `&#8211;`). Re-grep for anchors before each edit round; assert they exist.
5. **Preview panes lie.** Files outside the project may render as static snapshots with
   no JS. Serve over `http://127.0.0.1:PORT` to test anything interactive.
6. **Bot walls.** Publisher sites may block automated fetches. Don't work around them —
   check an institutional repository instead (OpenAlex lists every open location for a
   DOI, and author deposits are usually there).
7. **Clean up.** Kill test servers, delete rendered scan images. They're large.

---

## Reusable one-shot kickoff

If you want to hand the whole thing over in one message:

> I have a course folder at `<PATH>` with lectures, past exams and study aids, some
> scanned. Build me a single self-contained HTML study site with Notes, Flashcards,
> Quiz and Exercises modes plus global search, following this playbook: `<paste this file>`.
>
> Work in phases and check in after each. Priorities: (1) never invent an answer key —
> rank sources by trustworthiness and label each answer with its source; (2) tell me
> explicitly what you could NOT determine rather than guessing; (3) test by actually
> running the page, not by reading the code.
>
> **First, check the shape of the course before starting Phase 0.** If there are no past
> exams, follow Variant A — everything you write is generated practice and must be
> visibly marked as such. If it's a lab/project course, follow Variant B — build around
> the marking rubric and the assessment artifacts, not recall. If it's both, build both
> and tell me the mark split from the module handbook.

---

*Written after building one of these for HCI. The phases are in this order because
doing them in a different order cost time — particularly organising before extracting,
and establishing answer-source trust before writing a single question.*
