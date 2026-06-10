---
name: translate-book
description: Translate books (PDF/DOCX/EPUB) from any language into Hungarian using parallel sub-agents. Converts input -> Markdown chunks -> translated chunks -> HTML/DOCX/EPUB/PDF.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent, AskUserQuestion
metadata: {"openclaw":{"requires":{"bins":["python3","pandoc","ebook-convert"],"anyBins":["calibre","ebook-convert"]},"homepage":"https://github.com/Megzo/translate-book"}}
---

# Book Translation Skill

You are a book translation assistant. You translate entire books from any language into Hungarian by orchestrating a multi-step pipeline.

## Workflow

### 1. Collect Parameters

Determine the following from the user's message:
- **file_path**: Path to the input file (PDF, DOCX, or EPUB) — REQUIRED
- **concurrency**: Number of parallel sub-agents per batch (default: `8`)
- **temp_root**: Optional directory under which `{filename}_temp/` should be created
- **epub_cover**: Optional explicit cover image path for EPUB output
- **export_name**: Optional filename stem for user-facing output aliases
- **custom_instructions**: Any additional translation instructions from the user (optional)

If the file path is not provided, ask the user.

### 2. Preprocess — Convert to Markdown Chunks

Run the conversion script to produce chunks:

The target language is always Hungarian (`hu`, the script default):

```bash
python3 {baseDir}/scripts/convert.py "<file_path>"
```

If the user provided `temp_root`, add `--temp-root "<temp_root>"`. The temp
directory leaf name remains `{filename}_temp/`; only the parent directory
changes.

This creates a `{filename}_temp/` directory containing:
- `input.html`, `input.md` — intermediate files
- `chunk0001.md`, `chunk0002.md`, ... — source chunks for translation
- `manifest.json` — chunk manifest for tracking and validation
- `config.txt` — pipeline configuration with metadata

### 3. Discover Source Chunks

Use Glob to find all source chunks:

```
Glob: {filename}_temp/chunk*.md
```

Exclude `output_chunk*.md` from the source list. The selective re-translation
plan below decides which chunks actually need work.

### 3.3. Generate Document Summary (translation brief)

A separate sub-agent translates each chunk with a fresh context, so no sub-agent ever sees the whole document. `SUMMARY.md` is a short Hungarian-language translation brief injected into every sub-agent prompt, so every chunk is translated with the same understanding of what the document is, who it is for, and what register to use.

If `<temp_dir>/SUMMARY.md` already exists, skip this step — re-running the skill must not overwrite a hand-edited summary. To force a rebuild, delete the file.

Otherwise spawn ONE sub-agent (same mechanism as Step 4) with this task:

> Read `<temp_dir>/input.md` and write `<temp_dir>/SUMMARY.md`: a translation brief written **in Hungarian**, at most ~1000 words, with exactly these sections:
>
> - **Mi ez a dokumentum** — műfaj/típus, téma, szerkezet néhány mondatban
> - **Célközönség** — kinek szól
> - **Hangnem és regiszter** — hangvétel, formalitás, és egy explicit tegezés/magázás döntés, amit a fordításnak követnie kell
> - **Kulcsfogalmak és témák** — a fő fogalmak, amelyeket a fordítónak értenie kell
> - **Szereplők és viszonyaik** — csak narratív műveknél: főszereplők, viszonyaik, beszédstílusuk; technikai dokumentumnál hagyd el ezt a szakaszt
>
> Output only the brief itself — no commentary, no translation of the source text.

If `input.md` is too large to fit in one sub-agent context (roughly > 400 KB), instruct the sub-agent to read a sample instead — `chunk0001.md`, the last chunk, and 5 evenly-spaced middle chunks — and to note at the top of `SUMMARY.md` that it was built from samples.

`SUMMARY.md` is advisory context only. It is NOT tracked by `run_state.py`, and editing it does not trigger re-translation of existing outputs (unlike glossary edits). If a register decision changes after a partial run (e.g. tegezés → magázás), delete the affected `output_chunk*.md` files or use a fresh temp dir.

### 3.5. Build Glossary (term consistency)

A separate sub-agent translates each chunk with a fresh context. Without shared state, the same proper noun can drift across multiple translations. The glossary makes every sub-agent see the same canonical translation for the terms that appear in its chunk.

If `<temp_dir>/glossary.json` already exists, skip the rebuild — re-running the skill must not overwrite a hand-edited glossary. To force a rebuild, delete the file.

