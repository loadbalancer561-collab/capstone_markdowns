# Audio-Aware Coarse-to-Fine Fusion for Video Summarization (v4 — Overfit Fix)

Tri-modal CFSum with **SlowFast + PANN CNN14 + CLIP** on QVHighlights.

---

## 1. Architecture Overview

```
Input Features:
  SlowFast (2304-d) ─→ ProjectionLayer ─→ ModalEncoder (2 layers) ──┐
  PANN     (2048-d) ─→ ProjectionLayer ─→ ModalEncoder (2 layers) ──┼→ TriModalCoarseFusion (1 layer)
  CLIP     ( 512-d) ─→ ProjectionLayer ─→ ModalEncoder (2 layers) ──┘          │
                                                                                ▼
                                                                    TriFineInteraction
                                                                         ↑
                                                              CLIP text (S, 512)
                                                                         │
                                                                         ▼
                                                                   SaliencyHead
                                                                         │
                                                                    Output: (B, T)
```

### 1.1 ProjectionLayer
Two-layer MLP that maps each modality's raw features into a shared `d_model=256` space.

```
Linear(input_dim, 256) → LayerNorm(256) → ReLU → Dropout(0.4) → Linear(256, 256)
```

Three instances: `video_proj`, `audio_proj`, `clip_proj`.

### 1.2 ModalEncoder
Per-modality Transformer encoder with sinusoidal positional encoding.

| Parameter | Value |
|---|---|
| `d_model` | 256 |
| `n_heads` | 4 (head_dim = 64) |
| `d_ff` | 1024 |
| `n_layers` | 2 |
| `dropout` | 0.4 |
| `drop_path_rate` | 0.1 (linearly staggered: layer 0 = 0.0, layer 1 = 0.1) |

Each layer is a `TransformerLayerWithDropPath`:
```
x → MultiheadAttention(x, x, x) → DropPath → Add & LayerNorm
  → FFN(Linear→GELU→Dropout→Linear→Dropout) → DropPath → Add & LayerNorm
```

**Audio Encoder** additionally accepts `audio_mask` as `src_key_padding_mask` to handle samples with no audio (zero-filled features are masked from attention).

Three instances: `video_encoder`, `audio_encoder`, `clip_encoder`.

### 1.3 TriModalCoarseFusion
Joint cross-modal Transformer over concatenated modality tokens.

| Parameter | Value |
|---|---|
| `n_layers` | **1** (reduced from 2 in v3 to prevent overfitting) |
| Learnable modal embeddings | 3 × 256 (video=0, audio=1, clip=2) |

```
Input:  [V + vid_tok ; A + aud_tok ; C + clp_tok]  →  (B, 3T, 256)
        → 1× TransformerLayerWithDropPath
        → LayerNorm
Output: split back → V_fused (B,T,256), A_fused (B,T,256), C_fused (B,T,256)
```

### 1.4 TriFineInteraction
Triple cross-attention from each modality to text word tokens.

```
text_kv = Linear(512 → 256)(global_clip)       # Project CLIP text tokens

tv_att = CrossAttention(V_fused → text_kv)      # Video queries text
ta_att = CrossAttention(A_fused → text_kv)      # Audio queries text
tc_att = CrossAttention(C_fused → text_kv)      # CLIP queries text

z = ω_tv · tv_att + ω_ta · ta_att + ω_tc · tc_att
z = LayerNorm(z)
z = LayerNorm(z + FFN(z))
```

| Learnable Weight | Init Value |
|---|---|
| `omega_tv` (video) | **2.0** |
| `omega_ta` (audio) | 1.0 |
| `omega_tc` (CLIP) | 1.0 |

FFN: `Linear(256, 1024) → GELU → Dropout(0.4) → Linear(1024, 256) → Dropout(0.4)`

### 1.5 Residual Gate
Blends the refined output with a shortcut from the projection layer:

```
proj_residual = (video_emb + audio_emb + clip_emb) / 3.0
gate = sigmoid(residual_gate)        # init = 0.5
output = gate · refined + (1 - gate) · proj_residual
```

### 1.6 SaliencyHead
```
Linear(256, 128) → ReLU → Dropout(0.4) → Linear(128, 1) → squeeze
```

Output: raw logits `(B, T)` — sigmoid applied externally.

---

## 2. Model Configuration

| Parameter | Value | Notes |
|---|---|---|
| `video_dim` | 2304 | SlowFast features |
| `audio_dim` | 2048 | PANN CNN14 features |
| `clip_dim` | 512 | CLIP visual features |
| `d_model` | 256 | Shared hidden dimension |
| `n_heads` | 4 | head_dim = 64 |
| `d_ff` | 1024 | FFN intermediate dim |
| `dropout` | **0.4** | OVF-A: increased from 0.3 |
| `n_encoder_layers` | 2 | Per-modality encoder depth |
| `n_fusion_layers` | **1** | OVF-D: reduced from 2 |
| `drop_path_rate` | **0.1** | OVF-C: stochastic depth |
| **Total parameters** | **~9.2M** | |

---

## 3. Loss Functions

### Mode: `'full'` (BCE + Ranking + Contrastive)

```
total_loss = 1.0 × BCE + 0.1 × rank_gate × Ranking + 0.1 × contrast_gate × Contrastive
```

