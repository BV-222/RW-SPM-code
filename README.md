# RW-SPM — Region-Wise Statistical Parametric Mapping

RW-SPM partitions images into overlapping windows, transforms each window with the log quantile density (LQD) representation, runs a regional Ball Covariance (BCov) test, and combines the connected significant regions with RW-MTCCT. The repository contains two independent Python workflows. Both use an interior LQD quantile grid from `eta` to `1 - eta`, with default `eta = 1e-3` and 21 grid points.

| Directory | Workflow |
| --- | --- |
| [Simulation](Simulation/) | Seven synthetic settings. Each region in the selected cluster contributes to the Cauchy statistic with fixed weight `1/m`, where `m` is the total number of windows. |
| [Realdata](Realdata/) | Left and right hippocampal SNP scans with covariate adjustment, a compiled SNP-specific BCov kernel and a NumPy backend. ADNI data are not included in this repository. |

Use Python 3.12 and install the requirements for the workflow you want to run.

### Original implementation and comparison methods

The original repository is included in [`Original_Implementation/`](Original_Implementation/), including its R core functions, simulation settings, comparison methods, real-data scripts, documentation, R package and code archive. See [implementation notes](Original_Implementation/IMPORT_PROVENANCE.md) and the [original README](Original_Implementation/README.md). Run the original scripts with `Original_Implementation/` as the working directory where relative paths require it. This snapshot has not undergone a code review or runtime verification as part of the import. The Python and original implementations use different defaults, including LQD endpoints, aggregation weights and ADNI screening thresholds. Select the implementation and configuration corresponding to the results you intend to reproduce.

### Simulation

From the repository root:

```bash
cd Simulation
python -m pip install -r requirements.txt
python run_simu_parallel_eta.py --settings 1 --n_rep 2 --n 24 --M1 14 --M2 11 --window_width 7 --n_workers 1 --method gamma --n_perm 19
```

This small example checks execution. For experiment runs, set the settings, sample size, replication count, window width, and calibration method explicitly. Omitting `--window_width` on the command line enables JSD-based width selection. Use `--eta` to change the truncation boundary. Use `--help` for the complete options. The default output directory `Simulation/data/result/` (when run from `Simulation/`) contains per-region statistics and p-values and one cluster-level p-value per replication.

The simulation supports `gamma`, `permu` and `limit`. Build the Ball BDD library for `limit` using `bash cball_ext/compile_linux.sh` on Linux. On Windows with MinGW-w64 on PATH, run `compile_win.bat` from the `cball_ext` directory in Command Prompt.

### ADNI analysis

From the repository root, on Linux:

```bash
cd Realdata
python -m pip install -r requirements.txt
bash cball_ext/compile_snp_linux.sh
python smoke_test_GWAS.py --backend c --n_process 2 --n_perm 19
```

On Windows with MinGW-w64 on PATH, run `compile_snp_win.bat` from the `cball_ext` directory in Command Prompt. The smoke test generates its own data and runs both hemispheres. `--backend c` requires a loadable C library. `auto` falls back to NumPy when that library is unavailable. For `--method limit`, also build the separate Ball BDD library as described above.

ADNI data should not be shared publicly and is not included in this repository. If you need the data used for this analysis, please contact us for access, subject to applicable data-use requirements. Once obtained, place the four files listed below directly in the repository's `Realdata/` folder, alongside `GWAS_left.py` and `GWAS_right.py`. The scripts read this folder by default. Alternatively, place them in another authorized directory and specify its path with `--data_dir`. The genotype input is the sampled 25,000-SNP array. The pipeline assumes the same subjects in the same row order in all matrices and cannot verify subject identity from the matrices.

| File | Object and layout |
| --- | --- |
| `ADNI_1_3_sampled_25000.npy` | Numeric subject-by-SNP array with genotype dosages coded 0, 1 or 2. SNP indices start at zero. The filename is retained for compatibility. |
| `cov_inf_0711_1_3.rdata` | `cov_inf1_3`, subject-by-covariate design matrix including the intercept. |
| `left_hipp_ori.rdata` | `left_hipp_inf1_3`, subject-by-15,000 left hippocampal measurements, flattened in row-major order from a 150 × 100 grid. |
| `right_hipp_ori.rdata` | `right_hipp_inf1_3`, same layout for the right hippocampus. |

```bash
python GWAS_left.py --snp_start 0 --n_snp 1 --method gamma --n_perm 199 --backend c --n_process 2
python GWAS_right.py --snp_start 0 --n_snp 1 --method gamma --n_perm 199 --backend c --n_process 2
```

These commands illustrate the input and output interfaces. Each hemisphere writes a SNP summary and per-region p-values under `Realdata/results_gwas_left/` or `Realdata/results_gwas_right/`. The default partition is width 11 and step 3. The ADNI branch retains the source GWAS aggregation convention, using denominator `m - n_sig + B`, where `n_sig` counts all screened regions and `B` counts those in the selected component. The `gamma_sample` option fits the permutation null on a fixed genotype-stratified subsample and uses a separate calibration from full-sample `gamma`. Choose the calibration and permutation count for the analysis being reproduced. The synthetic smoke test checks execution, including the selected backend. It provides no evidence about extreme-tail calibration.

See [third-party source notices](THIRD_PARTY_NOTICES.md) for the bundled Ball sources.
