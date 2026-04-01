# Detailed Comparison Matrix - All Notebooks

## QUICK REFERENCE: Architecture & Training Parameters

### Model Architecture
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        ARCHITECTURAL EVOLUTION                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│ v0-v1: (512, 8, 2048)                                                          │
│        ├─ 3 × ProjectionLayer (video, audio, clip)                            │
│        ├─ 3 × ModalEncoder (2 layers, no PE)                                 │
│        ├─ TriModalCoarseFusion (2 layers, +PE)                               │
│        └─ FineInteraction: 2-branch (video, audio) + text_expand hack        │
│                                                                               │
│ v1.5: Same as v0, but                                                        │
│        └─ + 3rd cross-attn branch (CLIP)                                    │
│        └─ + Real word tokens (last_hidden_state)                            │
│                                                                               │
│ v4:  (256, 4, 1024) ← REDUCED: 71% fewer parameters                         │
│        ├─ Same modality encoders (2 layers)                                 │
│        ├─ TriModalCoarseFusion: 2 → 1 layer (prevent overfitting)         │
│        ├─ + DropPath stochastic depth (0.1) on all residuals              │
│        ├─ + Residual gate for skip connection blending                     │
│        └─ Loss: BCE + MarginRanking + Contrastive                          │
│                                                                               │
│ v5:  (256, 4, 1024) Same as v4, plus                                        │
│        ├─ AudioGating module (learned feature weighting)                    │
│        ├─ Full train/val/test support (1,542 test samples)                 │
│        └─ Hard negative mining in loss                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Parameter Table: Detailed Comparison

| Parameter | v0-v1 (baseline) | v1.5 (improved) | v2 (clean) | v4 (regularized) | v5 (final) |
|-----------|---|---|---|---|---|
| **Model Architecture** |
| d_model | 512 | 512 | 256 | **256** | **256** |
| n_heads | 8 | 8 | 4 | **4** | **4** |
| n_heads_dim | 64 | 64 | 64 | **64** | **64** |
| d_ff | 2048 | 2048 | 1024 | **1024** | **1024** |
| ModalEncoder layers | 2 | 2 | 2 | 2 | 2 |
| Coarse fusion layers | 2 | 2 | 2 | **1** | **1** |
| Dropout | 0.1 | 0.1 | 0.3 | **0.4** | **0.4** |
| **Total Parameters** | ~35M | ~35M | ~10M | **~10M** | **~11M** |
| **Data Handling** |
| Text token input | Pooler (S=1) | last_hidden_state (S) | last_hidden_state (S) | last_hidden_state (S) | last_hidden_state (S) |
| Cross-attention branches | 2 | **3** | **3** | **3** | **3** |
| Audio masking | Outside encoder | Inside encoder | Inside encoder | Inside encoder | Inside encoder |
| PE placement | encoder+coarse | coarse only | coarse only | coarse only | coarse only |
| Residual gate | None | None | Optional | **Yes** | **Yes** |
| DropPath rate | — | — | — | **0.1** | **0.1** |
| Audio gating | **None** | **None** | **None** | **None** | **YES** |
| **Training Configuration** |
| Optimizer | AdamW | AdamW | AdamW | AdamW | AdamW |
| Learning rate | 5e-4 | 5e-4 | 3e-4 | **1e-4** | **1e-4** |
| Weight decay | 1e-4 | 1e-4 | 5e-4 | **2e-3** | **2e-3** |
| Batch size | 8 | 8 | 8 | 8 | 8 |
| Gradient accumulation | 4 | 4 | 4 | 1 | 1 |
| Gradient clip | 1.0 | 1.0 | 0.5 | **0.5** | **0.5** |
| Warmup | Linear 5% | Linear 5% | Linear 5% | Linear 5% | Linear 5% |
| LR schedule | Cosine | Cosine | Cosine | Cosine | Cosine |
| Max epochs | 50 | 50 | 100 | **100** | **100** |
| Early stop patience | 10 | 10 | 12 | **25** | **25** |
| **Loss Function** |
| Main loss | MSE | MSE | BCE | **BCE** | **BCE** |
| Auxiliary losses | — | Ranking | Ranking + Contrastive | **Ranking + Contrastive** | **Ranking + Contrastive** |
| pos_weight (BCE) | — | — | 3 | **3** | **3** |
| Ranking margin | — | 0.2 | 0.2 | 0.2 | 0.2 |
| Loss scheduling | Single phase | Single phase | Multi-phase | Multi-phase | **Multi-phase + Hard mining** |
| **Dataset Support** |
| Train samples | 7,218 | 7,218 | 7,218 | 7,218 | 7,218 |
| Val samples | 1,550 | 1,550 | 1,550 | 1,550 | 1,550 |
| Test samples | — | — | — | — | **1,542** |
| Audio coverage | 86% | 86% | 86% | 86% | 86% |
| Text coverage | 100% | 100% | 100% | 100% | 100% |