### 3.1 BCE Loss (Primary)
Binary Cross-Entropy with logits. Provides base regression signal.

```python
F.binary_cross_entropy_with_logits(pred, target, pos_weight=3.0)
```

- **pos_weight = 3.0**: Upweights positive clips (saliency > 0) to address class imbalance
- Applied with padding mask

### 3.2 Margin Ranking Loss (Auxiliary)
Pairwise margin ranking with hard negative mining.

```python
ranking = relu(margin - (pred_i - pred_j) × sign(target_i - target_j))
```

| Parameter | Value |
|---|---|
| `margin` | 0.2 |
| `hard_neg_k` | 5 (only top-5 hardest violated pairs per anchor) |
| `lambda_rank` | 0.1 |
| Warmup | Ramps in from **epoch 5** over 10 epochs |

### 3.3 Contrastive Saliency Loss (Auxiliary)
Forces `max(relevant_scores) > max(irrelevant_scores) + margin`.

```python
loss = relu(margin - (max_relevant - max_irrelevant))
```

| Parameter | Value |
|---|---|
| `margin` | 0.2 |
| `lambda_contrast` | 0.1 |
| Warmup | Ramps in from **epoch 10** over 10 epochs |
| Target normalization | Per-video z-score before splitting relevant/irrelevant |

Relevant clips: z-score > 0.5 | Irrelevant clips: z-score < -0.5

### 3.4 Warmup Gating (IMP-C)
Auxiliary losses are phased in gradually to prevent early instability:

```
rank_gate     = clamp((epoch - 5) / 10,  0, 1)    # 0 at epoch 5 → 1.0 at epoch 15
contrast_gate = clamp((epoch - 10) / 10, 0, 1)    # 0 at epoch 10 → 1.0 at epoch 20
```

---

## 4. Saliency Target Construction

```python
saliency[clip_idx] = (mean(annotator_scores) - 1.0) / 4.0
# Maps [1, 5] → [0, 1]
```

- Uses **mean** of all annotator scores (not max)
- Raw [0, 1] scores — NOT z-scored
- Zero for non-relevant clips

---

## 5. Training Configuration

| Parameter | Value | Notes |
|---|---|---|
| `batch_size` | 8 | |
| `accum_steps` | 2 | Effective batch = 16 |
| `lr` | **1e-4** | IMP-A: reduced from 3e-4 |
| `weight_decay` | **2e-3** | OVF-B: increased from 5e-4 |
| `grad_clip` | **0.5** | IMP-F: tighter than 1.0 |
| `n_epochs` | 100 | |
| `patience` | 25 | IMP-F: increased from 15 |
| `min_delta` | 1e-4 | |
| `warmup_frac` | 0.05 | |
| Optimizer | AdamW | |
| Scheduler | Linear warmup + cosine decay | |

### Weight Decay Groups
- **With decay (2e-3)**: All weight matrices
- **Without decay (0.0)**: `omega_*`, `bias`, `norm`, `LayerNorm`, `residual_gate`

---

## 6. Dataset

**QVHighlights** with three pre-extracted modalities:

| Modality | Dim | Source |
|---|---|---|
| Video | (T, 2304) | SlowFast features |
| Audio | (T, 2048) | PANN CNN14 features (zero-filled when unavailable) |
| CLIP visual | (T, 512) | CLIP ViT features |
| CLIP text | (S, 512) | CLIP text word tokens per query |

- `max_seq_len = 128`
- All features loaded into RAM for zero-disk-IO training
- Temporal alignment via `adaptive_avg_pool1d`

---

## 7. Regularization Summary (v4 Overfit Fixes)

| Fix | Change | Rationale |
|---|---|---|
| **OVF-A** | Dropout 0.3 → **0.4** | More aggressive stochastic regularization |
| **OVF-B** | Weight decay 5e-4 → **2e-3** | Stronger L2 penalty on weight matrices |
| **OVF-C** | **DropPath** (stochastic depth) | Drops entire residual branches; staggered 0→0.1 |
| **OVF-D** | Fusion layers 2 → **1** | Reduces over-parameterization at the bottleneck |

---

## 8. Results

### Validation Set

| Threshold | mAP | HIT@1 |
|---|---|---|
| ≥ Very Good (max ≥ 4) | **0.5394** | **0.5168** |
| ≥ Fair (max ≥ 3) | **0.6064** | **0.5997** |

### Test Set (with correct features)

| Threshold | mAP | HIT@1 |
|---|---|---|
| ≥ Very Good (max ≥ 4) | **0.5504** | **0.5243** |
| ≥ Fair (max ≥ 3) | **0.6218** | **0.6080** |

- **Best epoch**: 10
- **Best val loss**: 0.6380

### Modality Ablation (Validation, ≥ Very Good)

| Config | mAP | HIT@1 | Notes |
|---|---|---|---|
| V+T | 0.5109 | 0.4732 | Video + Text only |
| V+A+T | 0.5126 | 0.4852 | + Audio |
| V+C+T | 0.5374 | 0.5081 | + CLIP |
| **V+A+C+T** | **0.5394** | **0.5168** | Full model |

- **Audio contribution**: +0.0020 mAP
- **CLIP contribution**: +0.0268 mAP
