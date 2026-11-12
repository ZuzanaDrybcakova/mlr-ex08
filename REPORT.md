REPORT TASK 4 - LEAF CLASSIFIER OPTIMIZATION
Zuzana Drybcakova

## 1. Experimental Setup & Baseline
- **Baseline:** Custom CNN with Batch Normalization (`LeafNetBN`, ~209k parameters) from Task 3, trained from scratch for 15 epochs with standard geometric augmentation, Adam (`lr = 1e-3`), batch size 32, and `CrossEntropyLoss`.
- **Camera-like Validation Set:** Created once from the 62 validation images by applying a fixed camera-like transformation pipeline (`RandomResizedCrop(96, scale=(0.6, 1.0))`, `RandomRotation(20)`, and `ColorJitter(0.3, 0.3, 0.3, 0.1)`) 3 times per image under a fixed seed (`seed = 42`), resulting in 186 evaluation images. This tests robustness against real-world mobile camera conditions (lighting, framing, and angle variations).

## 2. Ablation Study Results (Seed 42)

| Variant | Val Acc (%) | Camera-like Acc (%) | What changed |
| :--- | :---: | :---: | :--- |
| **Baseline (Task 3)** | 67.7% | 51.6% | Unmodified Task 3 network (`LeafNetBN`) trained from scratch |
| **A1: + ColorJitter** | 67.7% | 61.8% | Baseline + color jitter alongside geometric augmentation |
| **A2: Pretrained (Frozen)** | 77.4% | 70.4% | Pretrained MobileNetV3-small (`features` frozen, 2-layer classifier trained) |
| **A3: Pretrained (Fine-tuned)** | 87.1% | 78.0% | End-to-end fine-tuning of MobileNetV3-small (`lr = 1e-4`) |
| **A4: A3 + AdamW + Cosine LR** | 72.6% | 77.4% | Fine-tuned MobileNetV3-small with AdamW + CosineAnnealingLR (40 epochs) |
| **A5a: + Dropout (0.3)** | 62.9% | 52.2% | Baseline + 30% dropout regularization before classifier |
| **A5b: + Best Epoch** | 67.7% | 51.6% | Baseline + saving weights of the best validation epoch |
| **Combo (A1 + A4 + A5b)** | 79.0% | **83.3%** | Fine-tuned MobileNetV3-small + ColorJitter + AdamW + Cosine LR (40 ep) + Best Epoch |

### Multi-Seed Variance Check (Seeds 42, 123, 999)
To ensure improvements exceed random training noise on a small validation set, the baseline and top models were evaluated across three seeds on the camera-like validation set:
- **Baseline:** 52.9% ± 1.1%
- **A3 (Pretrained Fine-tune):** 78.7% ± 2.8% (+25.8 percentage points over baseline)
- **Combo Winner:** **83.7% ± 1.1%** (+30.8 percentage points over baseline)

### Historical Dataset Experiment (Last Year's Photographs)
- Adding 700 historical training images significantly boosted the scratch-trained baseline on the camera-like set (from **51.6% to 63.4%**).
- For the already pretrained **Combo** model, adding the extra historical data did not improve camera-like accuracy on our validation set (**82.8%** vs. **83.3%** without it; plain validation remained **79.0%** for both), as pretrained features already generalized well.
- Evaluated on the completely unseen historical validation set (120 images from other students/phones), the models that never saw historical training data ranked identically to our camera-like benchmark: **Baseline 45.8%**, **Pretrained Frozen 50.0%**, and **Combo 63.3%**.

## 3. Key Findings & Analysis
- **What helped most:**
  1. **Transfer Learning (Pretrained MobileNetV3-small):** Provided the single largest accuracy jump (+18.8 p.p. frozen, +26.4 p.p. fine-tuned on the camera-like set) because pretrained ImageNet features generalize far better on small datasets than a network trained from scratch.
  2. **Color Augmentation (`ColorJitter`):** Increased camera-like accuracy by **+10.2 percentage points** (51.6% → 61.8%) while leaving standard validation accuracy unchanged (67.7%). Although run on a single seed, this +10.2 p.p. jump far exceeds the baseline's ±1.1 p.p. seed spread and indicates that color augmentation specifically addresses lighting shifts typical of live phone cameras.
  3. **Combined Recipe (`Combo`):** While switching to AdamW + Cosine Annealing over 40 epochs (`A4`) on its own did not beat `A3` at seed 42 (77.4% vs. 78.0% camera-like), combining it with `ColorJitter` and best-epoch checkpointing pushed camera-like performance to **83.3%** (83.7% ± 1.1% across 3 seeds).
- **Noise vs. Actual Improvement:** The average standard deviation across different random seeds on the camera-like set was **~1.7 percentage points** (±1.1 p.p. for the winning combo). Since our gains are **+25.8 to +30.8 percentage points**, the improvements are vastly larger than random initialization noise. Re-running the baseline with seed 42 confirmed exact deterministic reproducibility.
- **What did not help:**
  - **Dropout (0.3):** Decreased standard validation accuracy (67.7% → 62.9%) and yielded virtually no gain on the camera-like set (52.2%), likely because the small custom CNN (`LeafNetBN`) was already capacity-constrained rather than overfitting.
  - **Best-epoch checkpointing (alone):** Had no effect on the baseline because the final epoch (epoch 15) was already the best-performing one.

## 4. Model Selected for the Phone & Verification
- **Selected Model:** **Combo (`A1 + A4 + A5b`)** — Pretrained MobileNetV3-small fine-tuned with geometric + color augmentation, AdamW, CosineAnnealingLR (40 epochs), and best-epoch tracking.
- **Justification:** Among all models trained strictly on our dataset, it achieved the highest robustness on both the simulated camera benchmark (**83.7% ± 1.1%**) and the unseen cross-phone historical validation set (**63.3%**), while keeping a compact footprint (**6.4 MB** ONNX file) suitable for fast browser inference.
- **ONNX Export & Verification:** Exported to `leaf_model_task4.onnx` (input `[1, 3, 96, 96]`, output logits `[1, 4]`, verified against PyTorch outputs with max difference `6.8e-06`) and tested with the course camera app.

---
*Full code, cell outputs, and raw experimental logs are available in `ex08_notebook.ipynb` and `task4_results.csv` in this repository.*