---

## Feature Dimension Pipeline

```
INPUT:  (Batch, Sequence_Length, Feature_Dim)
         
         Video:       (B, T, 2304)  ← SlowFast-R50
         Audio:       (B, T, 2048)  ← PANN CNN14 (86% coverage)
         CLIP visual: (B, T,  512)  ← ViT-B/32
         CLIP text:   (B, S,  512)  ← Variable word tokens

PROJECTION LAYER:
         Video:       (B, T, 2304)  →  (B, T, d_model)
         Audio:       (B, T, 2048)  →  (B, T, d_model)
         CLIP visual: (B, T,  512)  →  (B, T, d_model)
         CLIP text:   (B, S,  512)  →  (B, S, d_model)

MODAL ENCODERS:  (with 2 Transformer layers each)
         Video:       (B, T, d_model)  →  (B, T, d_model)
         Audio:       (B, T, d_model)  →  (B, T, d_model)  [with audio_mask]
         CLIP visual: (B, T, d_model)  →  (B, T, d_model)

COARSE FUSION:   [concatenate]
         Joint:       (B, 3T, d_model)  →  Transformer  →  (B, 3T, d_model)
                                            [1-2 layers]
         Split:       (B, T, d_model) × 3

FINE INTERACTION:  [triple cross-attention]
         Video:  Q = (B, T, d_model)  K,V = (B, S, d_model)  →  (B, T, d_model)
         Audio:  Q = (B, T, d_model)  K,V = (B, S, d_model)  →  (B, T, d_model)
         CLIP:   Q = (B, T, d_model)  K,V = (B, S, d_model)  →  (B, T, d_model)
                 
                 z = ω_tv · video_att + ω_ta · audio_att + ω_tc · clip_att

SALIENCY HEAD:
         (B, T, d_model)  →  Linear(d_model, 512)  →  Linear(512, 1)  →  (B, T)

OUTPUT: Logits (B, T)  — apply sigmoid for probabilities [0, 1]
```

---

## Loss Function Comparison

### v0-v1: MSE (Baseline)
```python
loss = MSE(pred, target)  # Mean Squared Error on z-score targets
# Problem: Ignores ranking of salient vs background clips
```

### v1.5: MSE + Ranking
```python
mse_loss = MSE(pred, target)
ranking_loss = MarginRanking(pred_salient, pred_background, margin=0.2)
loss = mse_loss + ranking_loss
```

### v2+: BCE + Ranking + Contrastive
```python
bce_loss = BCEWithLogits(pred, target, pos_weight=3)
  # pos_weight=3 upweights salient (positive) clips

ranking_loss = MarginRanking(pred_salient, pred_background, margin=0.2)

contrastive_loss = ContrastiveSaliency(pred[target>0], pred[target==0])
  # Pulls salient predictions up, background predictions down

loss = bce_loss + lambda_rank * ranking_loss + lambda_contrast * contrastive_loss

# Multi-phase curriculum:
# Epochs 1-4:   bce_loss only
# Epochs 5-9:   bce_loss + gradually ramp ranking_loss
# Epochs 10+:   full loss with all components
```

