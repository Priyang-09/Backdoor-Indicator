# BackdoorIndicator
### Leveraging OOD Data for Proactive Backdoor Detection in Federated Learning
**USENIX Security 2024** | [Paper](https://www.usenix.org/conference/usenixsecurity24/presentation/li-songze) | [GitHub](https://github.com/ybdai7/Backdoor-indicator-defense)

---

## What Is This Project?

BackdoorIndicator is a **server-side proactive defense** against backdoor attacks in Federated Learning (FL). In FL, malicious clients can secretly embed backdoor triggers into the shared global model while keeping their updates statistically similar to benign ones — making traditional defenses ineffective.

BackdoorIndicator solves this with a fundamentally different approach: instead of inspecting model parameters, the **server itself plants a controlled "indicator task"** using Out-of-Distribution (OOD) data as a watermark. Malicious clients performing backdoor injection will inadvertently maintain this watermark, exposing themselves to detection.

---

## Novel Contribution

### Key Intuition

The defense is built on two key observations:

1. **Backdoor samples are OOD** — they are out-of-distribution with respect to benign samples from the target class
2. **Subsequent backdoors maintain previous ones** — injecting a new backdoor task helps preserve a previously planted backdoor's accuracy

### How BackdoorIndicator Works

First, the server picks some random images from a completely different dataset — for example, if the main task is CIFAR-10 (everyday objects), the server uses CIFAR-100 images (different categories) as its secret watermark data.
 
Before sending the global model to clients, the server quietly trains it on these OOD images for a few extra steps. This plants a hidden "indicator task" inside the model — think of it like a secret stamp that only the server knows about.
 
The server also saves the internal statistics of the model at this point (called BN statistics — the mean and variance of each layer), so it can restore them later without the clients knowing.
 
Now the server sends this secretly stamped model out to all the clients for their normal local training round.
 
Each client trains the model on their own local data. Honest clients just do normal training and the secret stamp slowly fades away from their model. But malicious clients are also injecting their own backdoor — and doing that accidentally keeps the server's secret stamp alive in their model too.
 
When clients send their updated models back, the server restores those saved BN statistics on each received model, then checks how well each model still remembers the secret indicator task.
 
If a model still scores high on the indicator task, it means that client was doing backdoor training — because only backdoor injection keeps the stamp alive. The server flags that client as malicious and removes their update from the final aggregation.

### Why It's Novel

- Does **NOT** rely on statistical comparison of model parameters (unlike FLTrust, Krum, Deepsight)
- Works in both **IID and non-IID** data distributions
- **Agnostic to unknown backdoor types** — detects any backdoor that maintains OOD mappings
- Purely **server-side** — no client cooperation needed
- Handles **BN statistics shift** which previously undermined maintaining effects

---

## System Requirements

### Hardware
| Component | Requirement |
|-----------|-------------|
| GPU | CUDA-enabled (NVIDIA T4 or better recommended) |
| RAM | 12 GB+ recommended |
| Disk Space | ~5 GB (datasets + checkpoints) |
| Platform | Linux / Google Colab |
 
### Core Software
| Package | Version |
|---------|---------|
| Python | 3.7.15 |
| torch | 1.13.0 |
| torchvision | 0.14.0 |
 
### All Required Python Packages (`requirement.txt`)
| Package | Version |
|---------|---------|
| importlib-metadata | 6.7.0 |
| kmeans-pytorch | 0.3 |
| matplotlib | 3.5.3 |
| numpy | 1.21.6 |
| packaging | 23.2 
| requests | 2.31.0 |
| scikit-learn | 1.0.2 |
| scipy | 1.7.3 |
| torch | 1.13.0 |
| torch-kmeans | 0.2.0 |
| torchvision | 0.14.0 |
| urllib3 | 2.0.7 |
---
## How We Ran the Project (Google Colab)
### Step 1 — Clone & Install
```python
!git clone https://github.com/ybdai7/Backdoor-indicator-defense.git
%cd Backdoor-indicator-defense
!pip install -r requirement.txt -q
```
### Step 2 — Mount Google Drive (saves checkpoints permanently)
```python
from google.colab import drive
drive.mount('/content/drive')
!mkdir -p /content/drive/MyDrive/BackdoorIndicator/saved_models
!ln -s /content/drive/MyDrive/BackdoorIndicator/saved_models saved_models
```
### Step 3 — Download OOD Edge-Case Datasets from OOD_Federated_Learning Github link
```python
!git clone https://github.com/ksreenivasan/OOD_Federated_Learning.git /content/OOD_FL
!cp -r /content/OOD_FL/edge-case-examples/ data/
```
### Step 4 — Write the Config File 
```python
%%writefile Backdoor-indicator-defense/utils/yamls/indicator/params_vanilla_Indicator.yaml
resumed_model: false
save_on_round:
  - 100
  - 200
  - 300
 
ood_data_sample_lens: 100
ood_data_batch_size: 64
ood_data_source: CIFAR100
 
benign_lr: 0.1
benign_momentum: 0.9
benign_weight_decay: 0.0005
benign_is_projection_grad: false
benign_projection_norm: 3
benign_retrain_no_times: 1
 
poisoned_lr: 0.025
poisoned_momentum: 0.9
poisoned_weight_decay: 0.0005
poisoned_is_projection_grad: false
poisoned_projection_norm: 5
poisoned_retrain_no_times: 2
poisoned_start_round: 210
poisoned_end_round: 400
poisoned_round_interval: 1
 
adaptive_attack: false
adaptive_attack_round: 10
adaptive_attack_lr: 0.05
 
poison_task_name: "pixel pattern"
edge_case: false
semantic: true
pixel_pattern: false
 
poison_images_test:
  - 330
  - 3934
  - 12336
  - 30560
  - 30696
 
poison_images:
  - 568
  - 33105
  - 33615
  - 33907
  - 36848
  - 40713
  - 41706
 
poisoned_original_class: 1
poisoned_pattern_choose: 1
blend_alpha: 0.3
poison_label_swap: 2
poisoned_len: 7
poison_no_reuse: 10
poison_train_batch_size: 64
 
agg_method: FedProx
defense_method: Indicator
malicious_train_algo: Vanilla
Fedprox_mu: 0
watermarking_mu: 0.4
model_type: ResNet18
dataset: CIFAR10
class_num: 10
sample_dirichlet: true
dirichlet_alpha: 0.9
 
start_round: 1
end_round: 400
train_batch_size: 64
test_batch_size: 1000
no_of_total_participants: 30
no_of_participants_per_round: 5
no_of_adversaries: 1
 
norm_clip: false
fix_nc_bound: true
nc_bound: 2
eta: 0.5
 
global_retrain_no_times: 20
global_lr: 0.005
global_momentum: 0.9
global_weight_decay: 0.0005
global_is_projection_grad: false
global_projection_norm: 0.8
global_watermarking_start_round: 100
global_watermarking_end_round: 400
global_watermarking_round_interval: 1
 
global_milestones:
  - 5
  - 10
  - 20
  - 50
  - 100
global_lr_gamma: 0.8
 
malicious_milestones:
  - 2
  - 4
malicious_lr_gamma: 0.3
 
adaptive_malicious_milestones:
  - 10
  - 20
adaptive_malicious_lr_gamma: 1
 
benign_milestones:
  - 2
  - 4
benign_lr_gamma: 0.1
 
malicious_neurotoxin_ratio: 1
malicious_aggregate_all_layer: 1
 
show_local_test_log: false
show_train_log: false
VWM_detection_threshold: 95
replace_original_bn: true
```

### Step 5 — Phase 1: Train Clean Global Model
Disable attack and defense first to get a clean baseline checkpoint 
For this , we can set the poison start round to 10000, which is far beyond the training round of the model.
```python
# In your YAML temporarily set:
# poisoned_start_round: 10000
# global_watermarking_start_round: 10000

!python main.py --GPU_id "0" --params utils/yamls/params_vanilla_indicator.yaml
```
Checkpoints are saved to `saved_models/<timestamp>/`.

### Step 6 — Phase 2: Run Backresumed_model: false                  # Start training from scratch
For this, we can set the poison start round to 210, which is the training round for the model, while the clean model does not contain the malicious points.
```python
# In your YAML set:
# resumed_model: '<timestamp>/saved_model_global_model_300.pt.tar'
# poisoned_start_round: 210
# global_watermarking_start_round: 100
!python main.py --GPU_id "0" --params utils/yamls/indicator/params_vanilla_Indicator.yaml
```
### Step 7 — Run Other Experiment Variants
 
By simply changing the YAML file path in the command, we implemented 5–6 more experiments covering different attack methods and settings. Each YAML file inside `utils/yamls/` is pre-configured for a specific scenario:
 
```python
# Vanilla attack with No Defense (baseline — no protection)
!python main.py --GPU_id "0" --params utils/yamls/params_vanilla_indicator.yaml
 
# PGD (Projected Gradient Descent) attacker + Indicator defense
!python main.py --GPU_id "0" --params utils/yamls/indicator/params_pgd_Indicator.yaml
 
# Neurotoxin attacker + Indicator defense
!python main.py --GPU_id "0" --params utils/yamls/indicator/params_neurotoxin_Indicator.yaml
 
# Chameleon attacker + Indicator defense
!python main.py --GPU_id "0" --params utils/yamls/indicator/params_chameleon_Indicator.yaml
 
# Vanilla attacker + FLTrust defense (comparison baseline)
!python main.py --GPU_id "0" --params utils/yamls/rflbat/params_vanilla_rflbat.yaml
 
# Vanilla attacker + Flame defense (comparison baseline)
!python main.py --GPU_id "0" --params utils/yamls/flame/params_vanilla_flame.yaml
 
# Vanilla attacker + Deepsight defense (comparison baseline)
!python main.py --GPU_id "0" --params utils/yamls/deepsight/params_vanilla_deepsight.yaml
```
 
---
## Config Parameters Explained

| Parameter | Our Value | Paper Value | Purpose |
|-----------|-----------|-------------|---------|
| `end_round` | 400 | 1900 | Total FL training rounds |
| `no_of_total_participants` | 30 | 100 | Total federated clients |
| `no_of_participants_per_round` | 5 | 10 | Clients sampled per round |
| `global_retrain_no_times` | 20 | 200 | Server OOD retraining steps |
| `poisoned_retrain_no_times` | 2 | 10 | Attacker local steps |
| `ood_data_sample_lens` | 100 | 800 | OOD watermark sample count |
| `global_watermarking_start_round` | 100 | 1100 | When defense activates |
| `poisoned_start_round` | 210 | 1210 | When attacker starts |
| `dataset` | CIFAR10 | CIFAR10/EMNIST | Main task dataset |
| `ood_data_source` | CIFAR100 | CIFAR100 | OOD watermark source |
| `defense_method` | Indicator | Indicator | Defense algorithm |
| `agg_method` | FedProx | FedProx | Aggregation method |

> We reduced these values to fit Google Colab's T4 GPU time limits (~40 min per 100 rounds instead of 1.5 hrs).

---

## Experiments & Results

### Evaluation Metrics

| Metric | Meaning | Good Result |
|--------|---------|-------------|
| **TPR** (True Positive Rate) | % malicious clients correctly detected | 99.2 |
| **FPR** (False Positive Rate) | % benign clients wrongly flagged | 15.7 |
| **BA** (Backdoor Accuracy) | Backdoor success rate after defense | 15.9 |
| **Main Task Accuracy** | Accuracy on clean test data | 84.7 |

### Attack Types Tested

- **Vanilla** — Standard model replacement attack
- **PGD** — Projected Gradient Descent constrained attack
- **Neurotoxin** — Gradient-masked persistent backdoor
- **Chameleon** — Adaptive attack mimicking benign updates
  
### Backdoor Types
- **Pixel Pattern** — Small pixel trigger in corner of image
- **Blend** — Alpha-blended trigger pattern across image
- **Edge Case (Semantic)** — Natural images as backdoor

### Paper Tables Reproduced

| Table | Dataset | Setting |
|-------|---------|---------|
| Table 13 | CIFAR-100 | Single client attack, IID, all 4 attackers × 3 backdoor types |
| Table 14 | EMNIST | Single client attack, IID, all 4 attackers |
| Table 15 | CIFAR-100 | Non-IID, varying alpha ∈ {0.2, 0.5, 0.9} and plr ∈ {0.01, 0.03, 0.05} |
| Table 16 | EMNIST | Non-IID, same as Table 15 |

---

## Project Structure

```
Backdoor-indicator-defense/
│
├── main.py                          # Entry point — FL training loop
├── requirement.txt                  # Python dependencies
│
├── participants/                    # Client implementations
│   ├── benign_participant.py        # Honest client training
│   └── malicious_participant.py    # Backdoor attack logic
│
├── utils/
│   └── yamls/
│       ├── params_vanilla_indicator.yaml       # Base config (clean training)
│       └── indicator/
│           └── params_vanilla_Indicator.yaml   # BackdoorIndicator defense config
│
├── data/                            # Datasets (auto-downloaded)
│   └── edge-case-examples/          # OOD semantic backdoor images
│
└── saved_models/                    # Checkpoints saved here at runtime
```

---

## Common Issues & Fixes

| Problem | Fix |
|---------|-----|
| Colab disconnects mid-run | Mount Google Drive and symlink `saved_models` to Drive |
| CUDA out of memory | Reduce `train_batch_size` to 32 |
| OOD data not found | Run: `cp -r OOD_Federated_Learning/edge-case-examples/ data/` |
| `ModuleNotFoundError` | Run: `pip install <missing_package>` |
| YAML `True`/`False` error | Use lowercase `true`/`false` in YAML |
| Training too slow | Reduce `end_round`, `no_of_total_participants`, `global_retrain_no_times` |
| Checkpoint not found | Check `saved_models/` and copy exact path to `resumed_model` in YAML |

---
## Keep Colab Alive

Paste this in your browser console (F12 → Console) to prevent session timeout:

```javascript
function KeepAlive() {
  document.querySelector("#top-toolbar > colab-connect-button")
    .shadowRoot.querySelector("#connect").click();
}
setInterval(KeepAlive, 60000);
```
---
## Contributors

- **Princi Mantri** — 2023UCP1601  
- **Priyang** — 2023UCP1681
  
## References

- **Paper:** Yicong Dai et al., *BackdoorIndicator: Leveraging OOD Data for Proactive Backdoor Detection in Federated Learning*, USENIX Security 2024
- **Code:** https://github.com/ybdai7/Backdoor-indicator-defense
- **OOD Dataset:** https://github.com/ksreenivasan/OOD_Federated_Learning
- **Full PDF:** https://usenix.org/system/files/usenixsecurity24-dai-yicong.pdf
