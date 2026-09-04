# NSG-IDS — Revision Runbook (post IEEE ANTS 2026 rejection)

**Written:** 2026-09-03 · **Target:** run all seeds on the CUDA machine
**Scope of changes:** `NSG-IDS.ipynb` and `run_seeds.sh` only. **`NSG-IDS.tex` is byte-identical to your original** — no paper edits were kept.

---

## 0. TL;DR — what to do on the CUDA box

```bash
# 1. Copy NSG-IDS.ipynb + run_seeds.sh across. Fix the three dataset paths
#    in the CFG cell (cell 2) to match that machine.

# 2. SMOKE TEST FIRST — none of the new code has ever executed.
#    In the CFG cell set:  SMOKE_TEST = True
NSG_SEED=42 jupyter nbconvert --to notebook --execute \
    --ExecutePreprocessor.timeout=-1 \
    --output runs/smoke_seed42.ipynb NSG-IDS.ipynb
#    Check the numbers in §4 below, then set SMOKE_TEST = False.

# 3. Seed 42 alone, to compare against the published results.
./run_seeds.sh 42

# 4. If it looks right, the remaining four.
./run_seeds.sh 43 44 45 46

# 5. Open NSG-IDS.ipynb, run the §14 aggregator cell (cell 47)
#    -> mean ± std LaTeX rows + paired t-tests.
```

> **Do not skip step 2.** The NetFlow mapper, the two-pass stratified reader, the
> seed harness, per-class metrics, fused SMOTE and the `no_src` ablation are all
> **statically verified only** — they have never run against data. This Mac has no
> numpy/pandas/torch, so nothing could be executed here. Losing 9 h to a typo in
> hour eight is the failure mode a smoke test prevents.

---

## 1. Why the paper was rejected

Scores were all 3s with one 2 — middle of the pack, no advocate, at ~35 % acceptance.

| Reviewer | Novelty | Technical | Relevance | Presentation |
|---|---|---|---|---|
| R1 | 3 | 3 | 4 | 3 |
| R2 | 3 | 3 | 4 | 4 |
| R3 | 3 | **2** | 3 | 3 |
| R4 | 3 | 4 | 3 | 3 |

**All four reviewers raised the single-seed issue.** That is the one blocker they
converged on, and it is the cheapest to fix.

**Lesson worth internalising:** R2's entire "weak aspects" list is your own
Limitations paragraph reordered, and R3 quotes your own hedges on coverage and
attention back at you. An honest limitations section became the rejection. Close
what you can cheaply close; only confess what genuinely needs another paper.

---

## 2. Datasets — all three verified present

| Dataset | Path | Records | vs paper Table I |
|---|---|---|---|
| CIC-IDS-2017 | `../Legacy_ds/cicids/` (8 CSVs) | **2,830,743** | 2,830,743 ✅ exact |
| UNSW-NB15 | `../Legacy_ds/unsw/` (`UNSW-NB15_1..4.csv`) | **2,540,047** | 2,540,047 ✅ exact |
| NF-ToN-IoT-v3 | `../dataset/NF-ToN-IoT-v3/NF-ToN-IoT-v3.csv` (5.3 GB) | **27,520,260** | 422,086 ⚠️ see §3 |

**CIC-IDS-2017 note.** `Thursday-WorkingHours-Morning-WebAttacks.pcap_ISCX.csv`
has 458,968 physical lines but only 170,366 real rows; the other 288,602 are
blank comma-only padding. `map_cic2017`'s `dropna(subset=[label_col])` already
discards them, which is why your original log correctly showed 170,366. **Not a
problem — no action needed.**

**NF-ToN-IoT-v3 verified authentic:** 27,520,260 rows, 10,728,046 attack
(38.98 %) / 16,792,214 benign (61.02 %) — matches the official spec exactly.
55 columns = 53 features + `Label` + `Attack`, including all 10 v3 temporal
features (`FLOW_START/END_MILLISECONDS`, IAT min/max/avg/stddev × 2 directions).

Class counts:

| Class | Count | | Class | Count |
|---|---|---|---|---|
| Benign | 16,792,214 | | injection | 381,777 |
| ddos | 4,141,256 | | dos | 203,456 |
| xss | 2,834,435 | | Backdoor | 203,384 |
| password | 1,594,777 | | mitm | 6,013 |
| scanning | 1,358,977 | | ransomware | 3,971 |