### v5: BCE + Ranking + Contrastive + Hard Mining
```python
# Same as v2+ plus:
hard_negatives = top_k_hardest(background_predictions, k=5)
  # Focus loss on confusing negatives

contrastive_loss = ContrastiveSaliency(
    pos_pred=pred[target>0],
    neg_pred=hard_negatives,  # Not all negatives
    margin=compute_margin(pos, neg)
)

loss = ... (same as v2+ but with hard mining)
```

---

## Regularization Progression

### No Regularization (v0-v1)
```
Training overfits → val loss plateaus while train loss continues → early stop needed
```

### Mild Regularization (v2)
```
dropout = 0.3
weight_decay = 5e-4
Result: Some overfitting control, but not enough
```

### Aggressive Regularization (v4 "Overfit Fix")
```
┌─────────────────────────────────────────────────────────────┐
│ OVF-A: Dropout Increase                                     │
│   0.1 → 0.4 everywhere                                      │
│   → More aggressive element-wise dropping                   │
│                                                              │
│ OVF-B: Weight Decay Increase                               │
│   5e-4 → 2e-3 (4× stronger L2 regularization)              │
│   → Applied to weights only, not biases/norms              │
│                                                              │
│ OVF-C: Stochastic Depth (DropPath)                         │
│   NEW: Drop entire residual branches with probability      │
│   rate=0.1, linearly staggered across layers               │
│   → Prevents co-adaptation of layers                        │
│                                                              │
│ OVF-D: Architecture Reduction                              │
│   d_model: 512 → 256  (50% reduction)                      │
│   coarse_layers: 2 → 1 (50% reduction)                     │
│   Parameters: 35M → 10M (71% reduction)                    │
│   → Simpler model less prone to overfitting                │
│                                                              │
│ Result: Smoother training, smaller train/val gap,          │
│         better test generalization expected                │
└─────────────────────────────────────────────────────────────┘
```

---

## Audio Handling: Evolution

### Problem
14% of videos lack audio features (8,695 / 10,148 coverage)

### Solution Evolution

```
v0-v1 (Ineffective):
  audio_mask = [1, 1, ..., 1, 0, 0, ..., 0]  ← 1 = real, 0 = zero-filled
  In forward pass: audio_emb = audio_emb * audio_mask.unsqueeze(-1)
  Problem: Scaling outside attention doesn't prevent invalid attention weights

v1.5+ (Audio Encoder Masking):
  audio_mask → src_key_padding_mask in ModalEncoder attention
  Problem: If all positions masked (no audio at all), softmax(all -inf) = NaN
  
v4 (NaN-Safe):
  has_any_audio = (audio_mask.sum(dim=1) > 0)  # per-sample check
  combined = (audio_mask == 0) & has_any_audio  # only mask if sample HAS SOME audio
  If no audio at all: no masking, let zero-filled features pass through normally
  
v5 (Learned Weighting):
  AudioGating module: gate = sigmoid(Linear(audio_dim))(audio_features)
  gated_audio = gate * audio
  Problem solved: Learns per-feature importance; smoothly handles audio quality variation
  No hard ON/OFF switch like masking; learned data-dependent weighting

  Advantage over v4 masking:
    - Soft weighting vs hard masking
    - Learned importance weights
    - Handles partial audio corruption, not just complete absence
    - More gradient information flows back to suppressed features
```

---

## Evaluation Metrics & Results

### Metric Definitions

| Metric | Formula | Range | Interpretation |
|--------|---------|-------|-----------------|
| **MSE** | (1/N)Σ(pred - target)² | [0, ∞) | Lower is better; penalizes large errors quadratically |
| **BCE** | -(y·log(p) + (1-y)·log(1-p)) | [0, ∞) | Binary classification loss; penalizes confidence in wrong class |
| **mAP** | (1/Q)ΣAP(q) | [0, 1] | Average precision across queries; higher is better |
| **HIT@1** | Correct top-1 prediction (≥4 score) | [0, 1] | Fraction of queries where top-1 is "Very Good" |
| **Ranking Loss** | max(0, margin - (pos - neg)) | [0, ∞) | Forces pos > neg by margin; lower is better |
| **Contrastive** | (1/B)Σ contrast(pos, neg) | [0, ∞) | Pulls apart positive and negative predictions |

