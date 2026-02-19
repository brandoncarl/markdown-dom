# MDOM — Markdown Document Object Model

> **Treat Markdown like the DOM. Query it. Mutate it. Serialize it back — without losing a byte.**

---

## Install

```bash
npm install mdom   # placeholder — library coming soon
```

→ [Read the full technical specification](./MDOM_Specification.md)  
→ [Library repository](#) ← *link coming soon*

---

## The Problem: Markdown Is the Lingua Franca of LLMs, and We're Editing It with Blunt Instruments

Markdown has quietly become the default language of AI systems. It is how LLMs receive long-form context, how they return structured output, how documentation is written, stored, and passed between tools. When an agent reads a README, summarizes a spec, or updates a knowledge base — it is almost certainly working in Markdown.

But the way we actually *edit* Markdown programmatically is stuck in the past. There are three patterns in common use today, and all three break down on structured documents.

**Pattern 1: Read everything, write everything.** The agent receives the full document in context, generates a complete replacement, and the caller diffs the result. This is the default in most agentic frameworks because it requires no document understanding — just ask the model to return everything with the change applied. It works for small files. At scale it is wasteful by construction: every token in the document is read and re-emitted regardless of whether it was touched.

**Pattern 2: String-level search and replace.** Tools like Claude Code (`str_replace`), Aider (unified diff blocks), and Cursor ask the model to emit a before/after pair — a unique string to find and a string to replace it with. This is genuinely surgical at the phrase or line level, and it works well for code. For structured documents it breaks down quickly:

- The target string must appear exactly once. Duplicate phrases cause errors or silent mis-targeting.
- There is no concept of scope. You can replace the text "See installation guide" but you cannot say "replace the second paragraph *inside* the Installation section."
- Whitespace and formatting changes break the match. A blank line inserted above the target string is enough to cause a failure.
- Moving or reordering sections requires the agent to manually compute start and end boundaries by finding heading lines and the next sibling heading — heuristic, fragile, and re-derived from scratch every time.

**Pattern 3: Regex and line-number anchors.** Scripts and agents that don't use an LLM for the edit itself often locate target content by regex or line number. These break silently the moment a section is renamed, reordered, or reflowed. There is no indication that the anchor has drifted; the edit just lands in the wrong place.

None of these approaches have a *document model*. They treat Markdown as a flat string with syntactic hints. When an agent needs to move the Authentication section above the Configuration section, it must find the heading, find where the section ends by locating the next peer heading, slice and rejoin the string, and hope the surrounding whitespace survives. Every step is heuristic. Agents doing this in practice make boundary errors regularly — clipping the wrong content, losing blank lines between sections, or silently editing the wrong section because a heading string was not unique.

As documentation grows, as AI agents take on longer editing tasks, and as the same document gets updated by multiple passes of automated tooling, the costs compound:

- **Token waste.** An agent updating one paragraph in a 10,000-word spec pays full context cost for content it never touches.
- **Positional drift.** String anchors and line numbers go stale silently. The agent produces edits to the wrong location with no error signal.
- **No structural awareness.** There is no stable representation of "the Installation section and everything it owns." Structure is re-inferred from raw text on every operation.
- **Formatting destruction.** A round-trip through a Markdown parser and serializer normalizes bullet styles, collapses blank lines, and changes fence characters. Diffs become noisy. Version control history becomes unreliable.

**MDOM is the fix.** It parses Markdown into a hierarchical, mutable tree where headers are structural containers — just like `<div>` and `<section>` in HTML — and provides a DOM-like API for querying, editing, and re-serializing documents with surgical precision and lossless fidelity.

---

## How It Works

MDOM treats a heading not as a line of text but as an opening of a **Section** — a structural scope that owns everything beneath it until a peer or ancestor heading closes it.

```
# Getting Started          ← Section[H1]
  ## Installation          ← Section[H2], child of H1
    (paragraph)
    (code block)
  ## Configuration         ← Section[H2], child of H1
    (list)
# API Reference            ← Section[H1]
```

This tree is queryable with a Markdown-native selector syntax, mutable through a clean API, and serializes back to source with no reformatting of content you didn't touch.

```javascript
import { parse } from "mdom";

const doc = parse(source);

// Rename a section
doc.select("## [Installation]").setHeader("Quick Start");

// Replace the content of one section without touching anything else
doc.select("## [Quick Start]").setContent("Run `npm install mdom`.\n");

// Move a code block from one section to another
const snippet = doc.select(`## [Examples] > code[lang="js"]`);
const raw = snippet.render();
snippet.remove();
doc.select("## [Quick Start]").append(raw);

// Serialize — only changed nodes are touched; everything else is byte-for-byte identical
console.log(doc.render());
```

---

## Why This Makes LLMs Better at Their Jobs

The following three experiments are proposed hypotheses. Results pending — **if you run them, we want to know what you find.**

---

### Experiment 1: Surgical Section Updates — Token Cost Reduction

**Hypothesis:** An agent tasked with updating a single section in a large document spends the majority of its output tokens re-emitting content it was never asked to change. MDOM eliminates that waste.

**Setup.** Take a 5,000-word technical specification with 12 top-level sections. Ask an LLM to update the content of one section — say, "rewrite the Authentication section to reflect the new OAuth2 flow." Measure output tokens in two conditions:

- **Baseline:** The agent is given the full document as context and asked to return the full updated document.
- **MDOM:** The agent is given only the rendered output of `doc.select("## [Authentication]").render()` — roughly 400 words — and asked to return replacement content. The result is passed to `doc.select("## [Authentication]").setContent(result)`.

**Expected outcome:** The MDOM condition reduces output tokens by approximately 90% for this task. The input context is also dramatically smaller, meaning the agent can focus entirely on the relevant section rather than attending over thousands of tokens of unrelated content. For agents running in a loop across many sections, this compounds significantly.

**Why it matters at scale.** A documentation pipeline that processes 50 section updates per document run could cut token costs by an order of magnitude. For teams running automated doc maintenance on large codebases, this is the difference between a practical tool and an expensive one.

---

### Experiment 2: Selector-Based Targeting — Accuracy on Structural Tasks

**Hypothesis:** Asking an LLM to generate an MDOM selector to identify a target node is more accurate and more robust than asking it to generate a regex or line number to locate the same content.

**Setup.** Create a test set of 100 Markdown documents ranging from 500 to 3,000 words, each with 5–15 sections and varied heading levels. For each document, specify a target node in plain English (e.g., "the JavaScript code block inside the Examples section," "the second paragraph under Configuration"). Ask an LLM to produce a selector or locator for that node in three conditions:

- **Regex:** The agent writes a regular expression to match the target content.
- **Line number:** The agent identifies the line number of the target node.
- **MDOM selector:** The agent writes an MDOM selector (e.g., `## [Examples] > code[lang="js"]`, `## [Configuration] > p:2`).

Evaluate each by running the locator against the document and checking whether it resolves to the correct node. Then mutate the document (rename a section, add a paragraph above the target) and re-evaluate without asking the agent to update its locator.

**Expected outcome:** MDOM selectors degrade gracefully under document mutation — `## [Examples] > code[lang="js"]` still resolves correctly after a paragraph is inserted above it. Regex and line-number approaches break silently. Initial accuracy should also favor MDOM selectors because the syntax maps directly to how LLMs describe document structure in natural language ("the code block inside the Examples section").

**Why it matters.** Agents that plan multi-step document edits need stable references to nodes across multiple turns. A selector that survives document mutations without being recalculated is foundational to reliable multi-step agentic editing.

---

### Experiment 3: TOC-Guided Context Loading — Accuracy on Long Documents

**Hypothesis:** Providing an LLM with a structured Table of Contents as a navigation layer — rather than the full document — allows it to selectively load only the sections relevant to a task, dramatically improving accuracy on questions that require finding information across a large document.

**Setup.** Take a long technical document (15,000+ words, 30+ sections). Ask an LLM to answer 50 factual questions that each require finding information in one specific section. Test in three conditions:

- **Full document:** The agent receives the entire document in context.
- **TOC + fetch:** The agent receives `doc.toc()` output (a compact nested structure of section titles), identifies the relevant section by title, then fetches only that section via `doc.select("## [Section Name]").render()`.
- **No context:** Baseline — agent answers from training knowledge alone.

Measure answer accuracy and input tokens consumed.

**Expected outcome:** The TOC + fetch condition matches or exceeds full-document accuracy while consuming a fraction of the input tokens — because the agent can orient itself using the compact TOC and then load only what it needs. On very long documents where the full content exceeds the practical attention window, the TOC + fetch condition should significantly *outperform* the full-document condition because the relevant section is not diluted by thousands of tokens of unrelated content.

**Why it matters.** This pattern — `toc()` as a navigation index, `select().render()` as a targeted fetch — is a primitive for building memory-efficient document agents. It mirrors how humans read reference documentation: scan the table of contents, jump to the relevant section, read only that.

---

## Design Principles

MDOM is built around a few explicit choices that distinguish it from general-purpose Markdown parsers:

**Agent-first API.** Method names are chosen to be unambiguous in natural language. `setHeader()` changes the title. `setContent()` changes the body. `select("## [Installation]")` selects the Installation section. An LLM reading API documentation should be able to generate correct calls without memorizing argument conventions.

**1-based selector indices.** Selectors use 1-based positional indices (`##:1` = first H2) because LLMs and humans count from one in natural language contexts. This is a deliberate deviation from JavaScript array convention, documented and intentional.

**Lossless serialization.** Unmodified nodes serialize byte-for-byte identically to their source. Bullet styles, fence characters, blank lines, and heading markers are preserved exactly. You can run MDOM over a document and re-serialize it without creating noisy diffs.

**Strict by default.** Mixed header formats throw errors rather than silently normalizing. Pass `{ strict: false }` to opt into lenient mode.

**`HeadingBlock` for headings inside containers.** A heading inside a blockquote or list is parsed as a presentational `HeadingBlock`, not a structural `Section`. It does not appear in `toc()` and does not own subsequent content. This keeps the Section hierarchy clean and prevents container-level headings from polluting document structure.

---

## Specification

The full technical specification — including data structures, selector grammar, API surface, edge case resolution, and serialization guarantees — is available here:

→ **[MDOM Technical Specification v0.4.0](./MDOM_Specification.md)**

---

## Status

MDOM is currently in the **Request for Comment** phase. The specification is stable enough to implement against, but the API surface may change based on feedback before the 1.0 release.

**Library:** [link coming soon](#)  
**Spec version:** v0.4.0  
**Feedback:** Open an issue or submit a pull request against the specification document.

---

*Built for a world where Markdown is infrastructure, not just formatting.*
