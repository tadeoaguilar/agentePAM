# Context — PAM UUID Agent (session handoff)

## Project

PAM Hotels (Grupo KTRC, Sociedad 3400) — automates traceability of GL expense lines
(FBL3N / `AuxGastos.xlsx`) to supplier CFDIs (`ecommerce.xlsx`) and payment records
(FBL1N / `files/Pagos/*.xlsx`), so finance doesn't have to manually cross-reference
3-4 SAP exports per expense line.

Primary deliverable: **`PAM_UUID_Agent.ipynb`** (32 code+md cells, runs end-to-end,
writes `output/ReporteFinal_3400_202002.xlsx`).
There's also **`PAM_UUID_Agent_v3_AI.ipynb`** — identical through Section 4, then adds:
- rapidfuzz (WRatio) instead of `difflib.SequenceMatcher`
- Section 4.5: Claude Haiku (`claude-haiku-4-5-20251001`) as a judge for `NO_RESUELTO` rows
  (limit=100 for demo, ~$0.0002/record)

Background docs (Spanish, detailed column maps): `PAM_UUID_Agent_Architecture.md`,
`PAM_UUID_Agent_CHANGES_v2.md`, `Resumen_Ejecutivo_Agente_Trazabilidad.md`.

## Data sources (`files/`)

- `AuxGastos.xlsx` (sheet `Febrero_2020_KTRC`) — GL ledger, ~2,400 rows/month, only ~16-19% have UUID
- `Compensacion.xlsx` — full GL extract (171,397 rows), bridges expense doc ↔ payment doc via
  the transit/clearing account (`doc_compensacion`). Column P = `Folio Fisc` (UUID, often
  populated on the *payment* doc even when the expense doc's own UUID is blank)
- `Pagos/*.xlsx` — FBL1N payment ledger, 11 society files. **3400.xlsx = 123-col full format**
  (`skiprows=[1]`); all others = 23-col compact format (matched by header name)
- `Acreedor.xlsx` — vendor master (~24,600 records)
- `ecommerce.xlsx` — CFDI receipts, 116,335 rows / 108,599 vigentes. UUID is **lowercase** —
  normalize with `clean_uuid()` (upper-cases)

## 7-step matching flow

1. Load & normalize AuxGastos, FBL1N (Pagos), Acreedor, Ecommerce, Compensacion
2. Enrich AuxGastos with RFC from Acreedor master
3. **S1 UUID_DIRECTO**: AuxGastos.uuid → FBL1N.uuid (exact) — 378 matches (18.8%)
4. **S2 DOCUMENTO_CRUCE**: nro_documento join FBL1N (exact) — 100 matches (5.0%)
5. **S5 COMPENSACION_BRIDGE** (Cells 25-27, runs before S3/S4 to preserve exact fiscal data):
   - **S5-A** (3-hop): `nro_documento → Compensacion.doc_compensacion → reverse lookup
     (sibling docs cleared by same comp doc) → FBL1N.nro_documento → uuid_fbl` —
     `COMPENSACION_EXACTO`, 167 matches
   - **S5-B**: `nro_documento → Compensacion.uuid → FBL1N.uuid` — `COMPENSACION_UUID`, 0 matches
   - **S5-C**: `COMPENSACION_FOLIO`, 2 matches
   - Total S5: 169 (8.4%)
   - **Paso 6A-bis** (secondary ecommerce enrichment): uses `uuid_fbl` from the S5 hop to
     look up `ecommerce.xlsx` and fill RFC/Subtotal/IVA/Total/Concepto/UUID — 80 records
     enriched
6. **S3 FUZZY_CON_ACREEDOR**: same vendor + date ±5d + text similarity — 10 matches (0.5%)
7. **S4 FUZZY_SIN_ACREEDOR**: date ±3d + amount ±1% + token overlap — 1,242 matches (61.6%)

Remaining `NO_RESUELTO`: 117 rows (5.8%) — mostly WA/WE document classes (material
movements) which never appear in FBL1N; would need MM data to resolve. v3_AI's Claude
judge (Section 4.5) targets these.

**Current overall resolution: 94.2%** (2,016 active rows; 388 ANULADO excluded).

## Recent fix (this session)

**Bug:** `Confianza Ecommerce` showed `SIN_MATCH_ECOMMERCE` for the 80 rows enriched by
Paso 6A-bis (S5 secondary ecommerce lookup), even though their CFDI fields (RFC,
Subtotal, IVA, Total, UUID Pos.) were correctly populated.

**Root cause:** Cell 19 does `df_aux['confianza_eco'].fillna('SIN_MATCH_ECOMMERCE')` over
*all* rows before the S5 split. In Cell 27, `combine_first` only fills NaN, so it never
overwrote that stale value with the intended `'EXACTO_UUID'`.

**Fix:** Added one line in Cell 27 of both notebooks, right after the `_enriched_mask`
backfill:
```python
df_s5.loc[_enriched_mask, 'confianza_eco'] = 'EXACTO_UUID'
```
Then re-ran `PAM_UUID_Agent.ipynb` end-to-end to regenerate
`output/ReporteFinal_3400_202002.xlsx`. Verified: row 63 (doc 100015466, comp.
880006999) now shows `Confianza Ecommerce = EXACTO_UUID`,
`UUID Pos. = 9F8622C2-C8C8-478B-8E04-0169CF45D569`. 0 mislabeled rows remain. All
Confianza/Estrategia counts and the 94.2% resolution rate are unchanged — only the
label for those 80 rows changed.

**Uncommitted at end of session** (`git status`): `PAM_UUID_Agent.ipynb`,
`PAM_UUID_Agent_v3_AI.ipynb`, `output/ReporteFinal_3400_202002.xlsx`.

## How to (re-)run the notebook

The `.venv` does **not** have `nbconvert`/`nbformat`/`papermill`. To execute end-to-end
without them, extract code cells to a script and run it:
```python
import json
nb = json.load(open('PAM_UUID_Agent.ipynb'))
src = []
for c in nb['cells']:
    if c['cell_type'] == 'code':
        src.append(''.join(c['source']))
open('/tmp/run_notebook.py', 'w').write('\n'.join(src))
```
Then comment out the `%pip install ...` magic line (cell 1) and run
`.venv/bin/python /tmp/run_notebook.py`. Takes ~2 minutes (S4 fuzzy matching over ~1,356
rows is the slow part). Writes `output/ReporteFinal_3400_202002.xlsx` (3 sheets:
Resultado, Pendientes, Métricas).

## Open items / ideas for next session

- 117 `NO_RESUELTO` rows remain — checked whether any have a sibling `Folio Fisc` in their
  Compensacion group (the same bridge pattern as doc 100015466): **none do**, so S5
  doesn't currently help them further.
- Consider running `PAM_UUID_Agent_v3_AI.ipynb`'s Claude-judge step (Section 4.5) on the
  117 `NO_RESUELTO` rows to push resolution past 94.2%.
- `output/ReporteFinal_3400_202002.xlsx` is the only output file (both notebooks write to
  the same path) — re-running v3_AI would overwrite it with the AI-judged version.