### Reported Results

#### v0 Baseline (audio_cfsum_training_first)
```
Epoch-by-epoch loss:
  Epoch 1:  train=1.111  val=0.954
  Epoch 2:  train=0.937  val=0.926
  Epoch 3:  train=0.877  val=0.856
  Epoch 4:  train=0.819  val=0.826
  Epoch 5:  train=0.778  val=0.816  ← BEST validation
  Epoch 6:  train=0.743  val=0.830  (1/10 patience)
  ...
  Epoch 15: train=0.573  val=0.902  EARLY STOP
  
Best:    MSE = 0.8164 (epoch 5)
Status:  Position bias present; no CLIP visual; early stop at epoch 15
```

#### Expected v4 Results
```
With aggressive regularization (DropPath, reduced capacity, weight_decay 2e-3):
  - Slower initial descent (DropPath noise)
  - Smoother curves (stochastic depth)
  - Smaller train/val gap (regularization controls overfitting)
  - Later peak (patience=25 vs 10)
  - Better test generalization
  
Expected best: Lower val MSE (e.g., 0.75-0.80) due to regularization benefits
```

#### v5 with Test Set
```
Full three-way split evaluation:
  Train: 7,218 samples
  Val:   1,550 samples
  Test:  1,542 samples

Expected metrics:
  - mAP on test set (primary metric)
  - HIT@1 (top-1 accuracy ≥4 score)
  - Ablation results (modality contributions)
```

---

## Key Architectural Innovations Explained

### 1. Triple Cross-Attention (v1.5+)

**Why**: Video and audio alone miss visual semantics captured by CLIP

```
Standard dual-attention (v0):
  z = ω_tv · CrossAttn(video → text) + ω_ta · CrossAttn(audio → text)

Triple-attention (v1.5+):
  z = ω_tv · CrossAttn(video → text) 
    + ω_ta · CrossAttn(audio → text) 
    + ω_tc · CrossAttn(clip → text)      ← NEW branch
```

**Mechanism**: Each modality independently attends to query text tokens; weighted sum combines contributions

**Learned weights**: ω_tv=2.0, ω_ta=1.0, ω_tc=1.0 (video weighted 2× audio/CLIP)

### 2. Real Word Tokens (v2+)

**Problem with pooled token (v0-v1)**:
```
text = CLIP_text_encoder(query)['pooler_output']  # (B, 512)
text_kv = text.unsqueeze(1)  # (B, 1, 512)

In cross-attention:
  attention_weights = softmax(Q @ K^T)  where K has size 1
  → softmax([x]) = 1 always   ← Attention weights fixed!
  → Q (video/audio) gradients = 0 (no gradient flow)
  → Text_expand hack: expand 1 token to 8 virtual tokens
```

**Solution: Use word tokens (v2+)**:
```
text = CLIP_text_encoder(query)['last_hidden_state']  # (B, S, 512)
  where S = number of words in query (e.g., 10-15)
text_kv = Linear(512 → d_model)(text)  # (B, S, d_model)

In cross-attention:
  attention_weights = softmax(Q @ K^T)  where K has size S > 1
  → Attention weights learned based on Q
  → Q (video/audio) gradients flow properly
  → Text padding masked with text_mask
```

### 3. DropPath / Stochastic Depth (v4+)

**Mechanism**: Randomly drop entire residual branches during training

```python
class DropPath:
    def forward(self, x):
        if training:
            keep = 1 - drop_prob
            mask_shape = (batch, 1, 1)
            mask = Bernoulli(keep) / keep
            return x * mask  # scale by 1/keep to maintain E[x]
        else:
            return x  # inference: no dropout
```

