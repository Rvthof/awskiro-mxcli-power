# Writing MDL Script Files

## Critical Rule: Large Files Must Use fsWrite + fsAppend

`fsWrite` truncates content beyond ~50 lines. MDL scripts are almost always longer than 50 lines. **Always split writes using fsWrite for the first chunk and fsAppend for all subsequent chunks.**

### Required Pattern

```
1. fsWrite  → first ~40 lines (file header + first statement)
2. fsAppend → next ~40 lines
3. fsAppend → next ~40 lines
... repeat until file is complete
```

Never put more than ~40–45 lines in a single `fsWrite` or `fsAppend` call.

### Example: Writing a 120-line script

```
fsWrite  mdlsource/05-action-microflows.mdl  ← lines 1–40
fsAppend mdlsource/05-action-microflows.mdl  ← lines 41–80
fsAppend mdlsource/05-action-microflows.mdl  ← lines 81–120
```

## Never Retry by Creating a New File

If a script fails validation, **fix the existing file** using `strReplace` or by rewriting it with `fsWrite` + `fsAppend`. Do NOT create `script-part2.mdl`, `script-part3.mdl`, `script-fix.mdl`, etc.

One logical script = one file. Extra numbered files are a sign the write strategy failed, not that the MDL was wrong.

## Validation Before Execution

Always validate before executing:

```bash
./mxcli check mdlsource/my-script.mdl -p MyApp.mpr --references
```

If validation fails, fix the **same file** using `strReplace` for targeted edits, then re-validate.

## File Size Guidance

| Script content | Approximate lines | Chunks needed |
|---|---|---|
| 1–2 short microflows | < 80 lines | fsWrite + 1 fsAppend |
| 3–5 microflows | 80–200 lines | fsWrite + 3–4 fsAppend |
| Full task script (6+ microflows) | 200–500 lines | fsWrite + 5–10 fsAppend |

When in doubt, use smaller chunks. An extra `fsAppend` call costs nothing; a truncated file wastes a full retry cycle.

## Script File Organisation

Store all MDL scripts in `mdlsource/`:

```
mdlsource/
├── 01-domain-model.mdl
├── 02-security.mdl
├── 03-microflows.mdl
├── 04-pages.mdl
└── 05-navigation.mdl
```

Use numbered prefixes to indicate execution order when scripts have dependencies.