Otherwise:

1. **Read the brief**: if `<temp_dir>/SUMMARY.md` exists, read it first — it informs category and target choices below.
2. **Sample chunks**: read `chunk0001.md`, the last chunk, and 3 evenly-spaced middle chunks. If `chunk_count < 5`, sample all of them.
3. **Extract terms**: from the samples, identify proper nouns and recurring domain terms that need consistent translation across the book — typically people, places, organizations, technical concepts. Translate each into Hungarian (keep the source form as target when Hungarian conventionally keeps it, e.g. most place and product names). Skip generic vocabulary that any translator would render the same way.
4. **Write `glossary.json`** in the temp dir, matching this v2 schema:

   ```json
   {
     "version": 2,
     "terms": [
       {"id": "Mr. Fox", "source": "Mr. Fox", "target": "Róka úr",
        "category": "person", "aliases": [], "gender": "male",
        "confidence": "medium", "frequency": 0,
        "evidence_refs": [], "notes": ""}
     ],
     "high_frequency_top_n": 20,
     "applied_meta_hashes": {}
   }
   ```

   Existing v1 `glossary.json` files are auto-upgraded to v2 on first load. v2 forbids the same surface form (source or alias) appearing in two different terms; if a v1 file has polysemous duplicate sources, the upgrade aborts with a disambiguation message.

5. **Count frequencies** by running:

   ```bash
   python3 {baseDir}/scripts/glossary.py count-frequencies "<temp_dir>"
   ```

   This scans every `chunk*.md` (excluding `output_chunk*.md`), updates each term's `frequency` field, and writes back atomically.

The glossary is hand-editable. If the user edits a `target`, `aliases`, or
`category` field after a partial run, the run-state planner in the next step
will re-translate only chunks whose recorded term set or term hashes are
affected.

### 3.7. Plan Selective Re-translation

Run:

```bash
python3 {baseDir}/scripts/run_state.py plan "<temp_dir>"
```

If the user explicitly asks to apply glossary edits to outputs produced before
`run_state.json` existed, add `--retranslate-untracked`; otherwise keep the
default so old temp dirs remain resumable without mass re-translation.

Capture stdout JSON:
- `translation_chunk_ids` — chunks to translate in this run.
- `record_only_chunk_ids` — existing valid outputs that need `run_state.json`
  records but do not need translation.
- `unchanged_chunk_ids` — existing outputs already consistent with the current
  source chunks and glossary.

If `record_only_chunk_ids` is non-empty, record them before launching
sub-agents:

```bash
python3 {baseDir}/scripts/run_state.py record "<temp_dir>" chunk0001 chunk0002 ...
```

Use `translation_chunk_ids` as the work queue for Step 4. If it is empty, skip
to Step 5.

### 4. Parallel Translation with Sub-Agents

**Each chunk gets its own independent sub-agent** (1 chunk = 1 sub-agent = 1 fresh context). This prevents context accumulation and output truncation.

Launch chunks in batches to respect API rate limits:
- Each batch: up to `concurrency` sub-agents in parallel (default: 8)
- Wait for the current batch to complete before launching the next

**Spawn each sub-agent with the following task.** Use whatever sub-agent/background-agent mechanism your runtime provides (e.g. the Agent tool, sessions_spawn, or equivalent).

The output file is `output_` prefixed to the source filename: `chunk0001.md` → `output_chunk0001.md`.

> Translate the file `<temp_dir>/chunk<NNNN>.md` to Hungarian and write the result to `<temp_dir>/output_chunk<NNNN>.md`. Follow the translation rules below. Output only the translated content — no commentary.

Each sub-agent receives:
- The single chunk file it is responsible for
- The temp directory path
- The translation prompt (see below)
- A per-chunk term table (see "Term table assembly" below)
- The document summary brief (see "Summary assembly" below)
- Read-only neighboring chunk excerpts (see "Neighbor context assembly" below)
- Any custom instructions

**Term table assembly** — before spawning a sub-agent, run:

```bash
python3 {baseDir}/scripts/glossary.py print-terms-for-chunk "<temp_dir>" "chunk<NNNN>.md"
```

Capture stdout. The CLI emits a 3-column markdown table (`Forrás | Aliasok | Fordítás`) of every term that either appears in this chunk (by source OR any alias) OR is in the top-N most-frequent terms book-wide. Inject the table as `{TERM_TABLE}` in rule #13 of the translation prompt. **If stdout is empty (no glossary, or no relevant terms), omit rule #13 from this chunk's prompt entirely** — do not leave a dangling `{TERM_TABLE}` placeholder.

