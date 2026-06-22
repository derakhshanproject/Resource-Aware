# HFL-SDN-IDS Proposed Method Implementation

This repository implements only the **proposed method** from the article:

> A Resource-Aware Framework for Energy and Bandwidth Efficient IoT Intrusion Detection Using Lightweight Federated Learning over SDN

The code implements **HFL-SDN-IDS**: a resource-aware hierarchical federated learning framework with a lightweight IDS model and an SDN-based enforcement loop.

## What is included

- Complete project structure for academic reproducibility.
- Dataset preparation pipeline for CSV-based IDS datasets.
- Mutual-information feature selection with exactly **23 selected features**.
- IID and non-IID Dirichlet client partition generation.
- Exact implementation architecture: `23 -> 64 -> 32 -> C` feedforward IDS model.
- Hierarchical federated learning with clients, gateways, and global server.
- 8-bit update quantization.
- Energy, bandwidth, convergence, detection, and SDN-overhead metrics.
- Simulated SDN controller and optional Ryu/Mininet files.
- Automatic `.pth` checkpoint/model saving and `.md5` checksum generation.
- Evaluation and figure/table generation scripts.

## Important honesty note

The ZIP does **not** include the real benchmark datasets because CICIDS2017, N-BaIoT, TON-IoT, Edge-IIoTset, and UNSW-NB15 are large third-party datasets. Therefore, real article-level performance cannot be guaranteed from the ZIP alone. The code is ready to run once the datasets are placed under `data/raw/<DATASET_NAME>/`.

For a quick smoke test, you can generate a small synthetic CICIDS-like dataset using `scripts/00_generate_synthetic_demo.py`. Demo results are only for verifying the pipeline, not for publication.

## Project layout

```text
hfl_sdn_ids_proposed_final/
├── config.yaml
├── configs/
│   └── experiments/
│       ├── main_cicids2017.yaml
│       └── demo_synthetic.yaml
├── data/
│   ├── raw/
│   ├── processed/
│   └── splits/
├── src/
│   ├── data/
│   ├── federated/
│   ├── metrics/
│   ├── model/
│   ├── sdn/
│   └── utils/
├── scripts/
├── sdn_assets/
│   ├── mininet/
│   └── ryu/
├── supplementary/
├── outputs/
└── tests/
```

## Article-aligned default configuration

The default `config.yaml` uses the main experimental setup:

```yaml
num_clients: 100
num_clusters: 10
participation_ratio: 0.8
global_rounds: 30
local_epochs: 5
batch_size: 32
optimizer: SGD
learning_rate_initial: 0.01
learning_rate_minimum: 0.0001
quantization_bits: 8
dirichlet_alpha_values: [0.3, 0.5]
sdn_detection_threshold: 0.85
random_seed: 42
model_architecture: 23 -> 64 -> 32 -> C
```

The exact hidden-layer widths are an implementation decision because the article describes a compact feedforward model but does not provide the neuron counts. This repository fixes them explicitly for reproducibility.

## Installation

### Option A: venv

```cmd
python -m venv .venv
.venv\Scripts\activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### Option B: conda

```cmd
conda env create -f environment.yml
conda activate hfl_sdn_ids
```

## Real dataset placement

Place raw CSV files here:

```text
data/raw/CICIDS2017/*.csv
data/raw/N_BaIoT/*.csv
data/raw/TON_IoT/*.csv
data/raw/Edge_IIoTset/*.csv
data/raw/UNSW_NB15/*.csv
```

The loader automatically detects common label columns such as `Label`, `label`, `attack_cat`, and `Attack_type`.

## Full run sequence for real experiments

```cmd
python scripts\01_prepare_dataset.py --config config.yaml
python scripts\02_create_iid_splits.py --config config.yaml
python scripts\03_create_non_iid_splits.py --config config.yaml
python scripts\04_train_proposed_hfl_sdn_ids.py --config config.yaml
python scripts\05_evaluate_proposed.py --config config.yaml
python scripts\09_generate_tables_and_figures.py --config config.yaml
python scripts\10_export_model.py --config config.yaml

# Optional slow search for best operating point
python scripts\11_grid_search_operating_points.py --config config.yaml
```

## Quick smoke test without real datasets

```cmd
python scripts\00_generate_synthetic_demo.py --rows 400
python scripts\01_prepare_dataset.py --config configs\experiments\demo_synthetic.yaml
python scripts\03_create_non_iid_splits.py --config configs\experiments\demo_synthetic.yaml
python scripts\04_train_proposed_hfl_sdn_ids.py --config configs\experiments\demo_synthetic.yaml
python scripts\05_evaluate_proposed.py --config configs\experiments\demo_synthetic.yaml
python scripts\09_generate_tables_and_figures.py --config configs\experiments\demo_synthetic.yaml
```

## Outputs

Models and checkpoints:

```text
outputs/checkpoints/best/hfl_sdn_ids_best.pth
outputs/checkpoints/best/hfl_sdn_ids_best.md5
outputs/checkpoints/final/hfl_sdn_ids_final_checkpoint.pth
outputs/checkpoints/final/hfl_sdn_ids_final_checkpoint.md5
outputs/models/proposed/hfl_sdn_ids_final_model.pth
outputs/models/proposed/hfl_sdn_ids_final_model.md5
outputs/models/exported/hfl_sdn_ids_torchscript.pt
outputs/models/exported/hfl_sdn_ids_torchscript.md5
```

Metrics:

```text
outputs/metrics/<DATASET>/training_history.csv
outputs/metrics/<DATASET>/validation_metrics.json
outputs/metrics/<DATASET>/test_metrics.json
outputs/metrics/<DATASET>/evaluation_from_checkpoint.json
```

SDN logs:

```text
outputs/logs/sdn/sdn_enforcement_log.csv
outputs/logs/sdn/sdn_enforcement_summary.json
```

Figures and tables:

```text
outputs/figures/<DATASET>/
outputs/tables/<DATASET>/
```

## How non-IID Dirichlet partitions are generated

The script `scripts/03_create_non_iid_splits.py` calls:

```python
create_dirichlet_client_indices(labels, num_clients, alpha, seed)
```

For each class, class indices are shuffled and distributed to clients using a Dirichlet distribution with concentration parameter `alpha`. Lower `alpha` creates stronger non-IID behavior. The default values are `alpha=0.3` and `alpha=0.5`.

## Energy and bandwidth calculation

Energy follows:

```text
E_i = kappa * f_i^2 * C_i + P_i * (B_i / R_i)
```

The implementation is in:

```text
src/metrics/energy_metrics.py
src/metrics/bandwidth_metrics.py
```

Bandwidth is computed from model parameter count and quantization bit width:

```text
client upload bytes = parameter_count * quantization_bits / 8
gateway upload bytes = parameter_count * 32 / 8
```

## SDN / Mininet / Ryu

The default run uses a reproducible simulated SDN controller in:

```text
src/sdn/sdn_controller.py
```

Optional files for Linux environments:

```text
sdn_assets/mininet/mininet_hfl_topology.py
sdn_assets/ryu/ryu_hfl_ids_controller.py
```

## Reviewer-response coverage

This package addresses the requested reproducibility items:

- Repository-ready code structure.
- Experiment config files.
- Exact model architecture.
- Random seeds in `config.yaml`.
- Dirichlet partition script.
- Evaluation script.
- Energy and bandwidth calculation scripts.
- SDN/Ryu/Mininet files.
- Supplementary material file.

Recommended publication action: upload this repository to GitHub and archive a release on Zenodo/Figshare. Then add the repository DOI/link to the article.