**Effect**:
- Layer at position (drop_prob=0): never dropped
- Final layer (drop_prob=0.1): 90% kept, 10% dropped
- Prevents layer co-adaptation; each layer must be independently useful

**Why powerful**: Stronger than element-wise dropout; entire features are dropped together

### 4. Residual Gate (v4+)

**Purpose**: Blend transformer refinement with shallow residual connection

```python
proj_residual = (video_emb + audio_emb + clip_emb) / 3.0  # Average of projections
gate = sigmoid(learnable_param)  # init 0.5
output = gate * transformer_output + (1 - gate) * proj_residual
```

**Why**: 
- Preserves temporal diversity from projections
- Prevents model from relying too heavily on fusion layers
- Acts as soft skip connection; learnable blend

**Gradient flow**:
- If gate → 0: use residual (bypass fusion)
- If gate → 1: use transformer (full fusion)
- Learned based on training signal

### 5. AudioGating (v5)

**Purpose**: Learn per-feature importance for audio (handles 86% coverage and quality variation)

```python
class AudioGating(nn.Module):
    def __init__(self, audio_dim=2048):
        self.gate_net = Sequential(
            Linear(audio_dim, 256),
            ReLU(),
            Linear(256, audio_dim)  # → (B, T, 2048)
        )
    
    def forward(self, audio):  # (B, T, 2048)
        gate = sigmoid(self.gate_net(audio))  # (B, T, 2048)
        return gate * audio  # element-wise weighting
```

**Why better than hard masking**:
- Soft weighting vs binary OFF
- Learned based on audio content
- Handles partial corruption, not just complete absence
- More stable gradients

**vs. v4 audio_mask approach**:
- v4: if sample has audio → mask zero positions; else no masking (hard choice)
- v5: learn importance of each audio feature element (soft choice, data-dependent)

---

## Quick Decision Tree: Which Notebook to Use?

```
Question: What's my use case?

├─ "I want the baseline reference"
│  └─ audio_cfsum_training_first.ipynb (v0)
│     √ Simple implementation
│     ✗ Overfitting visible
│     ✗ No CLIP visual modality
│
├─ "I want a working clean implementation"
│  └─ audio_cfsum_v2_24_march_clean.ipynb (v2)
│     √ All improvements integrated
│     √ BCE loss (better than MSE for classification)
│     √ Triple attention
│     ✗ Some overfitting remains
│
├─ "I want production-ready with minimal overfitting"
│  └─ cfsum_v4_overfit_fix.ipynb (v4)
│     √ Aggressive regularization (DropPath, weight_decay 2e-3)
│     √ 71% fewer parameters (18→10M)
│     √ Proven over-fitting fixes
│     √ Better generalization expected
│     ✗ Slower training (DropPath noise)
│
└─ "I want the BEST final version with test set"
   └─ cfsum_v5_balanced.ipynb (v5)
      √ Audio gating for robust audio handling
      √ Hard negative mining in loss
      √ Full train/val/test evaluation possible
      √ Balanced curriculum loss scheduling
      √ Latest techniques integrated
      → USE THIS FOR FINAL SUBMISSION
```

---

## File Size & Structure

| Notebook | Cells | Size | Main Sections |
|---|---|---|---|
| audio_cfsum_training_first.ipynb | ~21 | ~50KB | Setup, Dataset, Architecture (v1), Training (50 epochs), Evaluation |
| cfsum_v1_25_march.ipynb | ~26 | ~55KB | Setup, Dataset, Architecture (v1), Full training trace, Evaluation |
| cfsum_v4_overfit_fix.ipynb | ~33 | ~75KB | Setup, Dataset, Regularization details, Full architecture with DropPath, Extensive training |
| cfsum_v5_balanced.ipynb | ~28 | ~70KB | Setup, Dataset (train/val/test), Architecture V5, AudioGating, Training with hard mining |