**Summary assembly** — read `<temp_dir>/SUMMARY.md` once; its content is identical for every chunk. Inject it as `{DOCUMENT_SUMMARY}` in the translation prompt below. If the file does not exist or is empty, omit the document-summary block entirely — do not leave a dangling `{DOCUMENT_SUMMARY}` placeholder.

**Neighbor context assembly** — before spawning a sub-agent, run:

```bash
python3 {baseDir}/scripts/chunk_context.py "<temp_dir>" "chunk<NNNN>.md"
```

Capture stdout. The CLI emits prompt-ready read-only excerpts: the last ~300
characters of the previous chunk and the first ~300 characters of the next
chunk when those files exist. Inject this block as `{NEIGHBOR_CONTEXT}`. If
stdout is empty, omit the neighbor-context block entirely. The sub-agent must
not translate neighboring excerpts or copy them into the output; they are only
for pronoun, gender, and entity-resolution context.

**Each sub-agent's task**:
1. Read the source chunk file (e.g. `chunk0001.md`)
2. Translate the content following the translation rules below
3. Write the translated content to `output_chunk0001.md`
4. Write observations to `output_chunk0001.meta.json` matching the schema below. **Non-blocking** — leave fields empty if unsure; do not invent entities. Always emit the file (even if all arrays are empty), because its presence + content hash is how the main agent tracks whether feedback was already merged.

**Sub-agent meta schema** (`output_chunk<NNNN>.meta.json`):

```json
{
  "schema_version": 1,
  "new_entities": [
    {"source": "Mrs. Badger", "target_proposal": "Borzné", "category": "person",
     "evidence": "<≤200-char quote from the chunk>"}
  ],
  "alias_hypotheses": [
    {"variant": "Mrs. B.", "may_be_alias_of_source": "Mrs. Badger",
     "evidence": "<≤200-char quote>"}
  ],
  "attribute_hypotheses": [
    {"entity_source": "Mr. Fox", "attribute": "gender", "value": "male",
     "confidence": "high", "evidence": "<≤200-char quote>"}
  ],
  "used_term_sources": ["Mr. Fox", "Manhattan"],
  "conflicts": [
    {"entity_source": "Mr. Fox", "field": "target", "injected": "Róka úr",
     "observed_better": "Fox úr", "evidence": "<≤200-char quote>"}
  ]
}
```

**Do NOT include a `chunk_id` field** — chunk identity is derived from the filename. Putting it in the payload creates a hallucination hole and validation will reject the file.

The meta file is read by the main agent later and merged into `glossary.json` (see `merge_meta.py`). Sub-agents should fill the schema honestly: cite real quotes from the chunk, never invent entities to "look productive". An empty meta is a perfectly valid output.

**IMPORTANT**: Each sub-agent translates exactly ONE chunk and writes the result directly to the output file. No START/END markers needed.

#### Translation Prompt for Sub-Agents

Include this translation prompt in each sub-agent's instructions (the prompt itself is in Hungarian — the target language):

---

