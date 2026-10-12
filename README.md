# malware-clf

Malware family classification from Windows API call sequences.

The main model is TPHNet (`epaht` in configs) with ACT-Loss (`use_afc_loss`). BERT, CANINE, RTF, LSTM, and BEACON are included as baselines. Training is config-driven via the `malware-clf` CLI.

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

`build-avast` writes `sha256` and `date` next to `api`, `class`, and `split`. A temporal retrain needs those columns. This repo does not include the parquet.

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

## BEACON

BEACON (`configs/model/beacon.yaml`) hashes the `api` column onto an 11×768 canvas. Saved scores are that port, not the paper's 0.985 weighted F1.

Official Avast lists, 20 epochs, class weights (`beacon_avast.yaml`, batch 32):

```bash
malware-clf train --model beacon --dataset avast --train-config beacon_avast
```

The other datasets use `beacon_catak`, `beacon_oliveira`, `beacon_virussample`, `beacon_virusshare`, and `beacon_winmet`:

```bash
malware-clf train --model beacon --dataset catak --train-config beacon_catak
malware-clf train --model beacon --dataset oliveira --train-config beacon_oliveira
malware-clf train --model beacon --dataset virussample --train-config beacon_virussample
malware-clf train --model beacon --dataset virusshare --train-config beacon_virusshare
malware-clf train --model beacon --dataset winmet --train-config beacon_winmet
```

Temporal Avast (`configs/dataset/avast_temporal.yaml`) holds out the newest 24% as test and takes one of five folds on the older 76% as validation. `build-avast` must have written `sha256` and `date` into `data/avast.parquet`. Check a fold, then train BEACON with the 200-epoch paper recipe (`beacon_avast_paper.yaml`: batch 64, no class weights) and TPHNet on the same fold:

```bash
malware-clf evaluate --dataset avast_temporal --cv-fold 0 --split-seed 42

malware-clf train --model beacon --dataset avast_temporal \
  --train-config beacon_avast_paper --cv-fold 0 --split-seed 42

malware-clf train --model epaht --dataset avast_temporal \
  --train-config epaht_avast_afc_v4 --cv-fold 0 --split-seed 42
```

`--split-seed 42` is the fold-shuffle seed. Repeat with `--cv-fold 1` through `--cv-fold 4`. `--no-cv-shuffle` assigns folds in date order.

## Layout

```
src/malware_clf/   package and CLI
configs/           dataset, model, and train YAML
results/           saved runs
```

## License

GPL-3.0
