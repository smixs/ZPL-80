
![Remove Background Image (5)](https://github.com/user-attachments/assets/0fab9da9-2d9c-414f-892b-c5e82b52eeeb)

# ZPL-80 – Zip Prompt Language v1.0
**Complete Technical Specification**
---

### 1. Purpose and Design Goals

1. Compress long, rule-heavy or data-heavy system prompts by **≥ 80 % token reduction** while remaining fully interpretable by any LLM that understands this spec.
2. Preserve **semantic fidelity**: nothing important is lost, merely encoded.
3. Keep syntax **human-writable** and **deterministically expandable** back to canonical plain text.
4. Require **no external decoder**: the prompt itself always embeds all macros and symbol legends needed for expansion.
5. Remain UTF-8 and Markdown compatible, so prompts flow through existing LLM channels unchanged.

---

### 2. File Structure

```
%MACROS          # section A – macro / alias definitions
%SYMBOLS         # section B – single-byte or emoji symbol table
📥 DATA … %END   # section C – payload(s) (can repeat)
⚙️  TASK … %END   # section D – instructions / tasks (can repeat)
%END             # final fence (closes the entire prompt)
```

*The sequence A → B → C* (one or many) *→ D* (one or many) *→ final %END is mandatory.*

---

### 3. Syntax Fundamentals

| Element             | Syntax                                    | Notes                                        |
| ------------------- | ----------------------------------------- | -------------------------------------------- |
| **Directive Fence** | `%MACROS`, `%SYMBOLS`, `%END`, `📥`, `⚙️` | Always starts at column 1                    |
| **Key–Value**       | `key:value` or `key=value`                | Colon or equals interchangeable              |
| **Tuple / Array**   | `[item,item,…]`                           | Comma-separated; primitives or nested arrays |
| **Obj / Map**       | `{key:value,…}`                           | JSON-style; whitespace optional              |
| **Comment**         | `#` until end-of-line                     | Ignored by expansion engine                  |
| **Escape**          | `\%`, `\#`, `\`                           | Escape special chars                         |

Tokens are case-sensitive except directive words (`%MACROS`, etc.).

---

### 4. Macro System (`%MACROS`)

* Define **aliases** to shrink recurring long strings.
* Left side = macro name (`[A-Z][A-Z0-9_]*`), right side = literal replacement text.
* Macros may reference earlier macros but **no forward reference**.

Example

```
%MACROS
KPI = "total_revenue,total_sales,average_price,unique_dealers"
BD  = "body_type_revenue"
%END
```

In payloads, write `KPI`; expander will inline the full string.

---

### 5. Symbol Table (`%SYMBOLS`)

* Map single UTF-8 code points (often emoji or punctuation) to verbose meanings.
* Each mapping saves tokens because the symbol later replaces a whole word.

Example

```
%SYMBOLS
📥 = ACTIVE_MEMORY
📤 = CACHED_MEMORY
🗄️ = ARCHIVED_MEMORY
⚙️ = TASK
%END
```

Symbols **must** be unique single code points.

---

### 6. Payload Blocks

#### 6.1 Data Blocks (`📥 DATA … %END`)

* Hold structured information: config, datasets, JSON, YAML, CSV, etc.
* **First token after `📥` is a label** (one word, PascalCase).
* Everything until the matching `%END` belongs to that data label.
* You may include multiple data blocks; label distinguishes them.

Example

```
📥 DATA SALES
{TR:671525465,TS:23906}
%END
```

#### 6.2 Task Blocks (`⚙️ TASK … %END`)

* Contain *actionable instructions* for the LLM.
* Header format: `⚙️ TASK|ID|conf:<H/M/L>`
* Body: free mix of key–value lines, ascii arrows, pseudo-code, etc.
* One block = one atomic task.

Example

```
⚙️ TASK|SPA_DASHBOARD|conf:H
UI_SEC:S1📈|S2🚗|S3🏢
%END
```

---

### 7. Compression Strategies

1. **Alias long keys** with `%MACROS`.
2. **Symbolise** ultra-common nouns (`PROJECT`, `STATUS`, etc.) via `%SYMBOLS`.
3. Collapse **whitespace and quotes** where JSON tolerates it (`{x:1,y:2}` vs full).
4. Replace repeated string sets with **arrays or index tables**.
5. Omit fields that hold *default values* (define defaults once in the spec).
6. Convert nested objects to **row arrays** when field order is fixed.

Result: 70-90 % token drop versus naïve plain English.

---

### 8. Expansion Algorithm (for reference)

1. Read full prompt.
2. Parse `%MACROS`, build macro dictionary.
3. Parse `%SYMBOLS`, build symbol dictionary.
4. For each block, recursively **expand symbols**, then **expand macros**.
5. Yield canonical plaintext.

All steps are deterministic; no semantic inference required.

---

### 9. Reserved Symbols & Keywords

| Token                       | Reserved For          |
| --------------------------- | --------------------- |
| `%MACROS` `%SYMBOLS` `%END` | Section fences        |
| `📥` `📤` `🗄️` `⚙️`        | Memory & task markers |
| `conf` `id` `name`          | Common scalar keys    |
| ASCII arrows `→`, `←`, etc. | Flow diagrams         |

Avoid redefining or reusing these.

---

### 10. Confidence Codes

Use three-level shorthand inside tasks or decisions:

* **H** (≥ 85 %)
* **M** (60-84 %)
* **L** (< 60 %)

Attach to any instruction or datum: `conf:H`.

---

### 11. Recommended Section-Types (Guideline)

| Block Type        | Typical Content                         |
| ----------------- | --------------------------------------- |
| `📥 DATA META`    | project name, authors, timestamps       |
| `📥 DATA SCHEMA`  | list of component ids, dependency graph |
| `📥 DATA PAYLOAD` | business datasets, tables               |
| `⚙️ TASK PLAN`    | step-by-step roadmap                    |
| `⚙️ TASK EXEC`    | single implementation order             |
| `⚙️ TASK DEBUG`   | repro steps, expected vs actual         |

Not enforced; names are free but stay singular and PascalCase.

---

### 12. Edge Flags (Behaviour Toggles)

* `DEPTH(n)`  → limit reasoning recursion to *n* levels
* `LINT?`   → if present, add quick refactor tips
* `TOKENS_LEFT?`→ report estimated remaining tokens
* `THINK SILENT` / `THINK OUT` → hide or show chain-of-thought
* `ABORT_IF(risk)`→ pause and ask if *risk* condition holds

Flags can be placed anywhere inside a TASK block.

---

### 13. Error-Handling Rules

* **Missing Section:** LLM **must** ask the user which section is absent.
* **Macro Collision:** later duplicate key overrides earlier and **raises low confidence**.
* **Undefined Symbol/Macro in Payload:** treat as literal and flag `[GAP]`.

---

### 14. Versioning & Precedence

1. **Newer timestamp** overrides older content.
2. **Explicit Decision** (block tagged `DECISION`) overrides implicit inference.
3. **Higher confidence** beats lower when merging contradictory tasks.

---

### 15. Security & Privacy

* ZPL-80 conveys data exactly as written; **do not** infer hidden intent.
* The spec carries no executable code, only declarative instructions.
* If a TASK requests unsafe operations, the LLM must still follow its own global safety policy before execution.

---

### 16. Minimal Working Example

```
%MACROS
TR = "total_revenue"
%END

%SYMBOLS
⚙️ = TASK
%END

📥 DATA KPI
{TR:671525465}
%END

⚙️ TASK|SHOW_KPI|conf:H
Print KPI TR on screen.
%END
%END
```

*Expansion →* `Print total_revenue 671525465 on screen.`

---

### 17. Compliance Checklist for Prompt Authors

* [ ] Sections ordered and closed with `%END`.
* [ ] All macros referenced are defined above first use.
* [ ] Symbols unique single-codepoints.
* [ ] Tasks carry `|ID|` handle.
* [ ] No reserved keywords redefined.
* [ ] Total size ≤ model context after compression.

---

**ZPL-80 ends.**
