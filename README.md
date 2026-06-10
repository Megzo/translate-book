# Translate Book — magyar fordító skill

Claude Code skill, amely teljes könyveket és dokumentumokat (PDF/DOCX/EPUB) fordít **bármilyen nyelvről magyarra**, párhuzamos subagentekkel.

> Ez a [deusyu/translate-book](https://github.com/deusyu/translate-book) magyarított forkja. Az eredeti projekt tetszőleges célnyelvet támogatott; ez a változat fixen magyar célnyelvre fordít, és az összes nyelvspecifikus (kínai) prompt-tartalom magyarra/angolra lett cserélve. Az eredeti projektet a [claude_translater](https://github.com/wizlijun/claude_translater) inspirálta.

---

## Hogyan működik

```
Bemenet (PDF/DOCX/EPUB)
  │
  ▼
Calibre ebook-convert → HTMLZ → HTML → Markdown
  │
  ▼
Darabolás chunkokra (chunk0001.md, chunk0002.md, ...)
  │  a manifest.json követi a chunkok hash-eit
  ▼
Párhuzamos subagentek (alapértelmezetten 8 egyszerre)
  │  minden subagent: 1 chunk beolvasása → fordítás → output_chunk*.md
  │  batch-ekben, az API rate limitek tiszteletben tartásával
  ▼
Validálás (manifest hash-ellenőrzés, 1:1 forrás↔kimenet megfeleltetés)
  │
  ▼
Összefűzés → Pandoc → HTML (tartalomjegyzékkel) → Calibre → DOCX / EPUB / PDF
```

Minden chunk saját, független subagentet kap friss kontextusablakkal. Ez megakadályozza a kontextus felhalmozódását és a kimenet csonkolódását, ami egyetlen munkamenetben fordított teljes könyvnél elkerülhetetlen lenne.

## Funkciók

- **Párhuzamos subagentek** — batch-enként 8 párhuzamos fordító, mindegyik izolált kontextussal
- **Folytatható + szelektív újrafordítás** — chunk-szintű folytatás; a `run_state.json` követi, hogy glossary-változás után mely chunkokat kell újrafordítani
- **Szomszéd-kontextus** — minden chunk rövid, csak olvasható részleteket lát a szomszédos chunkokból a névmás- és entitásfeloldáshoz
- **Manifest-validálás** — SHA-256 hash-követés akadályozza meg, hogy elavult vagy sérült kimenet kerüljön az összefűzésbe
- **Többformátumú kimenet** — HTML (lebegő tartalomjegyzékkel), DOCX, EPUB, PDF
- **Opcionális kimeneti beállítások** — explicit EPUB-borító, egyedi temp-könyvtár, felhasználóbarát exportnevek
- **Magyar célnyelv** — bármilyen forrásnyelvről magyarra fordít
- **PDF/DOCX/EPUB bemenet** — a konverzió nehezét a Calibre végzi

## Előfeltételek

- **Claude Code CLI** — telepítve és bejelentkezve
- **Calibre** — az `ebook-convert` parancsnak elérhetőnek kell lennie ([letöltés](https://calibre-ebook.com/))
- **Pandoc** — a HTML↔Markdown konverzióhoz ([letöltés](https://pandoc.org/))
- **Python 3** az alábbiakkal:
  - `pypandoc` — kötelező (`pip install pypandoc`)
  - `beautifulsoup4` — opcionális, jobb tartalomjegyzék-generáláshoz (`pip install beautifulsoup4`)

## Gyors kezdés

### 1. A skill telepítése

**A lehetőség: npx**

```bash
npx skills add Megzo/translate-book -a claude-code -g
```

**B lehetőség: Git clone**

```bash
git clone https://github.com/Megzo/translate-book.git ~/.claude/skills/translate-book
```

### 2. Könyv fordítása

A Claude Code-ban írd be:

```
fordítsd le magyarra: /path/to/book.pdf
```

Vagy használd a slash parancsot:

```
/translate-book fordítsd le /path/to/book.pdf
```

A skill automatikusan végigviszi a teljes pipeline-t — konvertálás, darabolás, párhuzamos fordítás, validálás, összefűzés és az összes kimeneti formátum legenerálása.

### 3. A kimenetek helye

Minden fájl a `{book_name}_temp/` könyvtárban van:

| Fájl | Leírás |
|------|--------|
| `output.md` | Összefűzött, lefordított Markdown |
| `book.html` | Webes verzió lebegő tartalomjegyzékkel |
| `book.docx` | Word-dokumentum |
| `book.epub` | E-könyv |
| `book.pdf` | Nyomtatásra kész PDF |

## Teszt-assetek a repóban

- A verziókövetett baseline bemenetek a `tests/baselines/<book-id>/` alatt találhatók.
- A generált teljes-pipeline kimenetek a `tests/.artifacts/` alá kerülnek, és nem szabad commitolni őket.
- Mivel a `scripts/convert.py` a `{book_name}_temp/` könyvtárat az aktuális munkakönyvtár alá írja, a baseline-teszteket a `tests/.artifacts/` könyvtárból futtasd, hogy a generált fájlok ne a repo gyökerébe kerüljenek.

### Teljes-pipeline baseline példa

```bash
mkdir -p tests/.artifacts
cd tests/.artifacts
python3 ../../scripts/convert.py ../baselines/standard-alice/standard-alice.epub
# ezután a fordítás a skillen keresztül fut
python3 ../../scripts/merge_and_build.py --temp-dir standard-alice_temp --title "teszt"
```

## A pipeline részletei

### 1. lépés: Konvertálás

```bash
python3 scripts/convert.py /path/to/book.pdf
```

A Calibre HTMLZ-be konvertálja a bemenetet, ami kicsomagolás után Markdownná alakul, majd ~6000 karakteres chunkokra darabolódik. A `manifest.json` minden forrás-chunk SHA-256 hash-ét rögzíti a későbbi validáláshoz. A célnyelv alapértelmezetten magyar (`--olang hu`).

Alapértelmezetten a munkakönyvtár a `{book_name}_temp/` az aktuális könyvtár alatt. A `--temp-root /path/to/work` kapcsolóval ugyanez a könyvtárnév egy másik szülő alá kerül.

### 1.3. lépés: SUMMARY.md (fordítási brief)

Minden chunkot friss kontextusú subagent fordít, így egyik sem látja a teljes dokumentumot. Ezért a fordítás előtt egy dedikált subagent elkészíti a `<temp_dir>/SUMMARY.md` fájlt: egy rövid (legfeljebb ~1000 szavas), magyar nyelvű fordítási briefet, amely rögzíti, hogy mi a dokumentum, kinek szól, milyen hangnemben és regiszterben íródott (explicit tegezés/magázás döntéssel), melyek a kulcsfogalmai, narratív műnél pedig kik a szereplők és milyen viszonyban állnak. Ez a brief minden subagent promptjába bekerül csak olvasható kontextusként.

- A meglévő `SUMMARY.md` soha nem íródik felül — kézzel szerkeszthető (pl. te döntöd el a tegezés/magázás kérdést); az újrageneráláshoz töröld a fájlt.
- A `SUMMARY.md` csak tájékoztató kontextus: a `run_state.py` nem követi, a szerkesztése nem vált ki újrafordítást (a glossary-szerkesztéssel ellentétben). Ha futás közben változtatsz rajta regiszter-döntést, töröld kézzel az érintett `output_chunk*.md` fájlokat, vagy használj friss temp-könyvtárat.

### 1.5. lépés: Glossary (terminológiai következetesség a chunkok között)

Minden chunkot friss kontextusú subagent fordít, ezért egy 100 chunkos könyvben ugyanaz a tulajdonnév többféleképpen is lefordulhatna. Ezt előzi meg a fordítás előtt felépített glossary:

1. 5 chunk mintavételezése (első, utolsó, 3 egyenletesen elosztott középső).
2. Tulajdonnevek és visszatérő szakkifejezések kigyűjtése; kanonikus magyar fordítások kiválasztása.
3. A `<temp_dir>/glossary.json` megírása (kézzel szerkeszthető, séma lentebb).
4. `python3 scripts/glossary.py count-frequencies <temp_dir>` futtatása a kifejezések gyakoriságának meghatározásához (az ASCII kifejezések szóhatár-figyelő regexszel számolódnak, így a `cat` nem találja meg a `category`-t; a CJK kifejezések substringként; az egykarakteres CJK alakok kizárva; az aliasok a saját kifejezésükhöz számítanak).
5. Minden chunkhoz az orchestrátor lefuttatja a `python3 scripts/glossary.py print-terms-for-chunk <temp_dir> chunkNNNN.md` parancsot, és az így kapott 3 oszlopos (`Forrás | Aliasok | Fordítás`) markdown táblázatot kemény megkötésként beinjektálja a chunk promptjába. A kiválasztás = (kifejezések, amelyek forrása VAGY bármely aliasa előfordul a chunkban) ∪ (a könyv egészében leggyakoribb top-N).

```json
{
  "version": 2,
  "terms": [
    {"id": "Mr. Fox", "source": "Mr. Fox", "target": "Róka úr",
     "category": "person", "aliases": [], "gender": "male",
     "confidence": "medium", "frequency": 12,
     "evidence_refs": [], "notes": ""}
  ],
  "high_frequency_top_n": 20,
  "applied_meta_hashes": {}
}
```

A meglévő v1-es `glossary.json` fájlok az első betöltéskor automatikusan v2-re frissülnek. A v2 tiltja, hogy ugyanaz a felszíni alak (forrás vagy alias) két különböző kifejezéshez tartozzon; ha egy v1-es fájlban poliszém duplikált források vannak, a frissítés egyértelműsítést kérő üzenettel leáll — javítsd kézzel a fájlt, és töltsd be újra.

Futtatások között a `glossary.json` szerkesztésével javíthatod a fordításokat; a meglévő `glossary.json` soha nem íródik felül — a nulláról építéshez töröld a fájlt. A `scripts/run_state.py` rögzíti, hogy az egyes chunkok mely glossary-kifejezéseket használták, így a későbbi glossary-változások csak az érintett chunkokat fordíttatják újra.

### 2. lépés: Fordítás (párhuzamos subagentek)

A skill batch-ekben indítja a subagenteket (alapértelmezés: 8 párhuzamosan). Minden subagent:

1. Beolvas egy forrás-chunkot (pl. `chunk0042.md`)
2. Lefordítja magyarra
3. Per-chunk terminustáblázatot, a `SUMMARY.md` fordítási briefet és rövid, csak olvasható előző/következő részleteket használ
4. Az eredményt az `output_chunk0042.md` fájlba írja
5. Megfigyeléseit az `output_chunk0042.meta.json` fájlba írja a glossary-visszacsatoláshoz

A subagentek indítása előtt a `scripts/run_state.py plan <temp_dir>` eldönti, mely chunkokat kell fordítani, mely meglévő kimeneteknek kell csak állapotrögzítés, és melyek változatlanok. A `--retranslate-untracked` kapcsolót csak akkor használd, ha egy régi temp-könyvtár meglévő kimeneteit akarod a jelenlegi glossary-n átfuttatni. Megszakadt futás után az újrafutás kihagyja a már érvényes kimenettel és aktuális állapottal rendelkező chunkokat. A sikertelen chunkok automatikusan egyszer újrapróbálódnak.

### 3. lépés: Összefűzés és buildelés

```bash
python3 scripts/merge_and_build.py --temp-dir book_temp --title "A lefordított cím"
```

Opcionális kimeneti kapcsolók:

```bash
python3 scripts/merge_and_build.py --temp-dir book_temp --title "A lefordított cím" --cover cover.jpg --export-name "leforditott-cim"
```

A `--cover` explicit képet ad át az EPUB Calibre-lépésnek. A `--export-name` alias-másolatokat készít (pl. `leforditott-cim.epub`), miközben a kanonikus `book.*` pipeline-artefaktumok megmaradnak.

Összefűzés előtt a szkript validálja, hogy:
- Minden forrás-chunkhoz tartozik kimeneti fájl (1:1 megfeleltetés)
- A forrás-chunkok hash-ei egyeznek a manifestben rögzítettekkel (nincs elavult kimenet)
- Egyetlen kimeneti fájl sem üres

Ezután: összefűzés → Pandoc HTML → tartalomjegyzék beszúrása → a Calibre legenerálja a DOCX, EPUB és PDF formátumokat.

**Megjegyzés:** a `{book_name}_temp/` egyetlen fordítási futás munkakönyvtára. Ha megváltoztatod a címet, a szerzőt, a sablont vagy a képeket, használj friss temp-könyvtárat, vagy töröld a meglévő végső artefaktumokat (`output.md`, `book*.html`, `book.docx`, `book.epub`, `book.pdf`) az újrafuttatás előtt.

## Projektstruktúra

| Fájl | Funkció |
|------|---------|
| `SKILL.md` | A Claude Code skill definíciója — a teljes pipeline orchestrációja |
| `scripts/convert.py` | PDF/DOCX/EPUB → Markdown chunkok Calibre HTMLZ-n keresztül |
| `scripts/manifest.py` | Chunk-manifest: SHA-256 követés és merge-validálás |
| `scripts/glossary.py` | Glossary-kezelés: per-chunk terminustáblázatok a következetes terminológiához |
| `scripts/chunk_context.py` | Csak olvasható előző/következő chunk-részletek a subagent-promptokhoz |
| `scripts/meta.py` | Per-chunk subagent-megfigyelési fájl sémája (`output_chunkNNNN.meta.json`) |
| `scripts/merge_meta.py` | Batch-határos merge: subagent-megfigyelések → kanonikus glossary |
| `scripts/run_state.py` | Szelektív újrafordítás-tervező és `run_state.json`-rögzítő |
| `scripts/merge_and_build.py` | Chunkok összefűzése → HTML → DOCX/EPUB/PDF |
| `scripts/calibre_html_publish.py` | Calibre-wrapper a formátumkonverzióhoz |
| `scripts/template.html` | Webes HTML-sablon lebegő tartalomjegyzékkel |
| `scripts/template_ebook.html` | E-könyv HTML-sablon |
| `tests/baselines/` | Verziókövetett baseline könyvbemenetek a teljes-pipeline tesztekhez |
| `tests/.artifacts/` | Git által ignorált teljes-pipeline tesztkimenetek |

## Hibaelhárítás

| Probléma | Megoldás |
|----------|----------|
| `Calibre ebook-convert not found` | Telepítsd a Calibre-t, és gondoskodj róla, hogy az `ebook-convert` a PATH-on legyen |
| `Manifest validation failed` | A forrás-chunkok megváltoztak a darabolás óta — futtasd újra a `convert.py`-t |
| `Missing source chunk` | A forrásfájl törlődött — futtasd újra a `convert.py`-t |
| Hiányos fordítás | Futtasd újra a skillt — onnan folytatja, ahol abbamaradt |
| Cím/sablon/assetek változtak, de a kimenet nem frissült | Töröld a temp-könyvtárból a meglévő `output.md`, `book*.html`, `book.docx`, `book.epub`, `book.pdf` fájlokat, majd futtasd újra a `merge_and_build.py`-t |
| Oldalszám-lábléceket akarsz kiszedni a PDF-ből | Alapértelmezetten a monoton oldalszám-sorozatok (pl. `1, 2, 3, ...`) automatikusan felismerésre és törlésre kerülnek, miközben az olyan kivételek, mint az évszámok (`1984`), fejezetszámok és hivatkozási indexek megmaradnak. Ha a felismerés nem kapja el a te esetedet, add át a `--strip-page-numbers` kapcsolót a `convert.py`-nak, ami agresszívan töröl minden önálló számjegysort. A kapcsoló leáll, ha már létezik cache-elt `input.md` vagy `chunk*.md` — ezeket előbb töröld, hogy a kapcsoló tényleg érvényesüljön. |
| `output.md exists but manifest invalid` | Elavult kimenet — a szkript automatikusan törli és újra összefűz |
| `Glossary upgrade rejected: duplicate source` | A v2 nem engedi, hogy két kifejezés ugyanazon forrás/alias alakon osztozzon. Szerkeszd a `glossary.json`-t az egyértelműsítéshez (pl. nevezd át az egyik forrást `Apple`-ről `Apple (Inc.)`-re), és töltsd be újra. |
| A PDF-generálás meghiúsul | Győződj meg róla, hogy a Calibre PDF-kimenet támogatással van telepítve |

## Tervezési elvek

- **A szkriptek könyvelnek; a szemantikus merge az LLM dolga.** Állapot, sémák, deduplikáció, hashelés, IO — determinisztikus Python. Elnevezés, nyelvtani nem megállapítása, alias-megítélés, konfliktusfeloldás — LLM-hívások.
- **Egyetlen író a megosztott állapothoz.** Csak a fő agent írja a `glossary.json`-t és a `run_state.json`-t; a subagentek per-chunk meta fájlokat írnak. Nem kell zárolás.
- **Konzervatív merge.** Új entitáshoz bizonyíték kell; az alias-összevonáshoz LLM-ítélet, nem csak string-hasonlóság; a nyelvtani nem `unknown`-ról indul, és csak explicit bizonyíték alapján változik; konfliktus esetén a kanonikus értékek nem íródnak felül csendben.
- **Háromrétegű állapot, három külön fájl.** `glossary.json` (kanonikus, a subagentek olvassák), `output_chunkNNNN.meta.json` (nyers per-chunk megfigyelések), `run_state.json` (orchestráció).

## Visszajelzés

Az upstream projekt fejlesztési előzményei és roadmapje a [deusyu/translate-book](https://github.com/deusyu/translate-book) repóban találhatók. Ezzel a magyar forkkal kapcsolatos hibákat a [Megzo/translate-book](https://github.com/Megzo/translate-book/issues) issue-iban jelezd.

## Licenc

[MIT](LICENSE)