---

## 3. ⚠️ Your published ToN-IoT numbers came from the wrong file

This is the most important finding in this document.

The paper cites **NF-ToN-IoT-v3** (`b20`) and describes "53 features, NetFlow".
The code, however, loaded `train_test_network_full.csv` — the **Zeek conn-log**
release — and it was loaded **twice**. Two independent numbers prove it:

| Paper claims | Zeek file on disk | ×2 |
|---|---|---|
| ToN-IoT records **422,086** | `train_test_network.csv` = 211,043 | **422,086** ✅ |
| §V-C "MITM (2,086 in NF-ToN-IoT-v3)" | `mitm` in that file = 1,043 | **2,086** ✅ |

Both match exactly. `map_toniot`'s own docstring confirms the schema confusion
("this file uses the Zeek connection-log schema … NOT the NetFlow-v2 names"),
and `load_source` carries a comment about an earlier `*.csv`/`*.CSV` glob bug
that double-loaded files on case-insensitive filesystems.

**Consequences for the resubmission:**

- Table I's ToN row (422,086) and fused total (5,792,876) must be recomputed.
- §V-C's MITM count is wrong for v3 — the real v3 has 6,013 (all kept, see §3.1).
- Switching to the real v3 file makes §III-A's "53 features / NetFlow" claim
  **true for the first time**, so that prose needs no edit.

### 3.1 Stratified subsample (implemented)

Using all 27.5 M rows would make ToN-IoT **83.7 %** of a 32.9 M-row corpus,
swamping CIC (2.83 M) and UNSW (2.54 M). `read_nf_toniot_stratified()` draws a
class-stratified subsample instead:

| Class | Full | Kept | Rate |
|---|---|---|---|
| Benign | 16,792,214 | 256,274 | 1.53 % |
| ddos | 4,141,256 | 63,202 | 1.53 % |
| xss | 2,834,435 | 43,258 | 1.53 % |
| password | 1,594,777 | 24,339 | 1.53 % |
| scanning | 1,358,977 | 20,740 | 1.53 % |
| injection | 381,777 | 5,826 | 1.53 % |
| dos | 203,456 | 3,105 | 1.53 % |
| Backdoor | 203,384 | 3,104 | 1.53 % |
| **mitm** | 6,013 | **6,013** | **100 % (whole)** |
| **ransomware** | 3,971 | **3,971** | **100 % (whole)** |
| **TOTAL** | 27,520,260 | **≈429,832** | 1.56 % |

→ **Fused corpus ≈ 5,800,622**, within 0.13 % of the paper's 5,792,876.

Knobs in the CFG cell: `TON_TARGET_ROWS = 420_000` (set `None` to use all rows),
`TON_RARE_KEEP = 10_000`, `TON_MIN_PER_CLASS = 2_000`.

It runs two streaming passes (pass 1 counts labels via `usecols`; pass 2 samples
in 2 M-row chunks), so peak memory is one chunk, not 5.3 GB. Bernoulli sampling
seeded by `SEED` — **so each seed draws a different ToN subsample**, which is
correct: subsample variance becomes part of the variance you report.

---

## 4. Smoke-test checklist

With `SMOKE_TEST = True`, confirm these before committing to a long run:

- [ ] Loader prints `Reading ../dataset/NF-ToN-IoT-v3` (**not** `ton_iot`)
- [ ] Prints `ToN-IoT schema: NetFlow (NF-ToN-IoT), N markers matched`
- [ ] Pass 1 reports **27,520,260 rows across 10 classes**
- [ ] Pass 2 keeps **≈429,832** rows; `mitm` and `ransomware` marked `[kept whole]`
- [ ] `Fused corpus: ≈5,800,622 rows`
- [ ] Per-source audit: CIC **2,830,743**, UNSW **2,540,047**
- [ ] Class audit shows **30 classes** (29 attack + Benign)
- [ ] `SEED = 42 (override with the NSG_SEED environment variable)`
- [ ] `outputs/seed_runs/results_seed42.json` is written

