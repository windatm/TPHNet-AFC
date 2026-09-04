# malware-clf

Malware family classification from Windows API call sequences.

The main model is TPHNet (`epaht` in configs) with ACT-Loss (`use_afc_loss`). BERT, CANINE, RTF, and LSTM are included as baselines. Training is config-driven via the `malware-clf` CLI.

## Setup

NVIDIA GPU with CUDA. 16 GB VRAM is enough for the configs in this repo (BERT on AVAST uses batch size 4).

```bash
conda env create -f environment.yml
conda activate malware-clf
pip install -e ".[dev]"
```

## Data

CSV datasets (VirusSample, VirusShare, Oliveira, Catak) go in `datasets/datasets/`. Column names are `api` and `class`.

If you only have the original Catak file:

```bash
malware-clf prepare-catak \
  --in datasets/datasets/CatakOriginal.csv \
  --out datasets/datasets/CatakPreprocessed.csv
```

AVAST and WinMET are built once from sandbox JSON reports:

```bash
malware-clf build-avast \
  --reports-dir /path/to/public_full_reports \
  --labels /path/to/public_labels.csv \
  --train-list /path/to/TrainFileList_AvastCuckooCom.txt \
  --test-list /path/to/TestFileList_AvastCuckooCom.txt \
  --out data/avast.parquet

malware-clf build-winmet \
  --reports-dir /path/to/winmet/DATASET \
  --out data/winmet.parquet \
  --dedup
```

```bash
malware-clf evaluate --dataset virussample
```

## Train

`--model`, `--dataset`, and `--train-config` are YAML stems under `configs/` (`--model` also checks `configs/model/ablations/`). Output goes to `results/<model>_<dataset>_<config>_<timestamp>/`.

```bash
malware-clf train --model canine --dataset virussample --train-config virussample
malware-clf train --model epaht --dataset avast --train-config epaht_avast
malware-clf train --model epaht --dataset avast --train-config epaht_avast_afc_v4
```

Each run writes `metrics.json` and `experiment.yaml`. Checkpoint selection uses validation macro AUC. Reported metrics are macro F1, AUC, accuracy, and MCC.

Completed runs used in the comparison tables are already under `results/`. Five-seed means are in `results/analysis/multiseed/summary.json`. Training logs are in `results/logs/`.

## Layout

```
src/malware_clf/   package and CLI
configs/           dataset, model, and train YAML
results/           saved runs
```

## License

GPL-3.0
