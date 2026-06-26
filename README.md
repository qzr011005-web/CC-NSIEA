\
# CC-NSIEA: Class-Contrast Neuro-Symbolic Evidence Audit

This repository contains the **code-only** release for the final method in the
paper *Neuro-Symbolic Class-Contrast Evidence Audit for Reliable Cross-Subject
Wearable Activity Recognition*.

CC-NSIEA is a **label-preserving** reliability-audit framework for six-class
wearable activity recognition on UCI HAR. A Temporal Residual Perception Network
(TRPN) produces the only activity label, posterior probabilities, and normalized
temporal embedding. The audit layer retrieves read-only training-subject evidence,
computes explicit consistency checks, and estimates an evidence-risk score. It
never changes the neural label.

## Final method boundary

The final CC-NSIEA pipeline contains:

- **TRPN**: Temporal Residual Perception Network for the activity label and embedding.
- **TSEM**: Training-Subject Evidence Memory built only from model-train and
  model-selection subjects within each outer fold.
- **SECA**: Symbolic Evidence Consistency Audit with data-validity,
  dynamic/static motion coherence, retrieval-support, and class-separation checks.
- **ESC**: Evidence Sufficiency Controller with deterministic ordered stopping.
- **CCER**: Class-Contrast Evidence Refinement. When a second evidence round is
  required, the method contrasts the neural predicted class against its strongest
  posterior competitor.
- **KPC**: Knowledge Provenance Chain containing evidence identifiers, rule
  outcomes, tau history, risk, and stopping reason.

The audit output is `(neural_label, audit_risk, provenance_record)`. The
invariant `override_permitted = False` is checked in the run scripts.

## What this release contains

- Source code, fixed configuration, and the predeclared four-way
  subject-disjoint protocol.
- Training, audit, matched-comparison, and subject-cluster-bootstrap scripts.
- Unit tests and setup documentation.

It deliberately excludes UCI HAR data, trained checkpoints, OOF predictions,
KPC records, logs, cache files, intermediate outputs, and paper-result files.

## Requirements

- Python 3.10 or later
- PyTorch 2.0 or later
- A CUDA-capable GPU is recommended for the five-fold perception runs

```bash
git clone https://github.com/<YOUR_GITHUB_USERNAME>/CC-NSIEA.git
cd CC-NSIEA

python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
pip install -e .
pytest -q
```

## Dataset preparation

Download **UCI Human Activity Recognition Using Smartphones** separately and
place it at:

```text
data/UCI HAR Dataset/
├── train/
│   ├── Inertial Signals/
│   ├── subject_train.txt
│   └── y_train.txt
└── ...
```

The code reads only `data/UCI HAR Dataset/train/`. It does not load the
official UCI `test/` partition or its labels.

## Reproduce the final CC-NSIEA comparison

The committed protocol file is the exact predeclared five-fold split used by
this code release. Run the following commands from the repository root.

```bash
# Optional integrity check for the committed configuration and protocol
python scripts/validate_setup.py \
  --config configs/cc_nsiea_uci.yaml \
  --protocol protocols/cc_nsiea_subject_cv.json

# 1) Train the shared perception module once per outer fold.
python -u scripts/run_shared_perception_cv.py \
  --config configs/cc_nsiea_uci.yaml \
  --protocol protocols/cc_nsiea_subject_cv.json

# 2) Run the matched dynamic deterministic controller (D-NSIEA).
python -u scripts/run_audit_controller_cv.py \
  --config configs/cc_nsiea_uci.yaml \
  --protocol protocols/cc_nsiea_subject_cv.json \
  --perception-run runs/uci_cc_nsiea_perception \
  --controller deterministic

# 3) Run the final fixed class-contrast controller (CC-NSIEA).
python -u scripts/run_audit_controller_cv.py \
  --config configs/cc_nsiea_uci.yaml \
  --protocol protocols/cc_nsiea_subject_cv.json \
  --perception-run runs/uci_cc_nsiea_perception \
  --controller class_contrast \
  --deterministic-tau-selection \
  analysis/uci_cc_nsiea_audit/deterministic/tau_selection.json

# 4) Verify that the two controllers used identical neural predictions.
python -u scripts/compare_audit_controllers.py \
  --left-oof runs/uci_cc_nsiea_audit/deterministic/oof_audit_predictions.csv \
  --right-oof runs/uci_cc_nsiea_audit/class_contrast/oof_audit_predictions.csv \
  --left-name D-NSIEA \
  --right-name CC-NSIEA \
  --output analysis/uci_cc_nsiea_audit/d_vs_cc_matched.json

# 5) Estimate paired uncertainty by resampling subjects, not individual windows.
python -u scripts/subject_cluster_bootstrap.py \
  --left-oof runs/uci_cc_nsiea_audit/deterministic/oof_audit_predictions.csv \
  --right-oof runs/uci_cc_nsiea_audit/class_contrast/oof_audit_predictions.csv \
  --left-name D-NSIEA \
  --right-name CC-NSIEA \
  --iterations 10000 \
  --seed 2030 \
  --output-json analysis/uci_cc_nsiea_audit/d_vs_cc_subject_cluster_bootstrap.json \
  --fold-output-csv analysis/uci_cc_nsiea_audit/d_vs_cc_subject_cluster_fold_deltas.csv
```


## Optional constrained language-model diagnostic

The submitted paper reports a constrained language-model controller only as a
**diagnostic control**, not as the final method. Its source is included under
`src/cc_nsiea/diagnostics/` and `scripts/run_llm_diagnostic.py` so that the
diagnostic can be independently inspected or rerun. It is not invoked by the
primary CC-NSIEA commands above.

No API key, prompt/response cache, or diagnostic output is included in this
release. To enable this optional control, install the separate dependency and
set the API key only in the shell environment:

```bash
pip install -r requirements-diagnostic.txt
read -r -s -p "DASHSCOPE_API_KEY: " DASHSCOPE_API_KEY
echo
export DASHSCOPE_API_KEY

python -u scripts/run_llm_diagnostic.py \
  --config configs/llm_diagnostic_uci.yaml \
  --protocol protocols/cc_nsiea_subject_cv.json \
  --perception-run runs/uci_cc_nsiea_perception \
  --controller llm \
  --deterministic-tau-selection \
  analysis/uci_cc_nsiea_audit/deterministic/tau_selection.json
```

The diagnostic controller is constrained to choose a whitelisted evidence view
and cannot revise a label, posterior probability, or stopping decision.

## Generated outputs

The commands create the following local directories. They are ignored by Git:

```text
artifacts/  # checkpoints and frozen neural audit inputs
runs/       # OOF predictions and KPC records
analysis/   # summaries, controller comparisons, bootstrap intervals
```

Keep these files locally or archive them as supplementary reproducibility
materials. Do not commit large raw data, model checkpoints, cached outputs,
API keys, or absolute server paths to the public repository.

## Citation

```bibtex
@article{li2026ccnsiea,
  title={Neuro-Symbolic Class-Contrast Evidence Audit for Reliable Cross-Subject Wearable Activity Recognition},
  author={Li, Qiang and Zhang, Xiaohong and Yan, Meng},
  journal={Sensors},
  year={2026},
  note={Manuscript under review}
}
```

## License

This repository is released under the MIT License. The UCI HAR dataset remains
subject to its original distribution terms.