If ToN row counts are wildly off, the likely cause is the old `ton_iot/` folder
being picked up as well — check `SRC_DIR_TON`. **Never load both ToN releases.**

---

## 5. What changed in the notebook (48 cells)

### 5.1 Multi-seed harness — *addresses all four reviewers*

| Cell | Change |
|---|---|
| 1 | `SEED = int(os.environ.get('NSG_SEED', 42))` + `set_all_seeds()` (numpy, torch CPU/CUDA, `PYTHONHASHSEED`). **Default is still 42.** |
| 16 | Records `train_wall_min` |
| 46 | **New** — dumps every headline metric to `outputs/seed_runs/results_seed<N>.json` |
| 47 | **New** — aggregator: mean ± std (ddof=1) LaTeX rows for the fidelity / TSTR / ablation tables, paired two-sided t-tests of NSG-IDS vs each baseline on macro-F1, and a **warning if pooled seeds used different budgets** |

`run_seeds.sh` drives the sweep (`./run_seeds.sh 42 43 44 45 46`).

### 5.2 Baseline fairness — *R1, R3*

| Cell | Change |
|---|---|
| 2 | `CTGAN_ROWS = None` → CTGAN trains on the **same corpus** as NSG-IDS (was a 100 k subsample); `CTGAN_EPOCHS` 50 → 100 |
| 32 | Prints the actual CTGAN budget vs NSG-IDS's and stores it in `baseline_results` |
| 32 | SMOTE split into **`SMOTE (CIC only)`** and **`SMOTE (Fused)`** — isolates "generative model" from "had three datasets" (R3 point 5) |
| 37, 38, 44 | Charts and the LaTeX mapper updated for the renamed keys |

⚠️ **`SMOTE` was renamed to `SMOTE (CIC only)`.** Any external script reading
`baseline_results['SMOTE']` needs updating.

⚠️ **CTGAN is the schedule risk.** It is CPU-bound and now runs on the full
corpus × 100 epochs. **Time one seed before committing to five.** If prohibitive,
the defensible fallback is to match NSG-IDS's *wall-clock budget* rather than its
epoch count, and say so explicitly in the paper.

### 5.3 Per-class evidence — *R1, R3*

| Cell | Change |
|---|---|
| 22 | `tstr_eval(per_class_out=…)` emits per-class P/R/F1/support **and confusion matrices from the same forest pass** (no extra training cost) |
| 24 | **New** — rare-attack table (Heartbleed, Worms, MITM, Shellcode, …) + full per-class CSV at `outputs/per_class_tstr_seed<N>.csv` |

This replaces the "≥500 samples per class" coverage claim that R1 and R3 both
rejected as meaningless.

### 5.4 Ablation — *R3 point 4 (the sharpest criticism)*

| Cell | Change |
|---|---|
| 14 | New **`no_src`** variant — keeps the attack-class embedding, zeroes only the dataset-source tag |
| 30 | `'w/o Source Tag': 'no_src'` added to `ABL_VARIANTS` |
| 30 | **Bug fix:** `no_cond`/`no_src` models were *sampled* with real source ids despite *training* with them zeroed — evaluated off-distribution, understating both. Now sampled with `class_to_source = {c: 0}`. |

### 5.5 Dataset plumbing

| Cell | Change |
|---|---|
| 2 | `SRC_DIR_CIC`, `SRC_DIR_UNSW`, `SRC_DIR_TON` — **set these for the CUDA box** |
| 5 | **New** `map_nf_toniot()` — NetFlow schema; ms→s and bits/s→bytes/s conversions; TCP flags decoded from the cumulative bitmask (`CLIENT_TCP_FLAGS`, else `TCP_FLAGS`; FIN=1 SYN=2 RST=4 PSH=8 ACK=16 URG=32, non-TCP zeroed); native `MIN_TTL`/`MAX_TTL` and `TCP_WIN_MAX_IN/OUT`; **real** `SRC_TO_DST_IAT_AVG/STDDEV`; packet-length std estimated from the five-bin `NUM_PKTS_*` histogram |
| 5 | **New** `map_toniot_auto()` — detects NetFlow vs Zeek by column markers, prints which it chose, and **raises rather than silently falling through** (a silent fallback is what previously made every ToN feature constant and produced KS ≈ 0.65) |
| 7 | **New** `read_nf_toniot_stratified()` (§3.1) |
| 7 | `load_source()` accepts a bare name *or* a full path, prints the directory read, and reports what it tried on failure |

