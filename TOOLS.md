{
  markdown_outline: {
    type: "function",
    function: {
      name: "markdown_outline",
      description:
        "Get a table of contents for a Markdown document. Call this first when you receive an unfamiliar document — it shows you the full section structure in minimal tokens so you can identify which sections to read with markdown_read or target with markdown_edit. Use format='text' (default) for a quick readable overview; use format='json' when you need pre-computed selectors to pass directly to other tools.",
      parameters: {
        type: "object",
        additionalProperties: false,
        properties: {
          markdown: {
            type: "string",
            description: "The Markdown document to outline.",
          },
          depth: {
            type: "integer",
            minimum: 1,
            maximum: 6,
            description:
              "Deepest heading level to include. Use 2 for a high-level overview, 6 (default) for the full structure.",
          },
          format: {
            type: "string",
            enum: ["text", "json"],
            description:
              "text (default): returns a compact indented TOC — use this for orientation. json: returns structured data with selectors per section — use this when you need to programmatically select sections for markdown_read or markdown_edit.",
          },
        },
        required: ["markdown"],
      },
    },
  },

  markdown_read: {
    type: "function",
    function: {
      name: "markdown_read",
      description:
        "Read the content of a specific section or block — or the entire document. Use this to inspect content before editing, to verify a section exists, or to export the final document after edits. Pass a selector to read just one section (e.g. '## [Installation]'); omit selector or pass '*' to read the whole document. If selector targets a Section, output includes the section header and all owned child content (including nested subsections). Prefer reading only the section you need — this keeps token usage low and your context focused.",
      parameters: {
        type: "object",
        additionalProperties: false,
        properties: {
          markdown: {
            type: "string",
            description: "The Markdown document to read from.",
          },
          selector: {
            type: "string",
            description:
              "MDOM selector identifying the section or block to read (e.g. '## [API Reference]', '## Setup > code[language=\"bash\"]', '## [Tasks] > ul > li:2'). Omit or pass '*' to read the full document. Compatibility aliases are supported: ul/ol/list, li/list-item/task-item, and code attributes lang/language. Section title matching is case-insensitive. Use selectors from markdown_outline output when available. CSS pseudo-selectors like :has(...) and :nth-of-type(...) are invalid.",
          },
          all: {
            type: "boolean",
            description:
              "Set to true to return every node matching the selector. Default false returns only the first match. Useful when scanning for all code blocks or all sections at a given level.",
          },
          format: {
            type: "string",
            enum: ["markdown", "json"],
            description:
              "markdown (default): returns the section as rendered Markdown. json: returns node metadata (type, level, headerText) alongside content — use when you need structural information about the node, not just its text.",
          },
          maxTokens: {
            type: "integer",
            minimum: 1,
            description:
              "Stop returning content after approximately this many tokens. A truncation notice is included in the response. Use this to stay within context limits on large sections.",
          },
        },
        required: ["markdown"],
      },
    },
  },

  markdown_find: {
    type: "function",
    function: {
      name: "markdown_find",
      description:
        "Find a section or block using a plain English description when you don't know its exact heading text. Returns ranked candidates with selectors and confidence scores. Use this when you need to locate something but aren't certain of the exact wording — for example, 'the section about authentication' or 'the bash install example'. Always check the score and snippet of the top result before writing: scores above 0.90 are safe to use directly; scores below 0.75 suggest you should call markdown_outline first to orient yourself. Returned selectors are MDOM selectors, not CSS selectors.",
      parameters: {
        type: "object",
        additionalProperties: false,
        properties: {
          markdown: {
            type: "string",
            description: "The Markdown document to search.",
          },
          query: {
            type: "string",
            description:
              "Plain English description of what you're looking for (e.g. 'the rate limiting section', 'the JavaScript installation example', 'the section about error codes').",
          },
          blockType: {
            type: "string",
            enum: ["section", "code", "list", "task-item", "table", "blockquote", "heading"],
            description:
              "Restrict matches to a specific node type. Omit to search all node types.",
          },
          within: {
            type: "string",
            description:
              "MDOM selector of a section to search within. Use this to narrow scope when you know the parent section (e.g. '## [API Reference]'). Omit to search the whole document. Supports ul/ol/li aliases and lang/language for code selectors.",
          },
          limit: {
            type: "integer",
            minimum: 1,
            description:
              "Maximum number of candidates to return. Default 3. Increase if the top result looks wrong and you want alternatives.",
          },
        },
        required: ["markdown", "query"],
      },
    },
  },

  markdown_edit: {
    type: "function",
    function: {
      name: "markdown_edit",
      description:
        "Apply one or more edits to a Markdown document in a single atomic call. Batch all the changes you need to make into one call — this is more reliable than chaining multiple calls and avoids intermediate state issues. Operations are applied in order; if any operation fails, the whole batch is rolled back by default and the original document is returned unchanged. Available operations: 'replace' (update a section's header or body), 'insert' (add new content before, after, or inside a node), 'remove' (delete a node and its children), 'move' (relocate a node relative to another), 'substitute' (find-and-replace text within a scoped node — use for small changes like version numbers or terminology).",
      parameters: {
        type: "object",
        additionalProperties: false,
        properties: {
          markdown: {
            type: "string",
            description: "The Markdown document to edit.",
          },
          ops: {
            type: "array",
            description:
              "Ordered list of edit operations. Each operation is an object with an 'op' field ('replace', 'insert', 'remove', 'move', 'substitute') plus op-specific fields. Operations are applied sequentially — later ops see the result of earlier ones. Prefer semantic selectors (e.g. '## [Section Name]') over positional selectors (e.g. 'p:2') to avoid selector drift across ops.",
            items: {
              type: "object",
              required: ["op", "selector"],
              properties: {
                op: {
                  type: "string",
                  enum: ["replace", "insert", "remove", "move", "substitute"],
                  description:
                    "replace: update a section's header text, body content, or both. insert: add new Markdown before, after, or inside a node. remove: delete a node and all its children. move: relocate a node to a new position relative to a target node. substitute: scoped find-and-replace within a node — cheaper than replace for small textual changes.",
                },
                selector: {
                  type: "string",
                  description:
                    "MDOM selector identifying the source node for this operation — the node being changed, removed, or moved. For move ops, this is the node being relocated, not its destination (use 'target' for that). Examples: '## [Installation]', '## API > code[language=\"js\"]', '## [Plan] > li:2', 'task-item[status=\"\"]'. Compatibility aliases are supported (ul/ol/li and lang/language). CSS pseudo-selectors are invalid.",
                },
                header: {
                  type: "string",
                  description:
                    "replace op only: new header text. The heading level and formatting are preserved — only the text changes.",
                },
                content: {
                  type: "string",
                  description:
                    "replace op only: new body content as a Markdown string. Replaces all existing children of the section.",
                },
                where: {
                  type: "string",
                  enum: ["before", "after", "first-child", "last-child"],
                  description:
                    "insert op: position of the new content relative to the selector node — 'before'/'after' as a sibling, 'first-child'/'last-child' inside the node. move op: position of the relocated node relative to the 'target' node — 'before'/'after' as a sibling of target, 'first-child'/'last-child' inside target.",
                },
                markdown: {
                  type: "string",
                  description:
                    "insert op only: the Markdown content to insert. May include headings to create new sections.",
                },
                target: {
                  type: "string",
                  description:
                    "move op only: MDOM selector of the destination reference node. The moved node is placed relative to this target using 'where'.",
                },
                match: {
                  type: "string",
                  enum: ["first", "all"],
                  description:
                    "remove op only: 'first' (default) removes only the first match; 'all' removes every matching node.",
                },
                find: {
                  type: "string",
                  description: "substitute op only: the text to search for.",
                },
                replace: {
                  type: "string",
                  description: "substitute op only: the replacement text.",
                },
                mode: {
                  type: "string",
                  enum: ["literal", "regex"],
                  description:
                    "substitute op only: 'literal' (default) for plain text matching; 'regex' for regular expression matching.",
                },
                count: {
                  type: "string",
                  enum: ["first", "all"],
                  description:
                    "substitute op only: 'first' (default) replaces only the first occurrence within scope; 'all' replaces every occurrence.",
                },
              },
            },
          },
          atomic: {
            type: "boolean",
            description:
              "Default true: if any operation fails, roll back all changes and return the original document with an error. Set to false only when you want partial success — successful ops will be applied even if others fail.",
          },
          headerFormat: {
            type: "string",
            enum: ["hash", "dot"],
            description:
              "Output header notation. 'hash' (default): standard Markdown (## Heading). 'dot': wiki-style notation (h2. Heading) for JIRA, Confluence, and similar platforms.",
          },
          stylePolicy: {
            type: "string",
            enum: ["preserve", "normalize"],
            description:
              "'preserve' (default): keep original bullet characters, fence styles, and spacing. 'normalize': apply consistent formatting throughout the document.",
          },
        },
        required: ["markdown", "ops"],
      },
    },
  },

  markdown_tasks: {
    type: "function",
    function: {
      name: "markdown_tasks",
      description:
        "Read and manage task items (checkboxes) in a Markdown document. Use this tool for everything task-related — querying status, checking items off, adding new tasks, or removing completed ones. Do not use markdown_edit for task state changes; use this tool instead. The 'mode' field controls what the tool does: query (read tasks), update (change status), toggle (flip open/complete without reading first), add (insert new tasks), remove (delete tasks).",
      parameters: {
        type: "object",
        additionalProperties: false,
        properties: {
          markdown: {
            type: "string",
            description: "The Markdown document containing the task list.",
          },
          mode: {
            type: "string",
            enum: ["query", "update", "toggle", "add", "remove"],
            description:
              "query: return matching tasks as structured JSON — use this to inspect task state before acting on it. update: change the status of matching tasks to a specified value. toggle: flip a task between open ('') and complete ('x') without needing to read its current state first — use this to quickly check off a task. add: insert new open task items into a list; creates the list if it doesn't exist. remove: delete matching task items.",
          },
          selector: {
            type: "string",
            description:
              "MDOM selector scoping the operation to a section or list (e.g. '## [Sprint Backlog]', '## [Sprint Backlog] > ul', '## [Sprint Backlog] > li:2'). Omit to operate across the whole document. Compatibility aliases are supported (ul/ol/list and li/list-item/task-item).",
          },
          filter: {
            type: "string",
            description:
              "Attribute filter to narrow which tasks are affected. Examples: '[status=\"\"]' for open tasks only, '[status=\"x\"]' for completed tasks, '[status=\"~\"]' for in-progress tasks. Combine with selector to precisely target a subset.",
          },
          include: {
            type: "array",
            items: { type: "string" },
            description:
              "query mode only: which fields to return per task. Options: 'status', 'text', 'selector', 'section'. Default returns all fields.",
          },
          status: {
            type: "string",
            description:
              "update mode only: the status character to apply. Use '' (empty string) for open/unchecked, 'x' for complete, or any single character for a custom status (e.g. '~' for in-progress, '-' for cancelled, '?' for blocked).",
          },
          match: {
            type: "string",
            enum: ["first", "all"],
            description:
              "update, toggle, remove modes: 'first' (default) affects only the first matching task; 'all' affects every task matching the selector and filter combination. Use 'all' for bulk status changes.",
          },
          items: {
            type: "array",
            items: { type: "string" },
            description:
              "add mode only: plain text labels for the new task items to insert (e.g. ['Write tests', 'Update docs']). Each string becomes one task item. Do not include Markdown syntax — MDOM handles formatting.",
          },
          where: {
            type: "string",
            enum: ["first-child", "last-child", "before", "after"],
            description:
              "add mode only: where to insert new tasks relative to the matched list. 'last-child' (default) appends to the end of the list. 'first-child' prepends to the top. 'before'/'after' inserts relative to the first task-item matched by the selector.",
          },
        },
        required: ["markdown", "mode"],
      },
    },
  },
}
