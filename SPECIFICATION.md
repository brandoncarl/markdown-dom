# Markdown Document Object Model (MDOM)
## Technical Specification — v0.3.0

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Core Architecture](#2-core-architecture)
3. [Data Structures](#3-data-structures)
4. [TaskItem](#4-taskitem)
5. [Selector Language](#5-selector-language)
6. [API Surface](#6-api-surface)
7. [Edge Case Resolution](#7-edge-case-resolution)
8. [Serialization & Lossless Round-Trip](#8-serialization--lossless-round-trip)
9. [Examples](#9-examples)

---

## 1. Introduction

The **Markdown Document Object Model (MDOM)** is a library-level abstraction that parses Markdown source into a hierarchical, mutable tree. Unlike flat AST representations (e.g., CommonMark's node list), MDOM treats headers as *implicit structural containers* — a heading does not merely introduce content; it *owns* it.

This design makes MDOM suitable for programmatic document manipulation: renaming sections, moving blocks, injecting content at precise locations, and re-serializing with guaranteed source fidelity.

### Design Goals

- **Structural clarity** — Headers create Sections; Sections own their children.
- **Source fidelity** — Style metadata and source ranges are preserved so that unmodified nodes serialize without loss.
- **Surgical editing** — Mutations affect only the targeted node and its immediate style boundaries.
- **Composable queries** — A Markdown-native selector syntax allows expressive, readable traversal.
- **Format portability** — Header syntax (hash or dot notation) is configurable at the Document level.

### Setext Headers

MDOM accepts Setext-style headers (`===` and `---` underlines) at parse time but does not preserve them as a heading style. Setext headers are normalized to their hash equivalents during parsing — `===` becomes `#`, `---` becomes `##` — and `doc.headingStyle` is set to `"setext"` to record that the source used this convention. On `render()`, if `headingStyle` remains `"setext"`, the output uses Setext notation; if changed to `"hash"`, hash notation is used instead. The lossless round-trip guarantee does not apply to Setext sources, since the underline length and exact whitespace of the original source are not preserved.

The primary motivation for accepting rather than rejecting Setext is pragmatic: a significant body of real-world Markdown — older GitHub READMEs, Jekyll content, Pandoc output — uses Setext headers, and throwing a parse error would create unnecessary friction for consumers of existing documents.

---

## 2. Core Architecture

### 2.1 Node Taxonomy

MDOM defines four node kinds. Understanding the distinction between Section and Block is fundamental to working with the model.

| Term | Role | Examples |
| :--- | :--- | :--- |
| **Document** | The root node. Contains top-level Sections and any leading Blocks. | — |
| **Section** | A *structural scope* opened by a header. Has no content of its own — only a header that names it and children that belong to it. Owns sub-Sections and Blocks until closed by a peer or ancestor heading. | `# Intro`, `## Setup` |
| **Block** | An *atomic content unit* that renders as a discrete element. Holds data, not structure. Never opens a new structural scope. A Block may contain other Blocks (e.g. list items) but never Sections. | `Paragraph`, `CodeBlock`, `List`, `BlockQuote`, `ThematicBreak`, `Table` |
| **Inline** | A text-level element inside a Block or a Section header. | `Strong`, `Emphasis`, `Link`, `Code`, `Image` |

The key distinction: **Sections create hierarchy; Blocks create content.** Nesting works in one direction — Sections can contain Blocks and other Sections, but Blocks cannot contain Sections. A heading that appears inside a code fence has no structural meaning in MDOM; it is raw text.

### 2.2 The Section Model

Parsing is a single, linear pass over the token stream. When a Header token of level *N* is encountered, the parser:

1. Closes any open Section whose level is ≥ *N*.
2. Opens a new `Section` node with level *N*, associating the header text as its `header` property.
3. Appends all subsequent Block tokens as children of this Section, until the Section is closed by rule (1).

A Section of level *N* is closed when a Header of level ≤ *N* is encountered, or when the end-of-file is reached.

```
Document
├── Section[H1] "Getting Started"
│   ├── Paragraph "Welcome to..."
│   ├── Section[H2] "Installation"
│   │   ├── Paragraph "Run the following..."
│   │   └── CodeBlock (lang="bash")
│   └── Section[H2] "Configuration"
│       └── List
└── Section[H1] "API Reference"
    └── Section[H2] "Methods"
        └── Paragraph "..."
```

### 2.3 Document-Level Header Format

MDOM supports two header serialization formats, configured at the Document level. The format applies globally to all Section headers when rendering and is enforced when headers are created or mutated via the API.

| Format | Value | Example H2 | Description |
| :--- | :--- | :--- | :--- |
| **Hash** | `"hash"` | `## Installation` | ATX Markdown — hashes prefix the header text. The default. |
| **Dot** | `"dot"` | `h2. Installation` | Level indicator followed by a dot — used by JIRA, Confluence, and similar wiki platforms. |

The default format is `"hash"`. The format is set on the `Document` node at parse time and may be changed at any point before `render()` is called.

```typescript
const doc = parse(src, { headerFormat: "dot" });
doc.headerFormat = "hash"; // revert to default
```

When parsing, the format is inferred from the first header encountered. If headers of mixed formats are found, a `FormatMixError` is thrown. A utility function `normalizeHeaders(src, targetFormat)` is provided to convert source before parsing.

---

## 3. Data Structures

### 3.1 BaseNode

All nodes extend `BaseNode`.

```typescript
interface BaseNode {
  /** Discriminated union tag. */
  type: "Document" | "Section" | "Block" | "Inline";

  /**
   * The raw source text this node was parsed from, including its own style
   * metadata but excluding the style of child nodes (which they own themselves).
   */
  raw: string;

  /** Byte offsets into the original source string: [start, end). */
  range: [number, number];

  /** Reference to the containing Section or Document. Null on the root. */
  parent: Section | Document | null;

  /** Child nodes. Empty for leaf nodes. */
  children: Array<Section | Block>;

  /** Source-preservation and presentational metadata. */
  style: Style;
}
```

### 3.2 Style

Every node carries a `style` object capturing presentational choices made in the source — character-level decisions that do not affect semantics but must be preserved for lossless round-tripping. Fields that do not apply to a given node type are `null`.

```typescript
interface Style {
  /**
   * Raw blank lines immediately BEFORE this node's first token (e.g. "\n\n").
   *
   * Ownership is leading rather than trailing by design: each node is
   * responsible for the whitespace that precedes it, not the whitespace
   * that follows it. This makes mutations clean and unambiguous — when a
   * node is removed, its spacingBefore is removed with it, and no orphaned
   * trailing space is left on the previous sibling. When a node is inserted,
   * its spacingBefore travels with it regardless of where it lands. There is
   * never a conflict over who owns the gap between two adjacent nodes.
   */
  spacingBefore: string;

  /** For List/ListItem nodes: the bullet character used ("-", "*", "+")
   *  or number style ("1.", "1)"). Null for all other node types. */
  bullet: string | null;

  /** For CodeBlock nodes: the fence string as written (e.g. "```", "~~~", "````").
   *  Null for all other node types. */
  fence: string | null;

  /** For Section nodes: the header prefix as written (e.g. "##", "h2.").
   *  Null for all other node types. */
  heading: string | null;

  /** Leading whitespace characters on this node's opening token.
   *  Significant for nested list items; empty string for non-indented nodes. */
  indent: string;
}
```

### 3.3 Document

```typescript
interface Document extends BaseNode {
  type: "Document";

  /**
   * Header serialization format applied globally on render().
   * Defaults to "hash". May be set freely before render().
   */
  headerFormat: "hash" | "dot";

  /**
   * The heading style detected in the source.
   * "hash" for ATX sources; "setext" when the source used underline notation.
   * Controls whether render() outputs hash or Setext notation when no
   * explicit conversion has been requested.
   */
  headingStyle: "hash" | "setext";

  /**
   * Raw trailing content after the last node (typically a final newline).
   * Preserved for lossless round-tripping.
   */
  trailingNewline: string;

  children: Array<Section | Block>;
  parent: null;
}
```

### 3.4 Section

```typescript
interface Section extends BaseNode {
  type: "Section";

  /** Header level: 1–6. */
  level: 1 | 2 | 3 | 4 | 5 | 6;

  /**
   * Parsed inline content of the header line.
   * Stored as an array of Inline nodes to support bold/link headings.
   */
  header: Inline[];

  /**
   * Plain-text rendering of `header`, used for selector matching.
   * Computed once at parse time. Invalidated when `header` is mutated.
   */
  headerText: string;

  children: Array<Section | Block>;
  parent: Section | Document;
}
```

### 3.5 Block

```typescript
interface Block extends BaseNode {
  type: "Block";

  /** Refined block type. */
  blockType:
    | "Paragraph"
    | "CodeBlock"
    | "List"
    | "ListItem"
    | "TaskItem"       // see §4
    | "BlockQuote"
    | "Table"
    | "ThematicBreak"
    | "HTMLBlock"
    | "LinkDefinition";

  /** Language tag for CodeBlocks; null for all others. */
  lang: string | null;

  /** Inline children for Paragraph, ListItem, TableCell, etc. */
  children: Array<Block | Inline>;
  parent: Section | Document | Block;
}
```

### 3.6 Inline

```typescript
interface Inline extends BaseNode {
  type: "Inline";
  inlineType:
    | "Text"
    | "Strong"
    | "Emphasis"
    | "Code"
    | "Link"
    | "Image"
    | "HardBreak"
    | "SoftBreak"
    | "RawHTML";

  /** href for Link/Image; null otherwise. */
  href: string | null;

  /** alt text for Image; null otherwise. */
  alt: string | null;

  children: Inline[];
  parent: Block | Inline;
}
```

---

## 4. TaskItem

`TaskItem` is a specialization of `ListItem` that carries a `status` field representing the completion state of the item. It is recognized during parsing when a list item's content begins with a bracket-enclosed status marker.

### 4.1 Recognized Syntax

MDOM recognizes the standard GitHub Flavored Markdown checkbox syntax as well as arbitrary single-character custom status markers:

| Source syntax | Parsed status |
| :--- | :--- |
| `- [ ] Buy milk` | `""` (open/unchecked) |
| `- [x] Buy milk` | `"x"` (complete) |
| `- [X] Buy milk` | `"X"` (complete, uppercase variant) |
| `- [~] Buy milk` | `"~"` (custom — e.g. in-progress) |
| `- [-] Buy milk` | `"-"` (custom — e.g. cancelled) |
| `- [?] Buy milk` | `"?"` (custom — e.g. blocked) |

The status marker is always a single character (or a space for the open state) enclosed in square brackets, immediately following the bullet and any indentation whitespace. Any character is accepted; MDOM imposes no closed vocabulary beyond the structural requirement. Interpretation of custom statuses is the responsibility of the consuming application.

### 4.2 Data Structure

`TaskItem` extends `Block` with an additional `status` field. All other `ListItem` properties apply.

```typescript
interface TaskItem extends Block {
  blockType: "TaskItem";

  /**
   * The raw status character from inside the brackets.
   * Empty string "" represents the open/unchecked state (source: "[ ]").
   * Any single character is valid. Common values: "x", "X", "~", "-", "?".
   */
  status: string;

  /**
   * The raw bracket expression as written in the source (e.g. "[ ]", "[x]", "[~]").
   * Preserved for lossless round-tripping.
   */
  rawMarker: string;

  children: Array<Block | Inline>;
  parent: Block; // parent must be a List
}
```

### 4.3 API Behavior

`TaskItem` nodes participate in all standard traversal and mutation APIs. In addition, the `status` field may be set directly on the node:

```typescript
const task = doc.select("## [Todo] > list > task-item:1");
task.status = "x"; // marks the first task item complete
```

Setting `status` updates `rawMarker` accordingly and marks the node dirty for re-serialization. The bullet style and body content are unaffected.

**Selector support:** `task-item` is a valid block selector and may be combined with attribute filters:

```typescript
doc.selectAll(`task-item[status=""]`);   // all unchecked items
doc.selectAll(`task-item[status="x"]`);  // all complete items
```

### 4.4 Mixed Lists

A `List` Block may contain a mix of `ListItem` and `TaskItem` children. This is valid MDOM and reflects legal Markdown. The `blockType` of each child distinguishes them. A `List` that contains at least one `TaskItem` is considered a *task list*; this is a derived property and is not stored on the `List` node itself.

---

## 5. Selector Language

MDOM uses a custom, Markdown-native selector syntax. Positional indices are **1-based** throughout, consistent with how Table of Contents entries and document sections are numbered by humans and by LLMs generating selectors in natural language contexts.

### 5.1 Section Selectors

| Syntax | Description |
| :--- | :--- |
| `##` | All H2 Sections (any header text). |
| `## [Installation]` | The H2 Section whose `headerText` equals `"Installation"` (case-sensitive). The space between `##` and `[` is optional. |
| `##:2` | The **second** H2 in the document (1-based). |
| `## [Notes]:2` | The second H2 whose `headerText` equals `"Notes"` (1-based). |

**Matching rules for `[text]`:**

- Matching is against the node's `headerText` (plain-text rendering of the header Inlines).
- Matching is exact and case-sensitive by default.
- Whitespace inside `[...]` is significant: `[My API]` will not match `[my api]`.
- Future versions may support `[/regex/i]` for case-insensitive pattern matching.

### 5.2 Block Selectors

| Syntax | Description |
| :--- | :--- |
| `p` | All Paragraph blocks. |
| `code` | All CodeBlock blocks. |
| `list` | All List blocks. |
| `list-item` | All ListItem blocks (excludes TaskItems). |
| `task-item` | All TaskItem blocks. |
| `blockquote` | All BlockQuote blocks. |
| `table` | All Table blocks. |
| `hr` | All ThematicBreak blocks. |

### 5.3 Attribute Filters

Attribute filters are appended to any selector using `[attr="value"]` syntax.

| Syntax | Description |
| :--- | :--- |
| `code[lang="js"]` | CodeBlocks whose `lang` property equals `"js"`. |
| `code[lang]` | CodeBlocks that have any non-null `lang`. |
| `task-item[status="x"]` | TaskItems with status `"x"`. |
| `task-item[status=""]` | TaskItems that are unchecked (open). |

Multiple filters may be chained: `code[lang="ts"][lang!="tsx"]`. Supported operators: `=` (equals), `!=` (not equals), `^=` (starts with), `$=` (ends with), `*=` (contains).

### 5.4 Combinators

| Syntax | Description |
| :--- | :--- |
| `## [API] > code` | **Child combinator.** Selects CodeBlocks that are *direct* children of the "API" H2 Section. Sub-sections are not traversed. |
| `## [Setup] p` | **Descendant combinator.** Selects all Paragraphs anywhere inside the "Setup" H2 Section (including those inside sub-sections). |
| `h2 + p` | **Adjacent sibling combinator.** Selects a Paragraph that is the immediately following sibling of an H2 Section, within the same parent. |

**Combinator precedence** (highest to lowest): attribute filter `[]` > child `>` > adjacent sibling `+` > descendant ` ` (space).

### 5.5 Selector Parsing Rules

1. Tokenize left-to-right. Tokens are: `HashSequence`, `BracketText`, `ColonIndex`, `ElementType`, `AttributeFilter`, `Combinator`.
2. A `HashSequence` (one or more `#` characters) identifies a Section selector; its length is the heading level.
3. `BracketText` immediately following a `HashSequence` (with optional whitespace) is a semantic ID filter.
4. `ColonIndex` immediately following a `HashSequence` or `BracketText` is a 1-based positional filter.
5. Combinators are inferred from context: `>` is explicit; a single space between two non-combinator tokens implies descendant; `+` is explicit.
6. An unrecognized token raises a `SelectorSyntaxError`.

---

## 6. API Surface

All API methods are available on `NodeHandle` — a wrapper returned by `select`, `selectAll`, or the root `document` object. `NodeHandle` is lazy: it holds a reference to the live node and re-evaluates queries on each call.

### 6.1 `select(selector): NodeHandle | null`

Returns a `NodeHandle` wrapping the **first** node matching `selector` in document order, or `null` if no match is found.

### 6.2 `selectAll(selector): NodeHandle[]`

Returns an array of `NodeHandle` objects for **all** matching nodes in document order.

### 6.3 `replace(md: string, header?: string): void`

Updates the content of the selected node. If only `md` is provided, the body is replaced and the header is unchanged. If `header` is also provided, the Section's header text is updated while its level and `style.heading` prefix are preserved. The operation is atomic — either the full parse succeeds or no mutation occurs and a `ParseError` is thrown.

```typescript
doc.select("## [Old Name]").replace("", "New Name");                 // header only
doc.select("## [Installation]").replace("Run `npm install`.\n");     // body only
doc.select("## [Draft]").replace("Final content.", "Final Version"); // both
```

### 6.4 `before(md: string, header?: string): NodeHandle`

Parses `md` (and optionally `header`) and inserts the result as a sibling **immediately before** the current node. Returns a `NodeHandle` for the first inserted node.

### 6.5 `after(md: string, header?: string): NodeHandle`

Identical to `before`, but inserts **immediately after** the current node.

### 6.6 `append(md: string, header?: string): NodeHandle`

Appends parsed content as the **last child** of the current Section. If `header` is provided, a new sub-Section is created at level + 1. Raises `InvalidOperationError` on Block nodes.

### 6.7 `prepend(md: string, header?: string): NodeHandle`

Identical to `append`, but inserts as the **first child**.

### 6.8 `remove(): void`

Removes the node and all of its descendants from the tree. The removed node's `parent` is set to `null`. Because each node owns its own `style.spacingBefore`, the preceding sibling is unaffected — no orphaned whitespace is left behind.

### 6.9 `move(delta: number): void`

Moves the node `delta` positions within its parent's `children` array. Positive values move the node later (downward); negative values move it earlier (upward). `style.spacingBefore` travels with the node so that inter-block spacing is preserved in its original relative position.

If `delta` would move the node past the first or last sibling, the node is placed at the boundary — the operation does not wrap and does not throw. A `delta` of `0` is a no-op.

```typescript
doc.select("## [Step 3]").move(-2); // move up two positions
doc.select("## [Intro] > p:1").move(1); // move the first paragraph down one
```

### 6.10 `children(): NodeHandle[]`

Returns `NodeHandle` wrappers for the immediate child nodes of the current node, in source order.

### 6.11 `parent(): NodeHandle | null`

Returns a `NodeHandle` for the containing Section or Document root. Returns `null` on the Document root.

### 6.12 `toc(): TocEntry[]`

Returns a nested Table of Contents structure derived from Section nodes only.

```typescript
interface TocEntry {
  level: number;
  headerText: string;
  children: TocEntry[];
}
```

### 6.13 `render(): string`

Serializes the Document tree (or the subtree rooted at the current node) back to a Markdown string. See §8 for the lossless round-trip guarantee.

---

## 7. Edge Case Resolution

### 7.1 Orphaned Content (Pre-Header Blocks)

Blocks that appear before any Header are attached as direct children of the `Document` root, which acts as an implicit level-0 container.

### 7.2 Skipped Heading Levels

A document may jump from `#` to `###` without an intermediate `##`. MDOM does not re-level or synthesize missing sections. The `###` Section attaches as a child of the nearest open Section whose level is less than 3. No error is raised; the structure is legal MDOM.

### 7.3 Malformed or Empty Headers

An ATX header line with no text (`##` alone, or `## `) produces a Section with `headerText: ""`. Selectors `## []` or `##` will match it; `## [anything]` will not.

### 7.4 Setext Headers

Setext headers are accepted and normalized to their hash equivalents during parsing. `doc.headingStyle` is set to `"setext"`. The lossless round-trip guarantee does not apply to Setext sources — underline length and original whitespace are not preserved.

### 7.5 Mixed-Format Header Input

If a source document contains both hash-style and dot-style headers, a `FormatMixError` is thrown at parse time. Use `normalizeHeaders(src, targetFormat)` to convert the source before parsing. Since the two formats are syntactically unambiguous — `##` cannot be mistaken for `h2.` — detection is reliable.

### 7.6 Deeply Nested Lists

List nesting is resolved at the Block level. A `List` Block may contain `ListItem` or `TaskItem` Blocks which may themselves contain further `List` Blocks. This does not interact with the Section hierarchy. `style.indent` is preserved per item.

### 7.7 move() at Boundaries

If `delta` would carry the node past the first or last position in the parent's `children` array, the node is clamped to the boundary. No error is thrown. If the node is the only child, `move()` is always a no-op regardless of `delta`.

### 7.8 Conflicting Selector Indices

If `##:5` is requested but fewer than 5 H2 Sections exist, `select` returns `null`; `selectAll` returns `[]`. No error is thrown.

### 7.9 Concurrent Mutation

MDOM trees are not thread-safe. Callers must serialize mutations or clone the tree before concurrent use.

---

## 8. Serialization & Lossless Round-Trip

### 8.1 The Round-Trip Guarantee

> **Guarantee:** For any valid Markdown input `src` using consistent hash-format headers, `parse(src).render()` produces a string that is losslessly equivalent to `src` — all content, structure, and style decisions present in the source are recoverable from the tree.

This holds because every byte of `src` is owned by exactly one node's `raw` field or one `Style` field — there is no data loss during parsing. `render()` concatenates these in source order with no normalization applied to unmodified nodes. The guarantee is scoped to the subtree of mutated nodes; all unmodified nodes serialize identically to their source.

"Lossless" is used deliberately rather than "byte-identical." The tree fully preserves the information needed to reproduce the source, but implementations are not required to guarantee identical byte output in the presence of system-level differences such as line ending normalization.

### 8.2 Serialization Algorithm

`render()` performs a pre-order depth-first traversal. Each node emits its own `style.spacingBefore` ahead of its content, so spacing ownership is always unambiguous:

```
function renderNode(node):
  output = node.style.spacingBefore
  output += node.style.indent
  output += node.raw                    // header line only, for Sections
  for child in node.children:
    output += renderNode(child)
  return output
```

The Document's `trailingNewline` is appended after the last child.

### 8.3 Mutation Serialization

When a node is replaced or created via API: the new sub-tree inherits the original node's `style.spacingBefore` to preserve inter-block spacing; the heading prefix is re-derived from `Document.headerFormat` at the current level; internal style is set from the parsed source of `md`.

### 8.4 Round-Trip Scope

| Scenario | Status |
| :--- | :--- |
| `parse(src).render()` — hash format, no mutations | ✅ Lossless |
| Mutate node A → `render()` | ✅ All nodes except A's subtree are lossless |
| `remove(nodeA)` → `render()` | ✅ nodeA's content absent; all others lossless |
| `move(delta)` → `render()` | ✅ Moved node and its spacingBefore travel together; all others lossless |
| Source uses Setext headers | ⚠️ Accepted; normalized to hash on parse; underline style not preserved |
| Source contains mixed hash and dot headers | ❌ `FormatMixError` thrown |

---

## 9. Examples

### 9.1 Rename a Section While Preserving Its Content

```javascript
import { parse } from "mdom";

const doc = parse(`
## Old Title

This is the section body.

- Item one
- Item two
`);

doc.select("## [Old Title]").replace("", "New Title");

console.log(doc.render());
// ## New Title
//
// This is the section body.
//
// - Item one
// - Item two
```

### 9.2 Move a Code Block Between Sections

```javascript
const doc = parse(`
## Source Section

Some intro text.

\`\`\`js
console.log("hello");
\`\`\`

## Destination Section

Content here.
`);

const codeBlock = doc.select(`## [Source Section] > code[lang="js"]`);
const codeRaw = codeBlock.render();
codeBlock.remove();
doc.select("## [Destination Section]").append(codeRaw);
```

### 9.3 Reorder Sections with move()

```javascript
const doc = parse(`
# Guide

## Step 1

First.

## Step 2

Second.

## Step 3

Third.
`);

// Move Step 3 up two positions so it becomes the first sub-section
doc.select("## [Step 3]").move(-2);

// Section order is now: Step 3, Step 1, Step 2
console.log(doc.toc()[0].children.map(e => e.headerText));
// ["Step 3", "Step 1", "Step 2"]
```

### 9.4 Working with TaskItems

```javascript
const doc = parse(`
## Sprint Backlog

- [x] Design the schema
- [ ] Write the parser
- [~] Draft the spec
- [ ] Add test coverage
`);

// Find all unchecked items (1-based index used in selector)
const open = doc.selectAll(`## [Sprint Backlog] > list > task-item[status=""]`);
console.log(open.length); // 2

// Mark the first open item as in-progress
open[0].status = "~";
```

### 9.5 Render with a Different Header Format

```javascript
const doc = parse(`# Title\n\n## Section\n\nContent.\n`);
doc.headerFormat = "dot";
console.log(doc.render());
// h1. Title
//
// h2. Section
//
// Content.
```

### 9.6 Query the Table of Contents

```javascript
function printToc(entries, depth = 0) {
  for (const entry of entries) {
    console.log("  ".repeat(depth) + `${"#".repeat(entry.level)} ${entry.headerText}`);
    printToc(entry.children, depth + 1);
  }
}

printToc(doc.toc());
```

### 9.7 Convert a Setext Document to Hash Notation

```javascript
const doc = parse(`
Title
=====

Introduction
------------

Some content.
`);

console.log(doc.headingStyle); // "setext"
doc.headingStyle = "hash";

console.log(doc.render());
// # Title
//
// ## Introduction
//
// Some content.
```

---

*MDOM Specification v0.3.0 — End of Document*
