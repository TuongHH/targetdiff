# Notes on the generated data

Contents of this directory are gitignored (see `.gitignore`); only this note and
`affinity_info.pkl` are tracked.

Expected result of running the three scripts in `scripts/data_preparation` on
CrossDocked2020 **v1.1**:

| file                                                     | expected                               |
|----------------------------------------------------------|----------------------------------------|
| `crossdocked_v1.1_rmsd1.0/index.pkl`                     | 184,087 pairs                          |
| `crossdocked_v1.1_rmsd1.0_pocket10/index.pkl`            | 184,087 entries, 17,587 with `None`    |
| `..._pocket10_processed_final.lmdb`                       | 166,500 entries (= 184,087 − 17,587)   |
| `crossdocked_pocket10_pose_split.pt`                      | train 99,990 / val 0 / test 100        |

## Why the training set has 99,990 pairs and not 100,000

`split_by_name.pt` lists 100,000 training pairs, but `split_pl_dataset.py` prints ten
`Warning: data with PDB fn ... not found!` lines and the resulting split has 99,990. This
is expected and is **not** a preprocessing error. All 100 test pairs are present.

The cause is a version mismatch. `split_by_name.pt` is the fixed split published by
[AR](https://arxiv.org/abs/2203.10446) and
[Pocket2Mol](https://arxiv.org/abs/2205.07249), which was built from CrossDocked2020
**v1.0**. This repository's `clean_crossdocked.py` reads the **v1.1** types file
(`types/it2_tt_v1.1_completeset_train0.types`). Ten of the v1.0 training pairs do not
survive into v1.1, for two different reasons.

### 1. RMSD above the 1.0 Å cutoff in v1.1 (8 pairs)

These poses exist in the v1.1 types file, but their re-computed RMSD exceeds the
`--rmsd_thr 1.0` filter in `clean_crossdocked.py`, so they are never copied into
`crossdocked_v1.1_rmsd1.0/` and never reach `index.pkl`.

| pose (v1.1 RMSD, Å) | pocket |
|---|---|
| `3sjq_C_rec_3sjq_phu_lig_tt_min_0` — **1.017** | `KCNN2_RAT_399_487_0` |
| `3rrm_A_rec_3rrm_adp_lig_tt_min_0` — **1.176** | `DBP5_YEAST_295_482_0` |
| `4wks_A_rec_3l91_oca_lig_tt_min_0` — **2.547** | `PVDQ_PSEAE_26_217_0` |
| `4k2g_A_rec_3l91_oca_lig_tt_min_0` — **3.872** | `PVDQ_PSEAE_26_217_0` |
| `2rd0_A_rec_5sx8_luz_lig_tt_docked_9` — **4.510** | `PK3CA_HUMAN_331_481_0` |
| `5lgk_D_rec_1f6a_cps_lig_tt_docked_7` — **5.930** | `IGHE_HUMAN_206_427_0` |
| `5lgj_A_rec_1f6a_cps_lig_tt_docked_17` — **6.507** | `IGHE_HUMAN_206_427_0` |
| `5lgj_A_rec_1f6a_cps_lig_tt_docked_13` — **6.714** | `IGHE_HUMAN_206_427_0` |

Two sit just over the boundary (1.017 and 1.176 Å) — these are genuine borderline poses
whose RMSD was recomputed slightly upward between releases. The other six are far above
1.0 Å, so those poses were substantively re-docked in v1.1.

### 2. Pose absent from v1.1 entirely (2 pairs)

| pose | pocket |
|---|---|
| `2y7q_B_rec_1f6a_cps_lig_it1_it2_tt_docked_11` | `IGHE_HUMAN_206_427_0` |
| `5hys_G_rec_1f6a_cps_lig_it1_it2_tt_docked_17` | `IGHE_HUMAN_206_427_0` |

These two pose names appear in **no** v1.1 types file, and no matching
`.gninatypes` file exists in the v1.1 download. The docking runs for that pocket were
redone between releases and the pose numbering changed, so these two v1.0 poses have no
v1.1 counterpart. They cannot be recovered without the v1.0 archive.

Nine of the ten come from just three pockets (`IGHE_HUMAN_206_427_0` ×5,
`PVDQ_PSEAE_26_217_0` ×2, plus one each from `KCNN2_RAT`, `DBP5_YEAST`, `PK3CA_HUMAN`),
so the loss is concentrated and has negligible effect on training.

## The .lmdb and the split file must be used as a pair

`extract_pockets.py` collects worker results with `imap_unordered`, so the row order of
`crossdocked_v1.1_rmsd1.0_pocket10/index.pkl` is **not** reproducible between runs. LMDB
keys are assigned by position in that index, and split entries are indices into the
LMDB's sorted key list. Two byte-identical rebuilds therefore produce different
index → molecule mappings.

Consequence: the locally built `.lmdb` and the locally built `_pose_split.pt` must be
used together. Mixing a locally built `.lmdb` with the `crossdocked_pocket10_pose_split.pt`
downloaded from the authors' Google Drive folder — or the reverse — silently trains and
evaluates on the wrong molecules, with no error raised. If you rebuild one, rebuild both.

## Element filtering is skipped when --fixed_split is used

The `allowed_elements = {1,6,7,8,9,15,16,17}` filter in `split_pl_dataset.py` lives only
in the branch taken when `--fixed_split` is *not* given. Running with
`--fixed_split split_by_name.pt` skips it, so the full LMDB legitimately retains 7,053
entries with ligand atoms outside that set (Br, I, B, …). None of them fall inside the
train or test split, which was verified by scanning all 99,990 + 100 split entries.

This matters because `configs/training.yml` sets `ligand_atom_mode: add_aromatic`, and
`utils/transforms.py` maps an unrecognised element to **hydrogen** with a bare `print`
rather than raising — an exotic atom inside the split would be silently relabelled.