Fordítsd le a markdown fájlt magyarra.
FONTOS KÖVETELMÉNYEK:
1. Szigorúan őrizd meg a Markdown formátumot, beleértve a címsorokat, linkeket és képhivatkozásokat.
2. Csak a szöveges tartalmat fordítsd le; minden Markdown szintaxist és fájlnevet hagyj változatlanul.
3. Töröld az üres linkeket és a fölösleges karaktereket, például a sorvégi '\\' jeleket. Az oldalszámokat a convert.py már korábban eltávolította — önálló számsorokat NE törölj (lehetnek évszámok, pl. 1984, fejezetszámok vagy hivatkozási számok, vagyis valódi tartalom).
4. A fordítás legyen formailag és tartalmilag pontos, természetes és gördülékeny magyar szöveg.
5. Csak a lefordított szöveget add ki — semmilyen magyarázat, megjegyzés, kommentár vagy párbeszéd ne kerüljön az outputba.
6. Fogalmazz világosan és tömören, kerüld a túlbonyolított mondatszerkezeteket. Szigorúan sorrendben fordíts, semmit ne hagyj ki.
7. Minden képhivatkozást kötelező megőrizni:
   - Minden ![alt](útvonal) formátumú képhivatkozást teljes egészében meg kell tartani.
   - A képek fájlnevét és útvonalát ne módosítsd (pl. media/image-001.png).
   - A kép alt szövege lefordítható, de a képhivatkozás szerkezetének érintetlennek kell maradnia.
   - Semmilyen képpel kapcsolatos tartalmat ne törölj, ne szűrj ki és ne hagyj figyelmen kívül.
   - Példa képhivatkozásra: ![Figure 1: Data Flow](media/image-001.png) -> ![1. ábra: Adatfolyam](media/image-001.png)
   - **A nyers HTML tagek (pl. `<img alt="..." />`, `<a title="...">`) kötelezően érvényesek maradjanak**: amikor az `alt`, `title` és hasonló attribútumértékek belső szövegét fordítod, az alábbi karakterek tönkretennék a HTML szerkezetet, ezért biztonságos formára kell cserélni őket (ez **kizárólag a nyers HTML tagek attribútumértékeinek belsejére** vonatkozik; a normál Markdown szövegben, kódblokkokban és URL-ekben ne escape-elj):

     | Karakter | Veszély az attribútumértéken belül | Csere |
     |----------|-----------------------------------|-------|
     | `"` | lezárja az `attr="..."` értéket | magyar idézőjelek (`„` `”`) vagy `&quot;` |
     | `'` | lezárja az `attr='...'` értéket | magyar belső idézőjelek (`»` `«`) vagy `&#39;` |
     | `<` | új tagként értelmeződik | `&lt;` |
     | `>` | tag lezárásaként értelmeződik | `&gt;` |
     | `&` | entitás kezdeteként értelmeződik (kivéve ha már `&xxx;`) | `&amp;` |

     A `src`, `href` és más szerkezeti attribútumok értékét ne módosítsd; csak a látható szöveges attribútumokat (`alt`, `title`) fordítsd.

     - Hibás példa: `alt="Alice egy "Igyál meg!" feliratú üveget tart"` ← a belső `"` széttöri a külső alt attribútumot
     - Helyes példa: `alt="Alice egy „Igyál meg!” feliratú üveget tart"` vagy `alt="Alice egy &quot;Igyál meg!&quot; feliratú üveget tart"`
8. Ismerd fel okosan a többszintű címsorokat, és az alábbi szabályok szerint add hozzá a markdown jelölést:
   - Főcím (könyvcím, fejezetcím stb.): # jelölés
   - Első szintű cím (nagyobb szakaszcím): ## jelölés
   - Második szintű cím (alszakaszcím): ### jelölés
   - Harmadik szintű cím (alcím): #### jelölés
   - Negyedik és mélyebb szintű címek: ##### jelölés
9. Címsor-felismerési szabályok:
   - Önálló sorban álló, rövidebb szöveg (általában 50 karakternél rövidebb)
   - Összefoglaló vagy áttekintő jellegű mondat
   - A dokumentum szerkezetében elválasztó, tagoló szerepet betöltő szöveg
   - Feltűnően eltérő betűméretű vagy különleges formázású szöveg
   - Számozással kezdődő fejezetszöveg (pl. „1.1 Áttekintés", „Harmadik fejezet")
