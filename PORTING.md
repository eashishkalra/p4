# NSG-IDS — Porting the run to another machine

Companion to `RUN_NOTES.md`. That file says *what* to run and why; this one says
what to carry across and how to stand the environment up.

---

## 1. Files to copy

### 1.1 Code — from `Publications/Conf2/`

| File | Size | Needed |
|---|---|---|
| `NSG-IDS.ipynb` | 633 KB | **yes** — the whole pipeline |
| `run_seeds.sh` | 5.5 KB | **yes** — sweep driver + preflight |
| `RUN_NOTES.md` | 14 KB | yes — the §4 smoke checklist lives here |
| `PORTING.md` | this file | yes |
| `ctgan_timing_snippet.py` | 2.4 KB | recommended — CTGAN budget probe |
| `NSG-IDS.tex`, `refs.bib`, `IEEEtran.cls` | 358 KB | only to rebuild the paper there |
| `*.aux .bbl .blg .fdb_latexmk .fls .log .pdf` | — | **no** — LaTeX build artifacts |
| `*.bak-*` | — | **no** |

### 1.2 Datasets — ~6.6 GB of CSV

Copy **only the CSVs**. The `.parquet` files beside them (545 MB) are never read:
the loader globs `*.csv` / `*.CSV` only.

| Source | Files | Size |
|---|---|---|
| `Legacy_ds/cicids/` | the 8 `*.pcap_ISCX.csv` files | 1.12 GB |
| `Legacy_ds/unsw/` | `UNSW-NB15_1.csv` … `UNSW-NB15_4.csv` | 0.55 GB |
| `dataset/NF-ToN-IoT-v3/` | `NF-ToN-IoT-v3.csv` | 4.94 GB |

Optional but harmless: the rest of `Legacy_ds/unsw/`. `load_source(prefer=...)`
selects the raw `UNSW-NB15_[1-4]` parts and prints which derived files it skips,
and `SKIP_CSV_TOKENS = ('feature','list_event','readme','schema','_gt')` drops
`NUSW-NB15_features.csv`. So a whole-folder copy still yields 2,540,047 rows.

> **Do NOT copy `dataset/ton_iot/`.** That is the Zeek conn-log release behind the
> wrong published numbers (`RUN_NOTES.md` §3). Leaving it off the new machine is
> the cleanest way to guarantee it is never picked up.

### 1.3 Layout — paths are relative

`CFG` uses `'../Legacy_ds/cicids'`, `'../Legacy_ds/unsw'`,
`'../dataset/NF-ToN-IoT-v3'`, so reproduce this shape and cell 2 needs no edit:

```
<anywhere>/
├── Legacy_ds/
│   ├── cicids/    *.pcap_ISCX.csv
│   └── unsw/      UNSW-NB15_1..4.csv
├── dataset/
│   └── NF-ToN-IoT-v3/  NF-ToN-IoT-v3.csv
└── Conf2/         NSG-IDS.ipynb, run_seeds.sh, ...   <- run from here
```

Any other layout: edit `SRC_DIR_CIC` / `SRC_DIR_UNSW` / `SRC_DIR_TON` in cell 2.
They accept a bare folder name under `DATASET_DIR` *or* a full path.

**Put the working copy on a local disk, not a synced cloud folder.** Multi-day
runs writing into OneDrive/Dropbox invite mid-write locks. (On the current
machine every dataset file is *pinned*, so this is not a live problem there.)

---

## 2. Environment

Needs an NVIDIA GPU + driver. Third-party imports across all 48 cells:

```
torch  pandas  numpy  scipy  scikit-learn  matplotlib  seaborn
joblib  tqdm  ctgan  imbalanced-learn        + nbconvert  ipykernel
```

```bash
conda create -n tf9 python=3.11 -y
conda activate tf9
# match the CUDA build to the target driver -- see pytorch.org
pip install torch --index-url https://download.pytorch.org/whl/cu128
pip install pandas numpy scipy scikit-learn matplotlib seaborn joblib tqdm \
            ctgan imbalanced-learn nbconvert ipykernel

# REQUIRED: register a kernel that points at THIS interpreter
python -m ipykernel install --user --name tf9 --display-name "Python (tf9)"
```

