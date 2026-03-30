# Dialogue Scripting

This guide explains how to write dialogue strings in `.inc` script files. Dialogue is defined using `.string` directives and a set of control character macros that tell the game how to display text.

---

## How the Text Box Works

The in-game message box displays **two lines at a time**. Everything you write goes into that box, and the control macros tell the game when to move to the next line, scroll the box, clear it, or end the string entirely.

The `.string` directive splits are purely for readability in the source file — the assembler concatenates them all into one flat sequence of bytes. Only the control macros affect what the player sees.

---

## The Four Control Macros

| Macro | Byte | What it does |
|-------|------|--------------|
| `\n`  | `FE` | Move to line 2 within the current box |
| `\l`  | `FA` | Scroll the box up one line (no button press needed) |
| `\p`  | `FB` | Clear the box and start fresh (player must press A first) |
| `$`   | —    | End the string |

### `\n` — New Line

Moves to the second line of the current box. Use this to fill both lines before deciding what to do next (`\l`, `\p`, or `$`).

```
"First line of text\n"
"Second line of text.\p"
```

```
┌─────────────────────────────────┐
│ First line of text              │
│ Second line of text.            │  ← player presses A
└─────────────────────────────────┘
```

### `\l` — Scroll Up

Scrolls the box up one line without requiring a button press. Line 1 disappears, line 2 becomes the new line 1, and the next piece of text appears on the new line 2. This lets you show three lines of a flowing thought without a hard paragraph break.

```
"First line of text\n"
"Second line of text\l"
"Third line of text.\p"
```

```
┌─────────────────────────────────┐
│ First line of text              │
│ Second line of text             │  ← scrolls automatically
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Second line of text             │
│ Third line of text.             │  ← player presses A
└─────────────────────────────────┘
```

### `\p` — New Paragraph

Waits for the player to press A, then clears the box entirely and starts the next text from the top. Use this between separate thoughts or sentences.

### `$` — End of String

Terminates the string. Every dialogue string must end with `$`. No `\p` is needed before `$` — the game closes the box on its own.

---

## Common Patterns

### One sentence, one box

Short sentences that fit on two lines or fewer:

```inc
LittlerootTown_Text_CanUsePCToStoreItems:
	.string "If you use a PC, you can store items\n"
	.string "and POKéMON.\p"
	.string "The power of science is staggering!$"
```

```
┌─────────────────────────────────┐
│ If you use a PC, you can store  │
│ items and POKéMON.              │  ← player presses A
└─────────────────────────────────┘
┌─────────────────────────────────┐
│ The power of science is         │
│ staggering!                     │  ← end
└─────────────────────────────────┘
```

### Three-line flow with `\l`

When a single thought is too long for two lines but isn't a new paragraph:

```inc
LittlerootTown_Text_BirchSpendsDaysInLab:
	.string "PROF. BIRCH spends days in his LAB\n"
	.string "studying, then he'll suddenly go out in\l"
	.string "the wild to do more research…\p"
	.string "When does PROF. BIRCH spend time\n"
	.string "at home?$"
```

```
┌─────────────────────────────────┐
│ PROF. BIRCH spends days in his  │
│ LAB studying, then he'll        │  ← scrolls automatically
│ suddenly go out in              │
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ studying, then he'll suddenly   │
│ go out in the wild to do more   │
│ research…                       │  ← player presses A
└─────────────────────────────────┘
```

### Single-line box with `\p`

A short, punchy line on its own is fine — you don't have to fill both lines:

```inc
	.string "Um, um, um!\p"
	.string "If you go outside and go in the grass,\n"
	.string "wild POKéMON will jump out!$"
```

---

## Rules of Thumb

1. **`\p` ends a complete thought.** Never put a `\p` in the middle of a sentence. The player will read the first half, press A, and then feel like the second half came out of nowhere.

2. **`\l` continues a thought.** Use it when one sentence is naturally three visual lines long. Don't use it to hop between unrelated ideas.

3. **`\n` fills line 2.** Almost every box should have a `\n` before its `\p` or `\l`, unless the line is very short and intentionally punchy on its own.

4. **End with `$`, not `\p$`.** `\p` before `$` would make the player press A into a blank box. Just end with `$`.

5. **Keep sentences together.** If a sentence starts in one box, it should end in the same box.

---

## Common Mistake: Splitting a Sentence Across `\p`

This is the most frequent error and it's easy to miss because the source looks plausible:

**Wrong:**
```inc
	.string "I should teach you how to use.\p"
	.string "your RUNNING SHOES.\l"
```

```
┌─────────────────────────────────┐
│ I should teach you how to use.  │  ← player presses A
└─────────────────────────────────┘
┌─────────────────────────────────┐
│ your RUNNING SHOES.             │  ← ???
└─────────────────────────────────┘
```

The player reads "I should teach you how to use." as a grammatically complete sentence, presses A, and is then confused by a box that starts with a lowercase fragment.

**Right:**
```inc
	.string "I should teach you how to use\l"
	.string "your RUNNING SHOES.\p"
```

```
┌─────────────────────────────────┐
│ I should teach you how to use   │  ← scrolls automatically
└─────────────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ I should teach you how to use   │
│ your RUNNING SHOES.             │  ← player presses A
└─────────────────────────────────┘
```

The sentence completes before the player has to do anything.

---

## Special Characters Inside Strings

The `.string` directive is delimited by plain ASCII double-quotes (`"`). If you want a visible quotation mark to appear in dialogue — for example, to show a character reading from a sign or manual — use the Unicode curly quotes `"` and `"` instead of plain `"`.

**Wrong** (breaks the assembler):
```inc
	.string "She said "hello" to me.$"
```

The assembler sees `"She said "` as a complete string, then `hello" to me.$"` as garbage.

**Right:**
```inc
	.string "She said "hello" to me.$"
```

The curly quotes are mapped to in-game character bytes by `charmap.txt` and are not interpreted as string delimiters.