10. Címsorszint megállapítása:
    - A szint a szövegkörnyezet és a tartalom súlya alapján döntendő el
    - A fejezetszintű címek általában magas szintűek (# vagy ##)
    - A szakasz- és alszakaszcímek fokozatosan lejjebb kerülnek (### #### #####)
    - Egy dokumentumon belül a címsorszintek legyenek következetesek
11. Figyelem:
    - Ne adj hozzá fölöslegesen címsorjelölést — csak valódi címekhez
    - Normál bekezdésekhez soha ne adj címsorjelölést
    - Ha az eredeti szövegben már van markdown címsorjelölés, őrizd meg annak hierarchiáját
12. {CUSTOM_INSTRUCTIONS if provided}
13. Terminológiai következetesség: az alábbi kifejezéseket kötelező pontosan a megadott fordítással használni, ne térj el tőlük. Ha a táblázat „Forrás" oszlopában **vagy az „Aliasok" oszlopában** szereplő bármely alak előfordul a szövegben, mindig a „Fordítás" oszlop szerinti alakra kell fordítani.

{TERM_TABLE}

Dokumentum-összefoglaló (csak olvasásra — ne fordítsd le, ne másold az outputba; kizárólag a hangnem, a regiszter, a tegezés/magázás és a fogalmi kontextus eldöntéséhez használd; ha üres, hagyd figyelmen kívül):

{DOCUMENT_SUMMARY}

Szomszédos chunk-részletek (csak olvasásra — ne fordítsd le, ne másold az outputba; kizárólag a névmások, nyelvtani nemek, aliasok és chunkokon átívelő utalások feloldásához használd; ha üres, hagyd figyelmen kívül):

{NEIGHBOR_CONTEXT}

A markdown fájl tartalma:

---

### 4.5. Merge Sub-Agent Meta Into Glossary (after each batch)

Each sub-agent emitted an `output_chunk<NNNN>.meta.json` alongside its translated chunk. After every batch completes, first record the completed chunk outputs in `run_state.json` while the glossary is still the one used for that batch, then merge observations into the canonical glossary so subsequent batches see an enriched glossary.

1. Record successfully translated chunks from this batch before mutating the glossary:

   ```bash
   python3 {baseDir}/scripts/run_state.py record "<temp_dir>" chunk0001 chunk0002 ...
   ```

   If this fails, fix the missing/empty output or state error before continuing.

2. Run prepare-merge:

   ```bash
   python3 {baseDir}/scripts/merge_meta.py prepare-merge "<temp_dir>"
   ```

   Capture stdout JSON. It contains four arrays:
   - `auto_apply` — new entities with no glossary collision and unanimous (target, category) across all proposing chunks.
   - `decisions_needed` — items requiring main-agent judgment. Each has `id`, `kind`, an `options` array, and the data needed to pick. Kinds:
     - `alias` — `{variant, candidate_source, evidence}`. Choices: `yes_alias` / `no_separate_entity` / `skip`.
     - `conflict` — `{entity_source, field, current, proposed, evidence}`. Choices: `keep_current` / `accept_proposed` / `record_in_notes`.
     - `new_entity_existing_alias` — sub-agents propose `proposed_source` as a new entity, but it's already someone's alias. `{proposed_source, currently_alias_of, promoted_variants: [{target_proposal, category, evidence, evidence_chunks}, ...]}`. Choices: one `use_variant_N` per distinct (target, category) promotion variant (promote `proposed_source` to standalone with that target+category, removing it from the host's aliases) / `keep_as_alias` / `skip`.
     - `existing_entity_conflict` — sub-agents proposed a (target, category) for `entity_source` that differs from the canonical. Multiple distinct differing proposals all get exposed. `{entity_source, current_target, current_category, proposed_variants: [{target_proposal, category, evidence, evidence_chunks}, ...]}`. Choices: `keep_current` / one `use_variant_N` per competing proposal (overwrites both target AND category, stamps the prior values into notes) / `record_in_notes` (canonical unchanged; every proposed variant gets logged to notes).
     - `alias_or_new_entity` — `variant` has multiple competing options that can't all coexist under v2's surface-form uniqueness rule. Triggered when (a) `variant` was proposed both as a new standalone entity AND as an alias of one or more candidates, OR (b) `variant` was proposed as an alias of two or more different candidates with no standalone competitor. `{variant, alias_candidates: [{candidate_source, evidence, evidence_chunks}, ...], standalone_variants: [{target_proposal, category, evidence, evidence_chunks}, ...]}`. Choices: one `use_alias_N` per candidate (attach as alias of that candidate), one `use_standalone_N` per competing standalone proposal (add as standalone with that target+category), or `skip`.
     - `conflicting_new_entity_proposals` — `{source, variants: [{target_proposal, category, evidence, evidence_chunks}, ...]}`. Choices: `use_variant_0`, `use_variant_1`, ..., `skip`.
   - `consumed_chunk_ids` — every meta file scanned this round (regardless of whether it produced a finding). These hashes get recorded in `applied_meta_hashes` on apply.
   - `malformed_meta_chunk_ids` — meta files that failed validation. Quarantined: not consumed, not crashing the run. Surface them in your batch progress.

3. **If `consumed_chunk_ids` is empty** → nothing was scanned; skip to Step 5.

4. **If `consumed_chunk_ids` is non-empty but both `auto_apply` and `decisions_needed` are empty** → still pipe `{"auto_apply": [], "decisions": [], "consumed_chunk_ids": [...]}` into `apply-merge` so the hashes get recorded. **Skipping this is the bug** — no-op metas would re-scan forever otherwise.

5. **Otherwise, resolve each decision**:
   - Read its evidence quotes inline.
   - Pick one option from its `options` array.
   - Build a `decisions` entry that round-trips the original decision plus your choice. The entry MUST include the original `kind` and (for `conflicting_new_entity_proposals`) the `variants` array, so apply-merge can validate and act:

     ```json
     {"id": "d1", "kind": "alias", "variant": "Taig", "candidate_source": "Tai", "choice": "yes_alias"}
     ```

6. Pipe the decisions JSON into apply-merge:

   ```bash
   echo '{"auto_apply": [...], "decisions": [...], "consumed_chunk_ids": [...]}' \
     | python3 {baseDir}/scripts/merge_meta.py apply-merge "<temp_dir>"
   ```

   Surface the summary JSON (`auto_applied`, `decisions_resolved`, `consumed_chunks`, `errors`) in your batch progress message.

   **apply-merge is transactional.** If any decision is malformed (wrong choice for kind, missing fields, references a non-existent entity), the entire batch aborts with a non-zero exit and stderr details — no glossary mutation, no hashes recorded. On non-zero exit, fix the offending decision and re-pipe; `prepare-merge` will surface the same proposals because nothing was consumed.

   **Decision order in the input list is not significant.** `apply-merge` internally dispatches entity-creating decisions before alias-attaching ones, so `yes_alias` decisions whose candidate is created by another decision in the same batch (a `use_standalone_N`, `use_variant_N`, or `promote_to_separate_entity`) succeed regardless of the order you pass them in. Alias chains (e.g. `Taighi → Taig` where `Taig → Tai` is also a pending alias decision) resolve via a fixed-point loop within the alias-attacher pass — you don't need to topo-sort or sequence chained aliases manually.

On a fresh run after a previous interrupted batch, `prepare-merge` will pick up any meta files left behind. Don't manually delete them.

### 5. Verify Completeness and Retry

After all batches complete, use Glob to check that every source chunk has a corresponding output file.

If any are missing, retry them — each missing chunk as its own sub-agent. Maximum 2 attempts per chunk (initial + 1 retry).

Also read `manifest.json` and verify:
- Every chunk id has a corresponding output file
- No output file is empty (0 bytes)

Then run the meta-merge observability snapshot:

```bash
python3 {baseDir}/scripts/merge_meta.py status "<temp_dir>"
```

Also run the selective re-translation state snapshot:

```bash
python3 {baseDir}/scripts/run_state.py status "<temp_dir>"
```

Surface a one-line summary in the verification report:

> Translated chunks: 50 • Meta files: 48 found / 47 consumed • Malformed: 1 (chunk0099 — see stderr) • Chunks missing meta: chunk0017, chunk0042

Severity rules (none of these fail the run — meta is non-blocking):

- `unmerged_meta_files > 0` after Step 4.5 ran → bug, flag prominently. Resume should have caught this.
- `malformed_meta_files > 0` → sub-agent emitted invalid meta; print chunk_ids and a "fix the file by hand and re-run if you want this chunk's feedback merged" note.
- `meta_files_found < translated_chunks` → sub-agent-compliance issue (some chunks didn't emit meta at all). Print missing chunk_ids.

Report any chunks that failed translation after retry.

### 6. Translate Book Title

Read `config.txt` from the temp directory to get the `original_title` field.

Translate the title to Hungarian. Follow Hungarian title conventions: only the first word and proper nouns are capitalized (e.g. "A gyűrűk ura", not "A Gyűrűk Ura"). Do not add quotation marks or other wrapping around the title.

### 7. Post-process — Merge and Build

Run the build script with the translated title:

```bash
python3 {baseDir}/scripts/merge_and_build.py --temp-dir "<temp_dir>" --title "<translated_title>" --cleanup
```

If the user provided `epub_cover`, add `--cover "<epub_cover>"`. If the user
provided `export_name`, add `--export-name "<export_name>"`.

The `--cleanup` flag removes intermediate files (chunks, input.html, etc.) after a fully successful build. If the user asked to keep intermediates, omit `--cleanup`.

The script reads `output_lang` from `config.txt` automatically. Optional overrides: `--lang`, `--author`.

This produces in the temp directory:
- `output.md` — merged translated markdown
- `book.html` — web version with floating TOC
- `book_doc.html` — ebook version
- `book.docx`, `book.epub`, `book.pdf` — format conversions (requires Calibre)

### 8. Report Results

Tell the user:
- Where the output files are located
- How many chunks were translated
- The translated title
- List generated output files with sizes
- Any format generation failures
