# Project Setup and Execution Guide

Follow these steps to preprocess data, train local baselines, run a federated learning simulation, and evaluate the results.

---

## Part 1: Environment & Data Preparation

### 1. Activate Virtual Environment
**Windows (PowerShell):**
```powershell
.venv\Scripts\Activate.ps1

```

**Linux / Mac:**

```bash
source .venv/bin/activate

```

### 2. Preprocess Data

Convert raw `.psv` files into per-patient `.pt` files (parallel processing).

```bash
python src/parallel_preprocess.py --raw_folder data/raw --out_folder data/processed/patients --seq_len 48 --nprocs 8

```

**(Optional) LMDB Packing**
Recommended for faster I/O if using large datasets (e.g., 40k files).

```bash
# Pack into shards
python src/lmdb_packer.py --in_folder data/processed/patients --out_folder data/processed/lmdb_shards --shard_size 4000

# Copy LMDB index
cp data/processed/lmdb_shards/index_lmdb.pt data/processed/patients/index_with_labels_lmdb.pt

```

---

## Part 2: Local Baselines & Search

### 3. Train Local Baseline (Transformer)

```bash
python src/train_local.py --index data/processed/patients/index_with_labels.pt --epochs 10 --batch_size 64 --model transformer

```

### 4. Train GRU-D (Missing-Data Aware)

```bash
python src/train_local.py --index data/processed/patients/index_with_labels.pt --epochs 10 --batch_size 64 --model grud

```

### 5. Hyperparameter Search (Small Grid)

```bash
python src/hyperparam_search.py

```

---

## Part 3: Federated Simulation

### 6. Split Clients

Simulate a federated environment by splitting data into 3 distinct clients.

```bash
python src/split_clients.py --processed_folder data/processed/patients --out_root data/processed/clients --n_clients 3

```

### 7. Start Server

**Action:** Open **Terminal 1** and run:

```bash
python src/fl_server.py --model transformer --n_features 40 --seq_len 48 --min_clients 2 --rounds 5

```

### 8. Start Clients

**Action:** Open new terminals for each client.

**Terminal 2 (Client 1):**

```bash
python src/fl_client.py --index data/processed/clients/client1/index.pt --model transformer --server_address 127.0.0.1:8080

```

**Terminal 3 (Client 2):**

```bash
python src/fl_client.py --index data/processed/clients/client2/index.pt --model transformer --server_address 127.0.0.1:8080

```

**Terminal 4 (Client 3):**

```bash
python src/fl_client.py --index data/processed/clients/client3/index.pt --model transformer --server_address 127.0.0.1:8080

```

### 9. Secure Aggregation PoC

Run locally to demonstrate that masks cancel out.

```bash
python src/secure_agg_poc.py

```

---

## Part 4: Evaluation & Comparison

### 10. Train "Local-Only" Baseline

Train a model exclusively on Client 1's data to compare against the Federated model.

```bash
python src/train_local.py --index data/processed/clients/client1/index.pt --model transformer --run_name local_client1_model --epochs 10

```

> **Note:** When finished, copy the path to `model_best.pt` printed in the output (e.g., `runs\20251115T...model\checkpoints\model_best.pt`).

### 11. Evaluate the Federated Model

Test the **Federated** model against the Client 3 test set.

```bash
python src/evaluate.py --index data/processed/clients/client3/index.pt --ckpt server_out/global_best.pt --model transformer --n_features 40 --seq_len 48 --out_file eval_results_federated.json

```

### 12. Evaluate the Local-Only Model

Test the **Local** model against the Client 3 test set.
*Replace the path in quotes below with the path you copied in Step 10.*

```bash
python src/evaluate.py --index data/processed/clients/client3/index.pt --ckpt "runs\YOUR_TIMESTAMP_local_client1_model\checkpoints\model_best.pt" --model transformer --n_features 40 --seq_len 48 --out_file eval_results_local.json

```

### 13. Generate Final Plot

Combine both evaluation files into a comparison graph.

```bash
python src/plot_results.py --results eval_results_federated.json eval_results_local.json --out_file model_comparison_plot.png
