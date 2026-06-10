# CLAUDE.md

## Project

translate-book is a Claude Code Skill that translates books (PDF/DOCX/EPUB) from any language into Hungarian using parallel subagents. This is a Hungarianized fork of `deusyu/translate-book`, on GitHub as `Megzo/translate-book`. The target language is fixed to Hungarian (`hu`); the sub-agent translation prompt in SKILL.md is written in Hungarian, the orchestration text stays in English.

## Structure

- `SKILL.md` — Skill definition, the orchestration logic that Claude Code / OpenClaw follows
- `scripts/convert.py` — PDF/DOCX/EPUB → Markdown chunks (via Calibre HTMLZ)
- `scripts/manifest.py` — SHA-256 chunk tracking and merge validation
- `scripts/glossary.py` — Term-consistency glossary; per-chunk term tables injected into sub-agent prompts
- `scripts/chunk_context.py` — Read-only previous/next chunk excerpts injected into sub-agent prompts
- `scripts/meta.py` — Per-chunk sub-agent observation file schema
- `scripts/merge_meta.py` — Batch-boundary merge of sub-agent observations into the canonical glossary
- `scripts/run_state.py` — Selective re-translation planner and run_state.json recorder
- `scripts/merge_and_build.py` — Merge translated chunks → HTML/DOCX/EPUB/PDF
- `scripts/calibre_html_publish.py` — Calibre format conversion wrapper
- `scripts/template.html`, `scripts/template_ebook.html` — HTML templates

## Testing changes

Test with a small PDF to verify the full pipeline:

```bash
python3 scripts/convert.py /path/to/small.pdf
# then run translation via the skill
python3 scripts/merge_and_build.py --temp-dir <name>_temp --title "test"
```

Verify: all output_chunk*.md files exist, manifest validation passes, output formats generate.

## Conventions

- Only `chunk*.md` naming — no `page*` legacy support
- Pipeline output artifacts use the canonical names `book.html`, `book_doc.html`, `book.docx`, `book.epub`, `book.pdf`. Internal scripts and skip/cache logic depend on these names; if title-based filenames are added later they must be optional aliases/copies, not silent replacements
- SKILL.md frontmatter must stay single-line per field (OpenClaw parser requirement)
- Script paths in SKILL.md use `{baseDir}` not hardcoded paths
- Subagent instructions in SKILL.md must be platform-neutral (work on Claude Code, OpenClaw, Codex)
- README.md is the only README (Hungarian); README.zh-CN.md was intentionally removed
- The glossary term-table header emitted by `scripts/glossary.py` (`Forrás | Aliasok | Fordítás`) must stay in sync with the wording referenced in SKILL.md rule #13
- Releases follow `.claude/commands/release.md` — three commands in order: `git push origin main`, `git tag vX.Y.Z && git push --tags`, `npx clawhub@latest publish ./ --version X.Y.Z`. Do not skip the git tag; it's the only version anchor in the repo

## Do not

- Do not reintroduce `page*` file support — it was intentionally removed
- Do not reintroduce other target languages or CJK prompt content (Chinese translation rules, CJK fonts, zh/ja/ko LANG_CONFIG entries) — this fork is Hungarian-target only. The CJK character-range helpers in `scripts/glossary.py` stay: they handle CJK *source* documents
- Do not hardcode `~/.claude/skills/` paths in SKILL.md — use `{baseDir}`
- Do not put platform-specific tool names (Agent, sessions_spawn) in `allowed-tools` as the only option — keep the whitelist cross-platform
- Do not add mtime-based incremental rebuild for HTML/format generation — the current skip logic is intentionally simple (existence check). Metadata/template changes require manual cleanup. This is documented in the README.