All columns the mapper references were checked against the real v3 header — every
lookup resolves.

---

## 6. How many seeds?

**No reviewer named a number.** What they asked for:

- **R1:** *"repeat all experiments over multiple independent seeds and report means, standard deviations, confidence intervals, and statistical significance"*
- **R2:** *"reports results from only a single random seed, lacking variance estimates"*
- **R3:** *"evaluate the model using multiple random seeds and report the average performance and variation"*

**Use 5.**

- **3 is too few for R1's list** — a std from 3 samples is unstable and a paired
  t-test has 2 df (95 % CI multiplier 4.30), so the intervals look decorative.
- **5 is the ML convention.** With 4 df the multiplier is 2.78, and your effects
  are enormous (ToN macro-F1 0.831 vs 0.344; CTGAN KS 0.443 vs 0.030), so every
  headline gap should clear p < 0.01 comfortably.
- **10+ buys little** when margins are this wide.

The real value isn't proving the gap — it's what the variance reveals. If one of
five seeds collapses, **that is the finding**, and better you discover it than a
reviewer suspecting it.

**If compute gets tight:** spend it where the statistics are load-bearing —
5 seeds on the main model and baselines (what the t-tests consume), 3 on the
ablation (where only component ordering matters). The aggregator skips the t-test
below 3 seeds and warns on mismatched budgets.

**Budget** (previous run: 8.9 h at `DATA_PCT=0.10`, corpus essentially unchanged):

| Component | Per seed | × 5 |
|---|---|---|
| Main model (100 ep) | ~8.9 h | ~45 h |
| Ablation (6 variants, 50 ep, 60 k rows) | ~1.5 h | ~7.5 h |
| Baselines (**CTGAN unknown — time it**) | ? | ? |

---

## 7. Open items — your decisions, recorded

### 7.1 VGM claim vs code — **you chose to leave as is**

The paper claims VGM / "mode-specific normalization" in the **abstract,
contribution #4, §IV-B, §IV-C and §V-D**. The code runs
`QuantileContTransformer` — a uniform quantile transform, one scalar per column.
`CIF32Preprocessor.__init__` accepts `K=CFG['K_VGM']` and **never uses it**; the
attribute is merely *named* `self.vgm`. Its docstring says VGM was replaced
because it "floored KS ~0.17 by smearing point masses (e.g. DURATION_OUT is
44.7 % zeros)".

So your headline KS of 0.030 comes from the quantile transform, not VGM. Left
untouched per your instruction. Worth revisiting only because you plan to release
the code on acceptance, where a reviewer could see the discrepancy.

### 7.2 Paper — **left at 6 pages, unmodified**

`NSG-IDS.tex` is byte-identical to your original. An edited version addressing
the reviewers (train/test isolation, budget disclosure, honest missingness, a
full 32-feature CIF-32 mapping table, trimmed limitations) exists but was **not**
applied.

### 7.3 Numbers that must change in any resubmission

- Table I: ToN row and fused total (see §3.1)
- §V-C: MITM count (2,086 → 6,013)
- All fidelity / TSTR / ablation tables: now mean ± std over seeds
- §V-D: disclose `DATA_PCT` — the original run used a **10 % subsample**, which
  the paper never states, while describing the ablation as the reduced-budget
  condition

---

## 8. Files

| File | Status |
|---|---|
| `NSG-IDS.ipynb` | **modified** — 48 cells, 0 syntax errors |
| `run_seeds.sh` | **new** — sweep driver |
| `RUN_NOTES.md` | this file |
| `NSG-IDS.tex` | **unchanged** (byte-identical to backup) |
| `NSG-IDS.ipynb.bak-20260903-202618` | pre-change notebook backup |
| `NSG-IDS.tex.bak-20260903-202618` | paper backup |

Outputs produced per run: `outputs/seed_runs/results_seed<N>.json`,
`outputs/per_class_tstr_seed<N>.csv`, `runs/NSG-IDS_seed<N>.ipynb`,
`logs/seed<N>.log`.
