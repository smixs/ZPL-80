


# ZPL-80 — Zip Prompt Language v1.1

## Compact Query Language for Token Efficiency

![Remove Background Image (5)](https://github.com/user-attachments/assets/0fab9da9-2d9c-414f-892b-c5e82b52eeeb)

## Why ZPL-80 Exists

Large prompts burn tokens, time, and cash.
ZPL-80 compresses instructions by ~80% while staying readable to any modern LLM.
Version 1.1 keeps the good parts of v1.0, drops the baggage, and builds in flexible CoT, format flags, and model wrappers.

## Core Design Rules

| Rule | What it means |
| --- | --- |
| **Zero dead tokens** | Every character must add meaning for the model |
| **Atomic blocks** | Prompt = sequence of self-describing blocks; omit what you don't need |
| **Short, stable labels** | `CTX`, `Q`, `A`, `Fmt`, `Thought`, etc. One- or two-word labels only |
| **System first** | Global rules live in the API's system role (or `[INST]…` wrapper for Llama) |
| **Model aware** | Add the wrapper tokens the target model expects—nothing more |
| **Optional CoT** | Fire chain-of-thought only for hard tasks via a single 🧠 trigger |
| **Token caps** | Limit verbose sections with inline guards: `Thought(TH<=128):` |

## Syntax Cheat-Sheet

```
%MACROS … %END     # global aliases
%SYMBOLS … %END    # single-char tokens → phrases

<<SYS>> … <</SYS>> # system message (optional)

CTX: …             # context / data (optional)
Q:   …             # the actual user query (required)
Fmt: ⧉             # ⧉=JSON, 📑=markdown, ✂️=plain text (optional)
Lang: EN           # target language (optional)
Thought(TH<=64):🧠  # CoT block, capped at 64 tokens (optional)
A:                 # assistant's final answer (required)

⌛                  # ask the model to report tokens left (optional)
```

*Block order is free but recommended:* **CTX → Q → Fmt/Lang → Thought → A**.
Omit any block that isn't needed.

## Macro & Symbol Tables

### `%MACROS`

Write a long phrase once, reuse it everywhere.

```
%MACROS
TR=total_revenue TTL=5   # TTL deletes the alias after 5 sessions
%END
```

### `%SYMBOLS`

Map one glyph to a whole term.

```
%SYMBOLS
🧠=COT    ⧉=JSON   📑=MARKDOWN   ⌛=TOK_LEFT
⚙️=TASK   ⛔=ABORT
%END
```

## Minimal Working Example

```zpl
%MACROS TR=total_revenue %END
%SYMBOLS 🧠=COT ⧉=JSON ⌛=TOK_LEFT %END

<<SYS>>You are a senior Python engineer. Be concise.<</SYS>>

CTX:
```py
def foo(x): return x**2
```

Q: Explain the code.
Fmt:⧉
Thought:🧠
A:
⌛
```

*Outcome*—model thinks step-by-step (hidden in `Thought`), replies with a compact JSON answer, then whispers remaining tokens.

## Blind Spots + Hard Fixes

| Pain point | Loss source | Fix | Example |
|------------|-------------|-----|---------|
| **Bloated macro list** | Obsolete aliases ride along | Add `TTL=n` to expire | `USR="user_prompt" TTL=3` |
| **Verbose JSON keys** | `total_revenue` repeated | Local alias inside `DATA` | `:TR:=total_revenue {TR:671525465}` |
| **CoT trigger is wordy** | "Let's think step by step." = 6-8 tokens | Emoji trigger | `Thought:🧠` |
| **Verbose format line** | "Format: JSON array …" | One-glyph codes | `Fmt:⧉` |
| **Model wrappers differ** | Llama vs GPT | `%M.BRK%` variable replaces with `[INST]` / none | `%M.BRK% CTX:…` |
| **Unbounded thoughts** | Model rambles 500 tokens | Token guard | `Thought(TH<=128):🧠` |
| **No token report** | Risk overflow | `⌛` flag | append `⌛` |

```

## Compression Tactics

1. **Alias anything ≥3 repeats**
2. **Collapse whitespace**: single newline separates blocks
3. **Strip pleasantries** ("Please" buys no accuracy)
4. **Inline small lists** `[1,2,3]` instead of five lines
5. **Cut CoT when the task is trivial**
6. **Send diffs only**: transmit changed keys, not whole `DATA`

## Model-Specific Wrappers

| Model family | Wrapper syntax |
|--------------|----------------|
| **OpenAI Chat** | Use `system`, `user`, `assistant` roles. Put ZPL in `user` |
| **Anthropic**   | Same; no extra tokens |
| **Llama-chat**  | `<s>[INST] … [/INST]` around the whole prompt |
| **Plain completions** | No wrapper, just the ZPL text |

## Compliance Checklist

- [ ] Only needed blocks present
- [ ] No decorative lines or duplicated labels
- [ ] All macros defined before first use
- [ ] Symbols are single code-points
- [ ] Format / language flags set if required
- [ ] CoT token cap added for heavy tasks
- [ ] Wrapper matches target model

## Roadmap

* v1.2 will add an auto-compression helper script and built-in diff-mode for interactive editors (Cursor, WindSurf)
* Community feedback welcome—open an issue or PR with failing edge cases

**Ship lighter prompts, spend fewer tokens, keep full clarity.**

---

**ZPL-80 ends.**
