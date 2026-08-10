---
name: en-uz-tech-translate
description: Translates English technical and programming documentation (books, tutorials, docs, READMEs, blog posts, API references, .md/.mdx files) into Uzbek. Preserves Markdown structure and code blocks verbatim, enforces the ş/ç/ğ/õ orthography, and maintains a living glossary at references/glossary.md. Activate when the user asks to translate technical content into Uzbek, says "tarjima qil", or wants to continue translating the next chapter of a book already in progress in the project.
---

# English → Uzbek Technical Translation

Translate English technical and programming documentation (books, tutorials, docs, READMEs, blog posts, API references) into Uzbek, preserving Markdown structure, code blocks, and technical accuracy.

## When to Activate

Activate this skill when the user, inside Zed's assistant panel:

- asks to translate technical/programming content from English to Uzbek
- references a specific source (e.g. "the Rust book", a docs site, or a `.md`/`.mdx` file open in the Zed workspace) and asks for a translation
- says "tarjima qil" or "translate this" while a technical English Markdown file is open in the active Zed tab
- asks to continue translating the next chapter of a book already in progress in the current Zed project

## Orthography Rule (non-negotiable)

Always use the letters **ş, ç, ğ, õ** in Uzbek output — never their digraph equivalents. Concretely:

- `ş` instead of `sh`
- `ç` instead of `ch`
- `ğ` instead of `g'`
- `õ` instead of `o'`, `ö`

Apply this consistently across every sentence, heading, and code comment that gets translated. Do not silently fall back to apostrophe-based or digraph spelling even if the source material or a pasted example uses it.

### Sub-rules

1. **Drop the modifier apostrophe (ʼ) entirely from native/assimilated Uzbek words.** Never write `ʼ`. `eʼlon` → `elon`, `yaʼni` → `yani`, `maʼno` → `mano`.
2. **Disambiguate homonym collisions with a doubled vowel, not with `ʼ`.** If dropping `ʼ` from a word would make it identical to an unrelated existing word, double the vowel that used to carry the `ʼ` instead of restoring the mark: `tana` (body) stays `tana`; `taʼna` (reproach) becomes `taana`. Only do this where a real collision exists — don't double vowels speculatively. Log every such pair the first time it comes up in a translation, in a **Homonimlar** table in `references/glossary.md` (source word, doubled-vowel form, its meaning, and the plain form's meaning), and ask the user for a ruling if it's unclear which of two candidate readings applies.
3. **`h` for Uzbek words, `x` only for foreign words.** Every word that functions as native/assimilated Uzbek vocabulary is spelled with `h`, never `x`. `x` is reserved for words that stay recognizably foreign/loaned (e.g. `sxema`, `taxi`-type borrowings, proper nouns). When a word's status is genuinely unclear (long-assimilated Arabic/Persian loans are the usual gray area), don't silently pick one — flag it and ask the user, then log the ruling in the glossary so it's consistent afterward.
4. **Use a plain single quote (`'`) only at a foreign-word/Uzbek-suffix boundary**, never inside a fully Uzbek word. `rust` + `ning` → `rust'ning`; `ownership` + `ga` → `ownership'ga`. This is a different mark from rule 1's `ʼ` — rule 1 is about dropping an orthographic mark from Uzbek-origin words, this rule is about attaching a Uzbek grammatical suffix to a term that's being kept in its original (usually English) spelling.

## Zed-Specific Workflow

1. **Locate the source file inside the Zed workspace.** Prefer files already open in a Zed tab or present in the project tree over fetching from the web. If the user references an external repo or book not yet in the workspace, ask them to open/add it in Zed, or fetch it and note that it should be added to the project so Zed's file tree and diffing stay useful.
2. **Use Zed's inline diff / multibuffer view for review.** When proposing a translated file, structure the output so it can be reviewed as a diff against the original inside Zed, rather than as a flat paste. Keep original file paths and only change the target directory (e.g. mirror `src/ch03-01-variables.md` into `src-uz/ch03-01-variables.md`).
3. **Maintain a glossary file in the project.** Keep a `references/glossary.md` file inside the Zed project directory as a living log of terminology decisions. Read it before starting a new chapter or file, and append new decisions immediately after making them so they're visible to future sessions and to collaborators opening the same project in Zed.
4. **Respect Zed's project-wide search.** Before introducing a new translated term, check whether it already appears elsewhere in the project (via Zed's project search) to avoid inconsistent terminology across files translated in different sessions.

## Style Defaults (unless the user specifies otherwise)

- **Tone: formal ("siz")** — technical-document register, not "sen"; prefer recommendation phrasing ("...qiling", "...mumkin") over blunt imperatives.
- **Terminology: mixed approach** — well-known, domain-specific terms (ownership, borrow checker, trait, etc.) stay in English; general programming concepts get translated into Uzbek. Record the exact boundary decisions in `references/glossary.md`.

## Translation Rules

1. **Separate structure from prose.** Translate only prose (plain text, comments meant for the reader, headings). Never touch, and always preserve verbatim:
   - code blocks (fenced code, compiler/terminal output)
   - inline code spans
   - link targets (translate the link text only, never the URL)
   - frontmatter, HTML tags, Markdown syntax characters (`#`, `**`, table pipes, etc.)
   - filenames, function/variable names, terminal commands — including when they appear inline in prose
   - Exception: in-code comments may be translated only if the user explicitly asks for it, and even then the code logic itself must never change.
2. **Translate prose for meaning, not word-for-word.** Rebuild sentences so they read naturally in Uzbek; split or restructure long English sentences (especially passive constructions or long chained clauses) rather than mirroring English syntax literally. Never lose technical meaning in the process.
3. **Handle new terms methodically.** When a term isn't yet in the glossary: decide based on the mixed-approach criteria, log the decision to `references/glossary.md` immediately, and ask the user for a ruling only if the choice is genuinely ambiguous — never guess silently on ambiguous cases.
4. **Self-check after each file/chapter.**
   - All code blocks are untouched (diff against the original if possible)
   - Terminology is consistent within the file and with previously translated files in the project
   - The ş/ç/ğ/õ orthography rule was applied consistently
   - No `ʼ` remains anywhere in the output, except where a logged homonym pair required a doubled vowel instead (rule 2)
   - No native/assimilated Uzbek word uses `x`; `x` appears only in words kept as foreign (rule 3)
   - Every `'` in the text sits at a foreign-word/Uzbek-suffix boundary, not inside an Uzbek word (rule 4)
   - Heading hierarchy (`#`, `##`, `###`) matches the source

## Multi-Chapter / Multi-File Projects

- Re-read `references/glossary.md` before starting each new file — it may have been updated in a previous Zed session.
- Periodically revisit earlier translated files to check consistency against the glossary, especially when a term was logged late.
- After each file, give the user a short summary: which new terms were decided, and anything left ambiguous.

## General Notes

- This is technical translation, not creative writing — accuracy comes first, but readability still matters (the audience is meant to read the book, not just extract facts).
- For openly licensed material (e.g. MIT/Apache-2.0 books like the Rust book), translation is normal practice — but verify the license per file and preserve license text/attribution in the translated output.
- If a source file is very long, translate it in sections and report progress after each section rather than trying to fit everything into one response.
