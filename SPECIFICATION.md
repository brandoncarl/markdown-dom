# Markdown Document Object Model (MDOM)
## Technical Specification — v0.6.0

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
10. [Agent Tool Interface](#10-agent-tool-interface)

---

## 1. Introduction

The **Markdown Document Object Model (MDOM)** is a library-level abstraction that parses Markdown source into a hierarchical, mutable tree. Unlike flat AST representations (e.g., CommonMark's node list), MDOM treats headers as *implicit structural containers* — a heading does not merely introduce content; it *owns* it.

This design makes MDOM suitable for programmatic document manipulation: renaming sections, moving blocks, injecting content at precise locations, and re-serializing with guaranteed source fidelity.

### 1.1 Design Goals

- **Structural clarity** — Headers create Sections; Sections own their children.
- **Source fidelity** — Style metadata and source ranges are preserved so that unmodified nodes serialize without loss.
- **Surgical editing** — Mutations affect only the targeted node and its immediate style boundaries.
- **Composable queries** — A Markdown-native selector syntax allows expressive, readable traversal.
- **Format portability** — Header syntax (hash or dot notation) is configurable at the Document level.
- **Agent-first design** — The API, selector language, and method names are optimized for use by LLMs and coding agents, not just human developers. Names are unambiguous in natural language, methods do one clearly-named thing, and conventions are chosen to minimize off-by-one errors and hallucination in generated code.

### 1.2 Intentional Deviations from Convention

MDOM makes several deliberate choices that diverge from Markdown standards or JavaScript conventions. These are not oversights — each is a reasoned trade-off explained here so that reviewers and implementors can evaluate them on their merits.

**1-based positional indexing in selectors.** MDOM selectors use 1-based indices (`##:1` means the first H2, `p:2` means the second paragraph). This deviates from JavaScript's 0-based array convention but aligns with how humans, Table of Contents entries, and LLMs naturally count ordered items in prose. The primary consumer of MDOM selectors is expected to be agent-generated code operating on natural language intent, where "the first section" should map to `:1`, not `:0`. The resulting array from `selectAll()` remains 0-based, as is standard in JavaScript — this distinction is intentional and documented.

**Section ownership model (deviation from flat AST).** Standard Markdown parsers produce flat sibling lists of blocks and headings. MDOM instead treats a heading as opening a *structural scope* that owns all subsequent content until a peer or ancestor heading closes it. This is a deliberate abstraction layer, not a Markdown standard. It enables DOM-like traversal and mutation but means MDOM trees cannot be trivially round-tripped through other Markdown parsers without going through `render()`.

**Headings inside container blocks are `HeadingBlock`, not `Section`.** In CommonMark, a heading inside a blockquote or list item is syntactically valid and creates a heading element. MDOM deliberately does not promote such headings to `Section` nodes — they are parsed as `HeadingBlock` blocks, carrying level and text but no structural ownership. This preserves the invariant that `Section` nodes are always document-level structural scopes, and prevents content inside a blockquote from accidentally hijacking the document's Table of Contents. This is a scoped deviation from full CommonMark compatibility and is the right trade-off for a document-manipulation library targeting well-structured prose.

**Setext headers normalized on parse.** MDOM accepts Setext-style headers (`===` / `---` underlines) but normalizes them to hash equivalents during parsing. The lossless round-trip guarantee does not cover Setext sources. This is a pragmatic acceptance of real-world Markdown corpora without committing to full Setext fidelity.

**Strict mode by default for mixed header formats.** When a document mixes hash (`##`) and dot (`h2.`) header formats, MDOM throws a `FormatMixError` by default. A `{ strict: false }` parse option is available to auto-normalize instead. Strict mode is the default because silent normalization hides likely copy-paste errors in source documents.

**`setHeader()` and `setContent()` over multi-argument `replace()`.** The API provides explicit named methods for changing a section's header text and body content separately, rather than relying on magic argument positions. This makes agent-generated code unambiguous: "change the title" maps directly to `setHeader()`, "update the content" maps directly to `setContent()`. The combined `replace()` shorthand is retained for convenience when updating both simultaneously.

**`NodeHandle` is a stable reference, not a dynamic query.** A `NodeHandle` returned by `select()` or `selectAll()` holds a direct memory reference to the matched node. It does not re-evaluate the selector on subsequent calls. This means a handle remains valid after mutations that don't affect its node, and becomes stale if the node is `remove()`d. Agents and developers can safely store handles across multiple operations.

### 1.3 Setext Headers

MDOM accepts Setext-style headers (`===` and `---` underlines) at parse time but normalizes them to hash equivalents during parsing — `===` becomes `#`, `---` becomes `##` — and `doc.headingStyle` is set to `"setext"` to record that the source used this convention. On `render()`, if `headingStyle` remains `"setext"`, the output uses Setext notation; if changed to `"hash"`, hash notation is used. The lossless round-trip guarantee does not apply to Setext sources.

The primary motivation for accepting rather than rejecting Setext is pragmatic: a significant body of real-world Markdown — older GitHub READMEs, Jekyll content, Pandoc output — uses Setext headers, and throwing a parse error would create unnecessary friction for consumers of existing documents.

---

## 2. Core Architecture

### 2.1 Node Taxonomy

MDOM defines four node kinds. Understanding the distinction between Section and Block is fundamental to working with the model.

| Term | Role | Examples |
| :--- | :--- | :--- |
| **Document** | The root node. Contains top-level Sections and any leading Blocks. | — |
| **Section** | A *structural scope* opened by a document-level header. Has no content of its own — only a header that names it and children that belong to it. Owns sub-Sections and Blocks until closed by a peer or ancestor heading. | `# Intro`, `## Setup` |
| **Block** | An *atomic content unit* that renders as a discrete element. Holds data, not structure. Never opens a new structural scope. A Block may contain other Blocks (e.g. list items) but never Sections. | `Paragraph`, `CodeBlock`, `List`, `BlockQuote`, `ThematicBreak`, `Table`, `HeadingBlock` |
| **Inline** | A text-level element inside a Block or a Section header. | `Strong`, `Emphasis`, `Link`, `Code`, `Image` |

The key distinction: **Sections create hierarchy; Blocks create content.** Nesting works in one direction — Sections can contain Blocks and other Sections, but Blocks cannot contain Sections.

**Headings inside container blocks (`HeadingBlock`).** A heading token that appears inside a `BlockQuote` or `List` is parsed as a `HeadingBlock` — a Block node that carries a level and header text but does not open a structural scope, does not own subsequent siblings, and does not appear in `toc()` output. This is a deliberate deviation from CommonMark (see §1.2). The behavioral difference is intentional: a heading at document level is structural; a heading inside a blockquote is presentational.

### 2.2 The Section Model

Parsing is a single, linear pass over the token stream. When a Header token of level *N* is encountered **at document or Section scope** (i.e., not inside a container block), the parser:

1. Closes any open Section whose level is ≥ *N*.
2. Opens a new `Section` node with level *N*, associating the header text as its `header` property.
3. Appends all subsequent Block tokens as children of this Section, until the Section is closed by rule (1).

A Section of level *N* is closed when a Header of level ≤ *N* is encountered at document scope, or when the end-of-file is reached. Headers encountered inside `BlockQuote` or `List` container blocks do not trigger rule (1).

```
Document
├── Section[H1] "Getting Started"
│   ├── Paragraph "Welcome to..."
│   ├── Section[H2] "Installation"
│   │   ├── Paragraph "Run the following..."
│   │   └── CodeBlock (lang="bash")
│   └── Section[H2] "Configuration"
│       └── List
│           └── ListItem
│               ├── HeadingBlock[H3] "Sub-option"   ← not a Section
│               └── Paragraph "..."
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

When parsing, the format is inferred from the first header encountered. If headers of mixed formats are found in strict mode (the default), a `FormatMixError` is thrown. Pass `{ strict: false }` to auto-normalize to the first format encountered instead.

```typescript
// Strict mode (default) — throws FormatMixError on mixed formats
const doc = parse(src);

// Lenient mode — auto-normalizes to first format found
const doc = parse(src, { strict: false });
```

A utility function `normalizeHeaders(src, targetFormat)` is also provided for pre-processing source strings before parsing.

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
   * metadata but excluding the content of child nodes (which they own themselves).
   */
  raw: string;

  /** Character offsets into the original source string: [start, end). */
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

  /** For Section and HeadingBlock nodes: the header prefix as written
   *  (e.g. "##", "h2."). Null for all other node types. */
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
   * Controls whether render() outputs hash or Setext notation.
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
    | "HeadingBlock"   // heading inside a container block; not a structural Section
    | "BlockQuote"
    | "Table"
    | "ThematicBreak"
    | "HTMLBlock"
    | "LinkDefinition";

  /** Header level for HeadingBlock nodes (1–6); null for all other block types. */
  level: 1 | 2 | 3 | 4 | 5 | 6 | null;

  /**
   * Plain-text header content for HeadingBlock nodes.
   * Null for all other block types.
   */
  headerText: string | null;

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

MDOM uses a custom, Markdown-native selector syntax designed to be written and interpreted naturally by both human developers and LLMs. Method names and selector syntax are chosen to be unambiguous in natural language contexts.

Positional indices are **1-based** throughout — `##:1` selects the first H2, `p:2` selects the second paragraph. This aligns with how Table of Contents entries and document sections are numbered in natural language. The array returned by `selectAll()` is 0-based as standard in JavaScript; this distinction is intentional (see §1.2).

### 5.1 Section Selectors

| Syntax | Description |
| :--- | :--- |
| `##` | All H2 Sections (any header text). |
| `## [Installation]` | The H2 Section whose `headerText` equals `"Installation"` (case-insensitive exact match). The space between `##` and `[` is optional. |
| `## Installation` | Short-form Section selector for single-token headers only (`[A-Za-z0-9_-]+`). Equivalent to `## [Installation]`. |
| `##:2` | The **second** H2 in the document (1-based). |
| `## [Notes]:2` | The second H2 whose `headerText` equals `"Notes"` (1-based). |

**Matching rules for `[text]`:**

- Matching is against the node's `headerText` (plain-text rendering of the header Inlines).
- Matching is exact and case-insensitive.
- Whitespace and punctuation are significant.
- Short-form `## token` supports only a single token header name with letters, numbers, `_`, or `-`.

### 5.2 Block Selectors

| Syntax | Description |
| :--- | :--- |
| `p` | All Paragraph blocks. |
| `code` | All CodeBlock blocks. |
| `list` | All List blocks. |
| `ul` | Alias for `list` (unordered list compatibility). |
| `ol` | Alias for `list` (ordered list compatibility). |
| `list-item` | All ListItem blocks (excludes TaskItems). |
| `li` | Alias for `list-item` plus `task-item` (compatibility alias for list item targeting). |
| `task-item` | All TaskItem blocks. |
| `blockquote` | All BlockQuote blocks. |
| `table` | All Table blocks. |
| `hr` | All ThematicBreak blocks. |
| `heading` | All HeadingBlock blocks (headings inside container blocks). |

### 5.3 Attribute Filters

Attribute filters are appended to any selector using `[attr="value"]` syntax.

| Syntax | Description |
| :--- | :--- |
| `code[lang="js"]` | CodeBlocks whose `lang` property equals `"js"`. |
| `code[language="js"]` | Alias for `code[lang="js"]`. |
| `code[lang]` | CodeBlocks that have any non-null `lang`. |
| `task-item[status="x"]` | TaskItems with status `"x"`. |
| `task-item[status=""]` | TaskItems that are unchecked (open). |
| `heading[level="2"]` | HeadingBlock nodes at level 2. |

Multiple filters may be chained: `code[lang="ts"][lang!="tsx"]`. Supported operators: `=` (equals), `!=` (not equals), `^=` (starts with), `$=` (ends with), `*=` (contains).

Attribute comparisons are case-insensitive for string values. Both single quotes and double quotes are valid in attribute filters.

### 5.4 Combinators

| Syntax | Description |
| :--- | :--- |
| `## [API] > code` | **Child combinator.** Selects CodeBlocks that are *direct* children of the "API" H2 Section. Sub-sections are not traversed. |
| `## [Setup] p` | **Descendant combinator.** Selects all Paragraphs anywhere inside the "Setup" H2 Section (including those inside sub-sections). |
| `## [Intro] + p` | **Adjacent sibling combinator.** Selects the Paragraph immediately following the "Intro" H2 Section, within the same parent. |

**Combinator precedence** (highest to lowest): attribute filter `[]` > child `>` > adjacent sibling `+` > descendant ` ` (space).

### 5.5 Selector Parsing Rules

1. Tokenize left-to-right. Tokens are: `HashSequence`, `BracketText`, `ColonIndex`, `ElementType`, `AttributeFilter`, `Combinator`.
2. A `HashSequence` (one or more `#` characters) identifies a Section selector; its length is the heading level.
3. `BracketText` immediately following a `HashSequence` (with optional whitespace) is a semantic ID filter.
4. `ColonIndex` immediately following a `HashSequence`, `BracketText`, or `ElementType` is a 1-based positional filter.
5. Combinators are inferred from context: `>` is explicit; a single space between two non-combinator tokens implies descendant; `+` is explicit.
6. An unrecognized token raises a `SelectorSyntaxError`.

### 5.6 Unsupported Selector Features

The selector language is intentionally small. CSS pseudo-selectors are not supported. In particular, `:has(...)`, `:nth-child(...)`, and `:nth-of-type(...)` are invalid and must raise `SelectorSyntaxError`.

---

## 6. API Surface

All API methods are available on `NodeHandle` — a stable reference wrapper returned by `select()`, `selectAll()`, or the root `document` object. A `NodeHandle` holds a direct memory reference to its node; it does not re-evaluate the originating selector on subsequent calls. A handle remains valid across mutations that do not affect its node, and becomes stale if the node is `remove()`d — accessing a stale handle raises a `StaleHandleError`.

### 6.1 `select(selector): NodeHandle | null`

Returns a `NodeHandle` for the **first** node matching `selector` in document order, or `null` if no match is found.

### 6.2 `selectAll(selector): NodeHandle[]`

Returns a 0-based array of `NodeHandle` objects for **all** matching nodes in document order. Note: selectors use 1-based indices; the returned JavaScript array is 0-based (see §1.2).

### 6.3 `setHeader(text: string): void`

Updates the header text of the selected Section. The heading level and `style.heading` prefix are preserved; only the text content changes. Raises `InvalidOperationError` if called on a non-Section node.

```typescript
doc.select("## [Old Title]").setHeader("New Title");
```

### 6.4 `setContent(md: string): void`

Replaces the body content of the selected node — its Block children — with the parsed result of `md`. The node's header (if a Section) is unchanged. The operation is atomic — either the full parse succeeds or no mutation occurs and a `ParseError` is thrown.

```typescript
doc.select("## [Installation]").setContent("Run `npm install`.\n");
```

### 6.5 `replace(md: string, header: string): void`

Updates both the header text and body content of the selected Section simultaneously. Equivalent to calling `setHeader(header)` then `setContent(md)` in a single atomic operation. Raises `InvalidOperationError` if called on a non-Section node.

```typescript
doc.select("## [Draft]").replace("Final content.", "Final Version");
```

### 6.6 `before(md: string, header?: string): NodeHandle`

Parses `md` (and optionally `header`) and inserts the result as a sibling **immediately before** the current node. Returns a `NodeHandle` for the first inserted node.

### 6.7 `after(md: string, header?: string): NodeHandle`

Identical to `before`, but inserts **immediately after** the current node.

### 6.8 `append(md: string, header?: string): NodeHandle`

Appends parsed content as the **last child** of the current Section. If `header` is provided, a new sub-Section is created at level + 1. Raises `InvalidOperationError` on Block nodes.

### 6.9 `prepend(md: string, header?: string): NodeHandle`

Identical to `append`, but inserts as the **first child**.

### 6.10 `remove(): void`

Removes the node and all of its descendants from the tree. The removed node's `parent` is set to `null` and any stored `NodeHandle` for this node becomes stale. Because each node owns its own `style.spacingBefore`, the preceding sibling is unaffected — no orphaned whitespace is left behind.

### 6.11 `move(delta: number): void`

Moves the node `delta` positions within its parent's `children` array. Positive values move the node later (downward); negative values move it earlier (upward). `style.spacingBefore` travels with the node. If `delta` would move the node past the first or last sibling, the node is clamped to the boundary with no error thrown. A `delta` of `0` is a no-op.

```typescript
doc.select("## [Step 3]").move(-2);       // move up two positions
doc.select("## [Intro] > p:1").move(1);   // move the first paragraph down one
```

### 6.12 `children(): NodeHandle[]`

Returns `NodeHandle` wrappers for the immediate child nodes of the current node, in source order.

### 6.13 `parent(): NodeHandle | null`

Returns a `NodeHandle` for the containing Section or Document root. Returns `null` on the Document root.

### 6.14 `toc(): TocEntry[]`

Returns a nested Table of Contents structure derived from `Section` nodes only. `HeadingBlock` nodes inside container blocks are excluded.

```typescript
interface TocEntry {
  level: number;
  headerText: string;
  children: TocEntry[];
}
```

### 6.15 `render(): string`

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

In strict mode (the default), a document containing both hash-style and dot-style headers throws a `FormatMixError` at parse time. In lenient mode (`{ strict: false }`), the parser auto-normalizes to the first format encountered. Since the two formats are syntactically unambiguous — `##` cannot be mistaken for `h2.` — detection is reliable in both modes.

### 7.6 HeadingBlock Inside Container Blocks

A heading token encountered inside a `BlockQuote` or `List` is parsed as a `HeadingBlock` rather than opening a new `Section`. It does not close any open Section, does not appear in `toc()`, and cannot be targeted by Section selectors (`##`, `## [text]`). It can be targeted by the `heading` block selector and filtered by `[level]` attribute. See §1.2 for the rationale.

### 7.7 Stale NodeHandle

If a `NodeHandle` is stored and the referenced node is subsequently `remove()`d from the tree, the handle becomes stale. Any method call on a stale handle raises a `StaleHandleError`. Handles for nodes that are `move()`d or whose ancestors are mutated remain valid.

### 7.8 Deeply Nested Lists

List nesting is resolved at the Block level. A `List` Block may contain `ListItem` or `TaskItem` Blocks which may themselves contain further `List` Blocks. This does not interact with the Section hierarchy. `style.indent` is preserved per item.

### 7.9 move() at Boundaries

If `delta` would carry the node past the first or last position in the parent's `children` array, the node is clamped to the boundary. No error is thrown. If the node is the only child, `move()` is always a no-op regardless of `delta`.

### 7.10 Conflicting Selector Indices

If `##:5` is requested but fewer than 5 H2 Sections exist, `select` returns `null`; `selectAll` returns `[]`. No error is thrown.

### 7.11 Concurrent Mutation

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
| Source contains mixed hash and dot headers (strict mode) | ❌ `FormatMixError` thrown |
| Source contains mixed hash and dot headers (lenient mode) | ⚠️ Auto-normalized; original format of minority headers not preserved |

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

// setHeader() changes only the title — body is untouched
doc.select("## [Old Title]").setHeader("New Title");

console.log(doc.render());
// ## New Title
//
// This is the section body.
//
// - Item one
// - Item two
```

### 9.2 Update Section Content While Preserving the Title

```javascript
const doc = parse(`
## Installation

Old instructions here.
`);

// setContent() updates only the body — header is untouched
doc.select("## [Installation]").setContent("Run `npm install`.\n");
```

### 9.3 Move a Code Block Between Sections

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

### 9.4 Reorder Sections with move()

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

// Move Step 3 up two positions in a single call
doc.select("## [Step 3]").move(-2);

// Section order is now: Step 3, Step 1, Step 2
console.log(doc.toc()[0].children.map(e => e.headerText));
// ["Step 3", "Step 1", "Step 2"]
```

### 9.5 Working with TaskItems

```javascript
const doc = parse(`
## Sprint Backlog

- [x] Design the schema
- [ ] Write the parser
- [~] Draft the spec
- [ ] Add test coverage
`);

// Selector index :1 = first item (1-based); result array index [0] = first item (0-based)
const open = doc.selectAll(`## [Sprint Backlog] > list > task-item[status=""]`);
console.log(open.length); // 2

open[0].status = "~"; // mark first open item as in-progress
```

### 9.6 Selecting a HeadingBlock Inside a BlockQuote

```javascript
const doc = parse(`
## Notes

> ### Important
> This is a callout.
`);

// The H3 inside the blockquote is a HeadingBlock, not a Section
// It does NOT appear in toc()
console.log(doc.toc()); // [{ level: 2, headerText: "Notes", children: [] }]

// But it can be selected as a block
const heading = doc.select(`## [Notes] > blockquote > heading[level="3"]`);
console.log(heading); // NodeHandle → HeadingBlock { headerText: "Important" }
```

### 9.7 Render with a Different Header Format

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

### 9.8 Convert a Setext Document to Hash Notation

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

### 9.9 Query the Table of Contents

```javascript
function printToc(entries, depth = 0) {
  for (const entry of entries) {
    console.log("  ".repeat(depth) + `${"#".repeat(entry.level)} ${entry.headerText}`);
    printToc(entry.children, depth + 1);
  }
}

printToc(doc.toc());
```

---

## 10. Agent Tool Interface

MDOM exposes a set of high-level tool definitions designed for direct use by LLM agents and coding agents. These tools wrap the underlying library API, handling parse-select-mutate-render pipelines internally so that agents never need to manage tree state directly.

Each tool is stateless: it accepts a Markdown string as input and returns either extracted content or an updated Markdown string. This makes the tools composable in any agentic framework without session management or cleanup.

Tool names use the `markdown_` prefix to be immediately recognizable to any LLM without prior knowledge of MDOM.

### 10.1 Tool Overview

| Tool | Purpose | Typical position in workflow |
| :--- | :--- | :--- |
| `markdown_outline` | Orient — get the document map | First call on any unfamiliar document |
| `markdown_read` | Read — fetch a targeted section, block, or full document | Before any edit; also used for final export |
| `markdown_find` | Find — locate a node by natural language | When exact heading text is unknown |
| `markdown_edit` | Edit — apply one or more mutations atomically | The primary write tool |

Task management is provided by `markdown_tasks` — see §10.6.

### 10.1.1 Canonical Tool Descriptions

These are the authoritative natural-language descriptions for agent-facing tools. Implementations should keep runtime tool descriptions aligned with this section.

- `markdown_outline`: Get a table of contents for a Markdown document. Call this first on unfamiliar documents to map section structure with minimal tokens. Use `format: "text"` for quick orientation; use `format: "json"` when you need pre-computed selectors for follow-up tool calls.
- `markdown_read`: Read a specific section or block, or the full document. If a Section selector is used, return the section header plus all owned descendants (including nested subsections). Supports selector compatibility aliases (`ul`/`ol`/`list`, `li`/`list-item`/`task-item`, `code[lang=...]` / `code[language=...]`).
- `markdown_find`: Locate a section or block from natural language when exact heading text is unknown. Returns ranked candidates with selectors and confidence scores. Returned selectors are MDOM selectors (not CSS selectors).
- `markdown_edit`: Apply one or more document edits atomically in a single call. Supported operations are `replace`, `insert`, `remove`, `move`, and `substitute`. Prefer semantic selectors over positional selectors in multi-op batches to reduce selector drift.
- `markdown_tasks`: Query and mutate checkbox task items (`query`, `update`, `toggle`, `add`, `remove`). Use this for task state changes instead of `markdown_edit`.

**Selector compatibility note:** MDOM intentionally does not support CSS pseudo-selectors such as `:has(...)`, `:nth-child(...)`, or `:nth-of-type(...)`. Use MDOM-native positional syntax (`:N`) and attribute filters instead.

---

### 10.2 `markdown_outline`

Returns the document's Table of Contents. This is the orientation call — an agent should call it first on any document it hasn't seen before. The default `text` format returns a compact, human-readable TOC block that is token-cheap and immediately scannable. The `json` format returns structured data suitable for programmatic use.

**Parameters**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `markdown` | string | yes | The source Markdown document. |
| `depth` | integer (1–6) | no | Maximum heading level to include. Default: 6. |
| `format` | string | no | `"text"` (default) or `"json"`. |

**Returns — `format: "text"` (default)**

A plain text TOC block. Token-cheap, immediately readable, and suitable for pasting directly into a prompt.

```
# Getting Started
  ## Installation
  ## Configuration
# API Reference
  ## Methods
  ## Examples
```

**Returns — `format: "json"`**

Structured JSON with pre-computed selectors per section, suitable for programmatic use. Selectors are guaranteed to resolve correctly against the same document.

```json
{
  "sections": [
    {
      "level": 1,
      "title": "Getting Started",
      "selector": "# [Getting Started]",
      "children": [
        {
          "level": 2,
          "title": "Installation",
          "selector": "## [Installation]",
          "children": []
        }
      ]
    }
  ],
  "stats": {
    "sections": 12,
    "blocks": 47,
    "tasks": 8
  }
}
```

**Usage note:** When using `json` format, selectors returned in the output are guaranteed to resolve correctly against the same document. Agents should prefer these pre-computed selectors over generating their own when the section title is known.

---

### 10.3 `markdown_read`

Fetches the rendered Markdown of a node matched by selector — the header and all content it owns, including sub-sections. Returns only what was asked for, not the whole document. To read the entire document, omit the `selector` parameter or pass `"*"`.

Because `markdown_read` can return the full document, it also serves as the export step at the end of a multi-step editing pipeline — no separate render tool is needed.

**Parameters**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `markdown` | string | yes | The source Markdown document. |
| `selector` | string | no | MDOM selector identifying the target node. Omit or pass `"*"` to read the full document. For Sections, returns the header plus all owned descendants. Supports compatibility aliases (`ul`, `ol`, `li`) and `code[lang="..."]` / `code[language="..."]`. |
| `all` | boolean | no | If true, returns all matching nodes. Default: false (first match only). |
| `format` | string | no | `"markdown"` (default) or `"json"` (returns node metadata alongside content). |
| `maxTokens` | integer | no | Approximate token budget. Content is truncated with a truncation notice if exceeded. |

Headers are always rendered in hash format (`##`). Use `markdown_edit` with a `headerFormat` option if dot notation output is required.

**Returns — `format: "markdown"` (default)**

```json
{
  "content": "## Installation\n\nRun `npm install`.\n",
  "selector": "## [Installation]",
  "truncated": false
}
```

**Returns — `format: "json"`**

```json
{
  "selector": "## [Installation]",
  "type": "Section",
  "level": 2,
  "headerText": "Installation",
  "content": "## Installation\n\nRun `npm install`.\n",
  "truncated": false
}
```

When `all: true`, returns `{ "items": [ { ... }, ... ] }` in either format.

**Reading the full document**

```json
{ "markdown": "...", "selector": "*" }
```

Returns the entire document — useful as the final export step after a series of `markdown_edit` calls.

---

### 10.4 `markdown_find`

Locates a node using a natural language description rather than an exact selector. Returns ranked candidates with their exact selectors, which can then be passed to `markdown_read` or used in `markdown_edit` ops. This is the tool that makes the system robust when the agent does not know the precise heading text.

**Parameters**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `markdown` | string | yes | The source Markdown document. |
| `query` | string | yes | Natural language description of the target (e.g., `"the section about rate limiting"`). |
| `blockType` | string | no | Constrain results to a node type: `"section"`, `"code"`, `"list"`, `"task-item"`, etc. |
| `within` | string | no | Selector of a Section to search within (e.g., `"## [API Reference]"`). |
| `limit` | integer | no | Maximum number of candidates to return. Default: 3. |

#### 10.4.1 Scoring Algorithm

Candidates are ranked by a composite score in the range 0.0–1.0. The score is computed as an equal-weighted average of two Jaro-Winkler passes against the section heading text:

```
score = 0.5 × jaro_winkler(query_raw, heading_raw)
      + 0.5 × jaro_winkler(normalize(query), normalize(heading))
```

The raw pass preserves signal from intentional casing and punctuation. The normalized pass catches matches obscured by formatting variation. Equal weighting ensures neither pass dominates.

**Normalization pipeline** (applied to both query and heading before the second pass):

1. **Lowercase** — case-fold to Unicode lowercase
2. **Trim punctuation** — strip leading and trailing punctuation characters
3. **Strip trailing `:` or `(…)` variants** — remove trailing colons and parenthetical suffixes (e.g., `"Installation:"` → `"Installation"`, `"Rate Limiting (Advanced)"` → `"Rate Limiting"`)
4. **Collapse whitespace** — reduce all internal whitespace sequences to a single space and strip leading/trailing whitespace
5. **Unicode normalize (NFKC)** — decompose and recompose characters to canonical form, resolving homoglyph and encoding variation

**Jaro-Winkler** is chosen because it is designed for short strings, gives extra weight to common prefixes, and handles the transposition and minor character variation typical of heading text. It is deterministic, requires no corpus or embeddings, and is implemented in standard libraries across all major languages.

**Score interpretation**

| Score | Interpretation |
| :--- | :--- |
| ≥ 0.90 | Strong match — safe to proceed without inspection |
| 0.75–0.89 | Probable match — inspect `snippet` before writing |
| 0.60–0.74 | Weak match — consider calling `markdown_outline` and re-querying |
| < 0.60 | Poor match — do not use without manual confirmation |

**Returns**

```json
{
  "matches": [
    {
      "selector": "## [Rate Limiting]",
      "title": "Rate Limiting",
      "score": 0.94,
      "snippet": "Requests are limited to 100 per minute…"
    },
    {
      "selector": "## [API Reference] > ## [Limits]",
      "title": "Limits",
      "score": 0.71,
      "snippet": "See rate limiting policy for details…"
    }
  ]
}
```

**Usage note:** Always inspect the top candidate's `snippet` before writing to confirm it is the intended target. If the top score is below 0.75, call `markdown_outline` first to orient, then re-query with a more specific description.

---

### 10.5 `markdown_edit`

Applies one or more mutations to a document atomically. All ops in the array are applied sequentially to the parsed tree; if any op fails, the entire batch is rolled back and the original Markdown is returned unchanged. Returns the updated Markdown plus a minimal diff of what changed.

This is the primary write tool. An agent that needs to make multiple changes to a document should batch them into a single `markdown_edit` call rather than chaining multiple tool calls.

**Parameters**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `markdown` | string | yes | The source Markdown document. |
| `ops` | array | yes | Ordered array of operation objects. Applied sequentially. |
| `atomic` | boolean | no | If true (default), roll back all ops on any failure. If false, apply successful ops and report failures. |
| `headerFormat` | string | no | `"hash"` (default) or `"dot"`. Applied to the rendered output document. Aligns with `Document.headerFormat` in the library API. |
| `stylePolicy` | string | no | `"preserve"` (default) keeps original style; `"normalize"` applies consistent formatting. |

**Returns**

```json
{
  "markdown": "...",
  "applied": 3,
  "diff": "@@ -4,7 +4,7 @@\n-## Authentication\n+## Auth & Security\n...",
  "warnings": []
}
```

On failure (when `atomic: true`):

```json
{
  "markdown": "...",
  "applied": 0,
  "error": "Op 2 failed: selector '## [Nonexistent]' matched 0 nodes.",
  "diff": ""
}
```

#### 10.5.1 Operation: `replace`

Updates the header text, body content, or both of a matched Section. Header level and style are preserved.

```json
{
  "op": "replace",
  "selector": "## [Authentication]",
  "header": "Auth & Security",
  "content": "Updated body content here.\n"
}
```

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `selector` | string | yes | Target Section. |
| `header` | string | no | New header text. Level and formatting are preserved. |
| `content` | string | no | New body Markdown. Replaces all existing children. |

At least one of `header` or `content` must be provided.

#### 10.5.2 Operation: `insert`

Adds new content at a position relative to a target node. Parses the `markdown` fragment and inserts it into the tree.

```json
{
  "op": "insert",
  "selector": "## [Installation]",
  "where": "after",
  "markdown": "## Troubleshooting\n\nSee the FAQ.\n"
}
```

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `selector` | string | yes | Reference node. |
| `where` | string | yes | `"before"`, `"after"`, `"first-child"`, `"last-child"`. |
| `markdown` | string | yes | Content to insert. May include a heading to create a new Section. |

#### 10.5.3 Operation: `remove`

Removes a node and all its descendants.

```json
{
  "op": "remove",
  "selector": "## [Deprecated]",
  "match": "first"
}
```

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `selector` | string | yes | Target node. |
| `match` | string | no | `"first"` (default) or `"all"` to remove every matching node. |

#### 10.5.4 Operation: `move`

Relocates a node to a new position relative to a target node.

```json
{
  "op": "move",
  "selector": "## [Examples]",
  "target": "## [Installation]",
  "where": "after"
}
```

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `selector` | string | yes | Node to move. |
| `target` | string | yes | Reference node for the destination. |
| `where` | string | yes | `"before"`, `"after"`, `"first-child"`, `"last-child"`. |

#### 10.5.5 Operation: `substitute`

Performs a scoped find-and-replace within a node's content. Cheaper than `replace` for small textual changes that don't require rewriting a whole section body. Named after the Vim `substitute` command (`:s/find/replace/`).

```json
{
  "op": "substitute",
  "selector": "## [Changelog]",
  "find": "v0.3.0",
  "replace": "v0.5.0",
  "mode": "literal",
  "count": "all"
}
```

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `selector` | string | yes | Scope of the search — only content within this node is affected. |
| `find` | string | yes | Text to find. |
| `replace` | string | yes | Replacement text. |
| `mode` | string | no | `"literal"` (default) or `"regex"`. |
| `count` | string | no | `"first"` (default) or `"all"`. |

#### 10.5.6 Selector Drift in Batch Ops

When ops in the same batch use positional selectors (e.g., `p:2`), be aware that earlier ops may shift node positions and invalidate later positional selectors in the same batch. To avoid this:

- Prefer semantic selectors (`## [Section Name]`) and attribute filters (`[status="x"]`) over positional indices within a single batch.
- If positional selectors are necessary, order ops from last position to first (bottom-up), so earlier ops do not shift the targets of later ops.
- The `diff` in the response can be used to verify that all ops landed as intended.
- `substitute` ops are immune to positional drift since they match by text content, not position.

---

### 10.6 `markdown_tasks`

Manages task items across a Markdown document. All task operations — reading, updating, adding, removing — are handled by this single tool so that agents never need to leave it for task-related work. The `mode` parameter determines the operation.

`markdown_tasks` is intentionally scoped to task state management. It is not a general document editor — for inserting new sections or restructuring content around tasks, use `markdown_edit`.

**Common parameters (all modes)**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `markdown` | string | yes | The source Markdown document. |
| `mode` | string | yes | `"query"`, `"update"`, `"toggle"`, `"add"`, or `"remove"`. |
| `selector` | string | no | MDOM selector scoping the operation. Defaults to the full document. |
| `filter` | string | no | Attribute filter applied on top of `selector` (e.g., `[status=""]` for open tasks only). |

**Return shape for write modes (`update`, `toggle`, `add`, `remove`)**

```json
{
  "markdown": "...",
  "changed": [
    { "selector": "## [Sprint] > list > task-item:1", "from": " ", "to": "x" }
  ]
}
```

The `changed` array provides compact per-item confirmation without requiring the agent to re-read the document. For `add`, the new item's selector is included with `"from": null`. For `remove`, `"to": null`.

---

#### Mode: `query`

Returns all matching tasks as structured JSON. Use this to orient before updating, or to report task state to a user.

**Additional parameters**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `include` | array | no | Fields to return per task. Options: `"status"`, `"text"`, `"selector"`, `"section"`. Default: all. |

**Returns**

```json
{
  "tasks": [
    {
      "selector": "## [Sprint Backlog] > list > task-item:1",
      "text": "Write the parser",
      "status": "",
      "section": "Sprint Backlog"
    },
    {
      "selector": "## [Sprint Backlog] > list > task-item:2",
      "text": "Draft the spec",
      "status": "~",
      "section": "Sprint Backlog"
    }
  ],
  "counts": { "open": 1, "in-progress": 1, "complete": 0, "total": 2 }
}
```

**Example — query all incomplete tasks in a section**

```json
{
  "markdown": "...",
  "mode": "query",
  "selector": "## [Sprint Backlog]",
  "filter": "[status=\"\"]"
}
```

---

#### Mode: `update`

Updates one or more properties of matching task items. Currently supports `status`; designed to be extended with additional properties (e.g., due dates, assignees) in future versions without breaking existing calls.

**Additional parameters**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `status` | string | no | New status character. `""` for open, `"x"` for complete, any single character for custom. |
| `match` | string | no | `"first"` (default) or `"all"`. |

**Example — mark all in-progress items as complete**

```json
{
  "markdown": "...",
  "mode": "update",
  "selector": "## [Sprint Backlog]",
  "filter": "[status=\"~\"]",
  "status": "x",
  "match": "all"
}
```

---

#### Mode: `toggle`

Flips a task between open (`""`) and complete (`"x"`) without requiring a prior read. If the task is open, it becomes complete. If it is anything else, it becomes open. This eliminates a read-then-write round trip for the most common agentic pattern.

**Additional parameters**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `match` | string | no | `"first"` (default) or `"all"`. |

**Example — toggle the first incomplete task**

```json
{
  "markdown": "...",
  "mode": "toggle",
  "selector": "## [Today]",
  "filter": "[status=\"\"]"
}
```

---

#### Mode: `add`

Inserts one or more new task items into a list. New items are open (`""`) by default. If the target list does not exist, it is created automatically under the matched section.

**Additional parameters**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `items` | array of strings | yes | Task text for each new item (plain text, not Markdown). |
| `where` | string | no | `"last-child"` (default), `"first-child"`, `"before"`, `"after"`. |

**Example — append two tasks to the backlog**

```json
{
  "markdown": "...",
  "mode": "add",
  "selector": "## [Backlog] > list",
  "items": ["Write integration tests", "Update changelog"],
  "where": "last-child"
}
```

---

#### Mode: `remove`

Deletes matching task items and their content. Does not affect surrounding list items or section structure.

**Additional parameters**

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `match` | string | no | `"first"` (default) or `"all"`. |

**Example — remove all completed tasks from a list**

```json
{
  "markdown": "...",
  "mode": "remove",
  "selector": "## [Done]",
  "filter": "[status=\"x\"]",
  "match": "all"
}
```

---

### 10.7 Typical Agent Workflows

**Read and update a known section (2 calls)**

```
1. markdown_read(markdown, "## [Installation]")
   → inspect current content

2. markdown_edit(markdown, [
     { "op": "replace", "selector": "## [Installation]", "content": "..." }
   ])
   → updated markdown + diff
```

**Orient, find, and edit an unknown document (3 calls)**

```
1. markdown_outline(markdown)
   → get section map and pre-computed selectors

2. markdown_read(markdown, "## [Authentication]")
   → inspect target section before editing

3. markdown_edit(markdown, [
     { "op": "replace",    "selector": "## [Authentication]", "header": "Auth & Security" },
     { "op": "substitute", "selector": "## [Auth & Security]", "find": "v1", "replace": "v2", "count": "all" },
     { "op": "insert",     "selector": "## [Auth & Security]", "where": "last-child",
       "markdown": "See migration guide for upgrade steps.\n" }
   ])
   → updated markdown + diff confirming 3 ops applied
```

**Locate by description when heading text is uncertain (3 calls)**

```
1. markdown_find(markdown, "the section about OAuth token refresh")
   → returns selector "## [Token Refresh]" with score 0.91

2. markdown_read(markdown, "## [Token Refresh]")
   → confirm this is the right section

3. markdown_edit(markdown, [
     { "op": "replace", "selector": "## [Token Refresh]", "content": "..." }
   ])
```

**Agent task loop — process a sprint board (3 calls)**

```
1. markdown_tasks(markdown, mode="query", selector="## [Sprint Backlog]")
   → returns structured JSON of all tasks with statuses and selectors
   → agent identifies which tasks to act on from the counts and task list

2. markdown_tasks(markdown, mode="update",
     selector="## [Sprint Backlog]", filter="[status=\"~\"]",
     status="x", match="all")
   → marks all in-progress tasks complete
   → returns updated markdown + changed[] confirmation

3. markdown_tasks(markdown, mode="add",
     selector="## [Sprint Backlog] > list",
     items=["Write release notes", "Tag v1.0.0"],
     where="last-child")
   → appends two new open tasks
   → returns updated markdown + changed[] with new selectors
```