> **The kernel step is not optional.** The notebook's stored kernelspec is named
> `python3`, whose `display_name` may read "tf9" while resolving to an entirely
> different env — a bare `python` off `PATH`. `run_seeds.sh` now pins
> `--ExecutePreprocessor.kernel_name` and *verifies* the kernel resolves to the
> same interpreter it package-checked, aborting if not.

Verify in one shot — this is exactly what the sweep preflight does:

```bash
./run_seeds.sh 42     # aborts at the SMOKE_TEST guard; env checks run first
```

Override the defaults if your env or kernel is named differently:
`NSG_ENV=myenv NSG_KERNEL=myenv ./run_seeds.sh 42`, or `NSG_PY=/path/to/python`.

---

## 3. Full-run steps

### Step 0 — tune cell 2 for the target GPU

- `MATCH_BACKENDS` (line ~81) forces fp32 / AMP off / `BATCH_SIZE=512` so CUDA
  and MPS agree. **Set it to `False` on a single-GPU sweep**: all seeds run on
  one card, so cross-backend matching buys nothing, and it unlocks
  `BATCH_SIZE_GPU=1024` + AMP (CFG rates this ~2-3x). If CUDA OOMs, drop
  `BATCH_SIZE_GPU` to 512 — the ≤6 GB note is at line 68.
- `NUM_WORKERS = 0` is already correct for notebook execution on Windows.
- Keep `DATA_PCT` **identical across all seeds**; the aggregator warns on
  mismatched budgets.

### Step 1 — smoke test (never skip; `RUN_NOTES.md` §4)

Set `SMOKE_TEST = True` in cell 2, then:

```bash
NSG_SEED=42 PYTHONIOENCODING=utf-8 python -m nbconvert \
    --to notebook --execute --ExecutePreprocessor.timeout=-1 \
    --ExecutePreprocessor.kernel_name=tf9 \
    --output runs/smoke_seed42.ipynb NSG-IDS.ipynb
```

Check every box in `RUN_NOTES.md` §4. The three that actually catch the §3
wrong-file bug: pass 1 = **27,520,260 rows / 10 classes**, pass 2 keeps
**≈429,832** with `mitm` + `ransomware` `[kept whole]`, fused **≈5,800,622**.

### Step 2 — measure the budget before committing

The 8.9 h/seed figure came from a different GPU; measure yours.

- **Main model:** `SMOKE_TEST = False`, `FAST_EPOCHS = 2`, run one seed. Cell 16
  records `train_wall_min`; × 50 gives the true 100-epoch cost.
- **CTGAN:** run cells 1,2,4,5,6,7,9,10 in Jupyter, then paste
  `ctgan_timing_snippet.py`. CTGAN never touches the NSG-IDS generator, so cell
  16 is *not* a prerequisite — you get the number without the long run.
  (In ctgan 0.12.1 `enable_gpu` defaults True, so the training loop is on GPU;
  the per-column BayesianGMM in `DataTransformer` is still CPU-bound.)

Reset `FAST_EPOCHS = 0` afterwards.

### Step 3 — the sweep

```bash
# cell 2: SMOKE_TEST = False, FAST_EPOCHS = 0
./run_seeds.sh 42            # one seed first, compare against published results
./run_seeds.sh 43 44 45 46   # then the rest
```

Preflight aborts on a missing package, a mismatched kernel, or a left-over
`SMOKE_TEST = True` — before the loop, so mistakes cost seconds not hours.

### Step 4 — aggregate

Open `NSG-IDS.ipynb`, run the §14 aggregator (cell 47) → mean ± std LaTeX rows
and paired t-tests. Needs ≥3 seeds for the t-test.

Per run you get: `outputs/seed_runs/results_seed<N>.json`,
`outputs/per_class_tstr_seed<N>.csv`, `runs/NSG-IDS_seed<N>.ipynb`,
`logs/seed<N>.log`.

### Step 5 — if compute is tight

`RUN_NOTES.md` §6: 5 seeds on the main model and baselines (the t-tests consume
those), 3 on the ablation. For CTGAN, §5.2's fallback is to match NSG-IDS's
*wall-clock* budget rather than its epoch count — and say so in the paper.

---

## 4. Bringing results home

Copy back `outputs/`, `logs/`, and `runs/` — a few hundred MB at most. The
aggregator only reads `outputs/seed_runs/results_seed*.json`, so those five
files are the minimum. Do not copy the datasets back.
