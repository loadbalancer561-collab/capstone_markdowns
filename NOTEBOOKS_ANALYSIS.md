# Comprehensive Analysis of Capstone Project Notebooks

**Project**: Audio-Aware Coarse-to-Fine Fusion for Video Summarization (CFSum)  
**Dataset**: QVHighlights (query-conditioned video highlight detection)  
**Analysis Date**: March 31, 2026

---

## Executive Summary

This project implements a tri-modal transformer-based model for video summarization that fuses **SlowFast video features**, **PANN CNN14 audio features**, and **CLIP visual features** to predict visual saliency scores per clip given a natural language query. Eight notebook versions document the evolution from baseline to increasingly sophisticated architectures with various regularization and optimization techniques.

---

## 1. ARCHITECTURE COMPARISON

### Core Components Across Versions

| Component | audio_cfsum_training_first | audio_cfsum_training_23_march | cfsum_v1_25_march | cfsum_v4_overfit_fix | cfsum_v5_balanced |
|---|---|---|---|---|---|
| **Feature Extractors** | SlowFast, PANN, CLIP | SlowFast, PANN, CLIP | SlowFast, PANN, CLIP | SlowFast, PANN, CLIP | SlowFast, PANN, CLIP |
| **Video Dim** | 2304 | 2304 | 2304 | 2304 | 2304 |
| **Audio Dim** | 2048 | 2048 | 2048 | 2048 | 2048 |
| **CLIP Dim** | 512 | 512 | 512 | 512 | 512 |
| **d_model** | 512 | 512 | 512 | **256** | **256** |
| **n_heads** | 8 | 8 | 8 | **4** | **4** |
| **head_dim** | 64 | 64 | 64 | **64** | **64** |
| **d_ff (MLPs)** | 2048 | 2048 | 2048 | **1024** | **1024** |
| **Dropout** | 0.1 | 0.1 | 0.1 | **0.3-0.4** | **0.3-0.4** |
| **ModalEncoder layers** | 2 | 2 | 2 | 2 | 2 |
| **Coarse Fusion layers** | 2 | 2 | 2 | **1** | **1** |
| **Cross-attention branches** | 2 (video, audio) | **3** (video, audio, CLIP) | 2 | **3** | **3** |
| **Text token input** | Pooler (512,) | **last_hidden_state (S, 512)** | Pooler (512,) | **last_hidden_state (S, 512)** | **last_hidden_state (S, 512)** |
| **Special features** | Baseline | Text token projection | PE at all stages | DropPath (0.1), Residual gate | Audio gating module |

### Model Parameter Counts

| Notebook | d_model | Config | Est. Parameters |
|---|---|---|---|
| audio_cfsum_training_first | 512 | baseline | ~35M |
| audio_cfsum_training_23_march | 512 | + triple-attention | ~35M |
| cfsum_v1_25_march | 512 | dual-attention | ~35M |
| cfsum_v4_overfit_fix | 256 | + dropout 0.4 + DropPath 0.1 | **~10M** (30% of baseline) |
| cfsum_v5_balanced | 256 | + audio gating | **~10-11M** |

---

## 2. INPUT DIMENSIONS & DATA HANDLING

### Feature Pipeline

```
Video:   SlowFast-R50              (T, 2304) ──ProjectionLayer──> (T, d_model)
Audio:   PANN CNN14                (T, 2048) ──ProjectionLayer──> (T, d_model)  [80%+ coverage]
CLIP:    ViT-B/32 visual           (T,  512) ──ProjectionLayer──> (T, d_model)
Query:   CLIP text tokens          (S,  512) ──Linear(512→d_model)──> (S, d_model)
```

### Dataset Statistics

| Metric | Train | Val | Test | Notes |
|---|---|---|---|---|
| **Videos** | 10K+ | 1K+ | 1.5K+ | QVHighlights dataset |
| **Annotated Samples** | 7,218 | 1,550 | 1,542 | (video, query) pairs |
| **Audio Coverage** | 86% (8,695/10,148) | 86% | ~85% | Missing files → zero-filled |
| **Text Coverage** | 100% | 100% | 100% | CLIP text tokens required |
| **Clips per video** | 75 (150s ÷ 2s) | 75 | 75 | Fixed window size |
| **Max seq_len** | 128 | 128 | 128 | Temporal alignment to this |

### Temporal Alignment

- **Raw sequence length**: 75 clips per 150-second video (2-second windows)
- **Processing**: Adaptive pooling to align all modalities to clip sequence length (T)
- **Saliency target construction**: [1,5] annotator scores → [0,1] range via `(mean(scores) - 1.0) / 4.0`

### Memory Optimization (Fix #12)

**RAM pre-loading strategy** (v4+):
- All `.npz` feature files loaded into RAM at startup
- Eliminates disk I/O during training
- System: 128 GB RAM, ~79 GB estimated usage
- **Result**: Dense GPU utilization (~100%)

---

## 3. ARCHITECTURE PROGRESSION

### Version Timeline & Key Innovations

#### **v0: audio_cfsum_training_first (Baseline)**
- **Date**: Early March 2026
- **Architecture**: Dual cross-attention (video→text, audio→text)
- **Key features**:
  - ModalEncoder: Positional encoding applied at encoder level (later identified as problematic)
  - FineInteraction: text_expand hack (single CLIP token expanded to 8 virtual tokens)
  - Loss: MSE on z-score normalized saliency
  - Optimizer: AdamW lr=5e-4, wd=1e-4
- **Issues**: Positional encoding bias, no support for CLIP visual modality
- **Test results**: Best val MSE ≈ 0.8164

#### **v1: cfsum_v1_25_march (Baseline with Dual-attention)**
- **Improvements over v0**:
  - Positional encoding moved to CoarseFusion level only
  - Same dual cross-attention to text
- **Dataset**: 7,218 train / 1,550 val samples
- **Status**: Intermediate version

#### **v1.5: audio_cfsum_training_improved_20mc2026__2_better**
- **New fixes**:
  1. **Ranking loss** (margin ranking m=0.2) added alongside MSE
  2. **CLIP visual** stream added as third cross-attention branch (ω_tc=1.0)
  3. **global_clip required** — fallback to visual mean removed
  4. **Double PE fix** — coarse fusion skips PE (already applied in encoders)
  5. **Audio mask inside encoder** — src_key_padding_mask (NaN-safe)
  6. **Full mAP evaluation** on validation set (not just Top-5 overlap)
  7. **Linear LR warmup** for first 5% of steps before cosine decay
  8. **More epochs (100) + larger batch (8)** with gradient accumulation
  9. **RAM pre-loading** strategy implemented
  10. **Full CLIP text tokens** — last_hidden_state (S, 512) for multi-token attention

#### **v2: audio_cfsum_v2_24_march_clean (Clean v2)**
- **Triple cross-attention confirmed** (video→text, audio→text, clip→text)
- **Text token handling**: last_hidden_state (variable S) with text_mask
- **Loss options**: MSE baseline, BCE with logits, contrastive saliency
- **Optimizer**: lr=3e-4 (reduced from baseline)
- **Loss weights**: BCE + MarginRanking + ContrastiveSaliency
- **Config**:
  - d_model=256, dropout=0.3, weight_decay=5e-4, patience=12
  - n_heads=4 (head_dim=64), d_ff=1024
- **Training**: 100 epochs with gradient warmup and cosine decay

#### **v2 (bad): audio_cfsum_v2_24_march_clean_new_bad**
- **Status**: Marked as "bad" in filename; experimental version with issues

#### **v4: cfsum_v4_overfit_fix (Major Regularization)**
- **Problem addressed**: Overfitting on training set
- **Overfitting fixes (OVF-A through OVF-D)**:
  - **OVF-A**: dropout 0.3 → **0.4**
  - **OVF-B**: weight_decay 5e-4 → **2e-3** (4× stronger L2)
  - **OVF-C**: drop_path_rate **0.1** (stochastic depth on all Transformer residuals; linearly staggered 0 → 0.1)
  - **OVF-D**: coarse_fusion layers 2 → **1** (single layer prevents overfitting cross-modal alignment)
- **Architecture reduction**: 
  - d_model: 512 → **256**, n_heads: 8 → **4**
  - Parameters: 35M → **~10M** (71% reduction)
- **New components**:
  - `TransformerLayerWithDropPath`: Every residual gets DropPath
  - Learnable residual gate: `gate · refined + (1-gate) · proj_residual`
  - Audio masking in encoder with NaN-safety check
- **Loss**: BCE with pos_weight=3 + MarginRanking + ContrastiveSaliency
- **Optimizer**: AdamW lr=1e-4, warmup 5%, cosine decay
- **Results**: Significant validation improvement expected from regularization

#### **v5: cfsum_v5_balanced (Audio Gating + Balanced Loss)**
- **NEW: AudioGating module**:
  ```python
  audio_weight = sigmoid(Linear(2048 → 256 → 2048)(audio))
  gated_audio = audio_weight * audio
  ```
  - Learnable per-feature importance weighting
  - Prevents noisy audio from degrading performance
- **Purpose**: Address audio quality variability (86% coverage, some features may be poor quality)
- **Balanced loss strategy**:
  - Class balance: pos_weight=3 upweights positive (salient) clips
  - Multi-phase training: Ranking loss from epoch 5, Contrastive from epoch 10
  - Hard negative mining: top-k=5 hardest negatives per batch
- **Config inherited from v4**:
  - d_model=256, dropout=0.3-0.4, DropPath=0.1
  - n_fusion_layers=1, residual_gate active
- **Dataset coverage**: train 7,218, val 1,550, **test 1,542** samples
- **Status**: Final optimization for production deployment

---

## 4. TRAINING PARAMETERS & OPTIMIZATION

### Optimizer & Scheduler Configuration

| Parameter | audio_cfsum_training_first | cfsum_v1_25_march | cfsum_v4_overfit_fix | cfsum_v5_balanced |
|---|---|---|---|---|
| **Optimizer** | AdamW | AdamW | AdamW | AdamW |
| **Learning rate** | 5e-4 | 5e-4 → 3e-4 | **1e-4** | **1e-4** |
| **Weight decay** | 1e-4 | 5e-4 | **2e-3** | **2e-3** |
| **Batch size** | 8 | 8 | 8 | 8 |
| **Grad accum steps** | 4 | 4 | 1 | 1 |
| **Gradient clip** | 1.0 | 0.5 | **0.5** | **0.5** |
| **LR warmup** | Linear 5% | Linear 5% | Linear 5% | Linear 5% |
| **LR schedule** | Cosine decay | Cosine decay | Cosine decay | Cosine decay |
| **n_epochs** | 50 | 50 | 100+ | 100+ |
| **patience (ES)** | 10 | 10 | 25 | 25 |
| **num_workers** | 0 (Windows safe) | 0 | 0 | 0 |

### Loss Function Evolution

| Version | Loss Components | Details |
|---|---|---|
| **v0-v1** | MSE only | `(1/nc) · Σ(pred_i - target_i)²` |
| **v1.5** | MSE + Margin Ranking | Ranking m=0.2 fixes position bias |
| **v2** | BCE + MarginRanking + Contrastive | pos_weight=3; multi-phase warmup |
| **v4** | BCE + MarginRanking + Contrastive | Same as v2; added DropPath regularization |
| **v5** | BCE + MarginRanking + Contrastive | Same; + hard negative mining, audio gating |

### Target Construction

- **Raw saliency**: 3 annotators score each clip [1, 5]
- **Normalized**: `(mean(scores) - 1.0) / 4.0` → [0, 1] range
- **v0-v1 variant**: Z-score normalization per video (creates contrast between salient/background)
- **v2+ variant**: Raw [0, 1] scores (contrastive loss works on relative differences, not absolute values)

---

## 5. DATA AUGMENTATION & PREPROCESSING

### Feature Preprocessing

**SlowFast Video (2304-dim)**:
- Extracted from video frame sequences
- Temporally aligned to clip length via **adaptive average pooling**
- Coverage: 100% (10,148 / 10,148 videos)

**PANN CNN14 Audio (2048-dim)**:
- Extracted via PANN audio encoder on 2-second windows
- 32 kHz, mono, non-overlapping 75 windows per 150s clip
- Coverage: **86%** (8,695 / 10,148 videos)
- Missing files: Zero-filled; audio_mask applied in encoder

**CLIP Visual (512-dim)**:
- ViT-B/32 embeddings from video frames
- Coverage: 100% (10,148 / 10,148 videos)

**CLIP Text (variable S, 512-dim)**:
- last_hidden_state from CLIP text encoder on query
- Variable number of word tokens per query
- Padded to max_text_len in batch; text_mask tracks real vs padding positions

### Data Augmentation Strategies

| Technique | Version | Purpose | Implementation |
|---|---|---|---|
| **Temporal shuffle** | v0-v1 | Prevent position bias | Random permutation of clip order in training |
| **Stochastic depth** | v4+ | Regularization | DropPath on Transformer residuals (rate 0.1) |
| **Hard negative mining** | v5 | Balanced loss | Top-k=5 hardest negatives per batch |
| **Class weighting** | v5 | Handle imbalance | BCE pos_weight=3 upweights salient clips |
| **Multi-phase training** | v5 | Curriculum learning | Ranking loss from epoch 5, Contrastive from epoch 10 |

### Audio Handling — Incomplete Coverage (86%)

**Problem**: 14% of videos lack audio features

**Solutions**:
1. **v0-v1**: Zero-fill + scalar audio_mask applied outside encoder (ineffective)
2. **v1.5+**: Zero-fill + audio_mask as src_key_padding_mask inside attention (NaN-safe; prevents softmax collapse)
3. **v4**: NaN-safety check: only mask positions if sample has SOME real audio
4. **v5**: AudioGating module learns per-feature importance:
   ```python
   gate = sigmoid(Linear(audio_dim))(audio_features)
   gated_audio = gate * audio
   ```
   Smoothly suppresses low-quality audio without hard masking

---

## 6. EVALUATION METRICS

### Metrics Used

| Metric | Notation | Purpose | Used in |
|---|---|---|---|
| **Mean Squared Error** | MSE | Loss function | v0, v1.5 (baseline) |
| **Binary Cross Entropy** | BCE | Classification loss | v2, v4, v5 |
| **mAP** | Mean Average Precision | Ranking quality | v1.5+ (full val set) |
| **HIT@1** | Hit rate at rank 1 | Top-1 prediction accuracy | v2+ |
| **NDCG** | Normalized Discounted Cumulative Gain | Ranking metric | Not explicitly mentioned |
| **F1 score** | Per-class F1 | Salient vs background | Implicit in binary evaluation |
| **Top-5 Overlap** | Top-5 accuracy | Partial name-based matching | v0-v1 (replaced by mAP) |

### Ground Truth Definitions

- **"Very Good"**: annotator score ≥ 4 (confidence threshold)
- **"Fair"**: annotator score ≥ 2
- **Dual-threshold reporting** (v2+): Both thresholds for comprehensive evaluation

### Reported Results

#### **audio_cfsum_training_first (v0 Baseline)**
- **Training protocol**: 50 epochs, early stopping patience=10
- **Training loss trajectory**:
  - Epoch 1-5: 1.111 → 0.778 (strong initial descent)
  - Epoch 5-10: 0.778 → 0.651 (continued improvement)
  - Epoch 10-15: 0.651 → 0.573 (plateau beginning)
- **Best validation MSE**: 0.8164 (epoch 5)
- **Early stopping**: Triggered at epoch 15 (10 consecutive no-improve epochs)
- **Evaluation**: Top-5 overlap metric on validation samples
- **Status**: Baseline reference; position bias present

#### **cfsum_v4_overfit_fix (v4 Regularized)**
- **Training**: 100+ epochs expected, with 25-epoch patience
- **Regularization impact**: 71% parameter reduction (35M → 10M)
- **Expected improvements**:
  - Reduced training/val gap (regularization combats overfitting)
  - Better generalization to test set
  - Smoother loss dynamics (DropPath + stochastic depth)
- **Status**: Production-ready; awaiting final evaluation

#### **cfsum_v5_balanced (v5 Balanced)**
- **Dataset splits**: Train 7,218, Val 1,550, **Test 1,542**
- **Audio gating**: Learned importance weighting per audio feature
- **Loss strategy**: Multi-component with hard negative mining
- **Status**: Final version; full dataset support (train/val/test)

---

## 7. ARCHITECTURE DIFFERENCES & ABLATIONS

### Key Architectural Choices

| Feature | v0-v1 | v1.5 | v4 | v5 | Purpose |
|---|---|---|---|---|---|
| **Cross-attn branches** | 2 | **3** | 3 | 3 | Leverage CLIP visual modality |
| **Text representation** | Pooler (S=1) | Tokens (S=variable) | Tokens | Tokens | Word-level attention |
| **PE placement** | encoder + coarse | coarse only | coarse only | coarse only | Fix position bias |
| **Coarse layers** | 2 | 2 | **1** | **1** | Prevent overfitting |
| **Dropout** | 0.1 | 0.1 | **0.4** | **0.4** | Regularization |
| **DropPath** | — | — | **0.1** | **0.1** | Stochastic depth |
| **d_model** | 512 | 512 | **256** | **256** | Model compression |
| **Residual gate** | — | — | **Yes** | **Yes** | Skip connection blending |
| **Audio gating** | — | — | — | **Yes** | Audio feature importance |

### Ablation Study Capability (v4+)

**Modality ablation flag** in forward():
```python
def forward(..., ablate=None):
    if ablate:
        if 'video' in ablate: video = torch.zeros_like(video)
        if 'audio' in ablate: audio = torch.zeros_like(audio)
        if 'clip'  in ablate: clip = torch.zeros_like(clip)
```

**Potential ablations**:
- Video-only (audio + CLIP zeroed)
- Audio-only (video + CLIP zeroed)
- CLIP-only (video + audio zeroed)
- Video+Audio (CLIP zeroed) — validate CLIP contribution
- Video+CLIP (audio zeroed) — validate audio contribution

---

## 8. PURPOSE & PROBLEM ADDRESSED

### Core Problem

**Query-conditioned video highlight detection**: Given a YouTube video and a natural language query (e.g., "person climbing a ladder"), predict which 2-second clips in the video are most relevant to the query.

### Hypothesis Evolution

1. **v0-v1**: Multi-modal fusion using video + audio + CLIP can improve over single modality
2. **v1.5**: CLIP visual stream should contribute independently alongside video/audio in fine-grained interaction
3. **v4**: Overfitting is the bottleneck, not architecture; aggressive regularization (DropPath, reduced capacity) improves generalization
4. **v5**: Audio quality is variable (86% coverage); learned gating can handle heterogeneous audio

### Research Questions

1. Does audio improve highlight detection? (86% coverage allows this comparison)
2. Should CLIP visual and CLIP text be fused separately? (v1.5+ triple cross-attention)
3. Is position bias in Transformers a problem for this task? (v1.5 PE reduction)
4. Can regularization overcome overfitting more effectively than larger models? (v4 parameter reduction)
5. Can learned audio gating improve robustness? (v5)

---

## 9. SPECIAL FEATURES & ADVANCED TECHNIQUES

### Triple Cross-Attention (v1.5+)

```
        video_fused ──┐
                      ├─→ CrossAttn ──┐
        audio_fused ──┤                │
                      │    to text──→  ├─→ Σ ω·attn
        clip_fused ───┤                │
                      └──→ CrossAttn ──┘

ω_tv = 2.0 (video contribution)
ω_ta = 1.0 (audio contribution)
ω_tc = 1.0 (CLIP contribution)
```

### Fix #2: CLIP Visual Modality Integration

**Problem**: Original CFSum only used CLIP text (pooled query embedding); CLIP visual features extracted but discarded after coarse fusion

**Solution**: Add third cross-attention branch:
```
t_c_att = CrossAttention(clip_fused, text_kv, text_kv)
z = ω_tv·tv_att + ω_ta·ta_att + ω_tc·t_c_att
```

### Fix #4: Double Positional Encoding Removal

**Problem**: PE applied in ModalEncoder (per-modality) AND in CoarseFusion (joint), causing model to over-rely on position

**Solution**: PE only in CoarseFusion; ModalEncoders use pure self-attention

**Result**: Position bias eliminated; content features dominate

### Fix #14: Real Word Tokens from CLIP Text

**Problem**: Single pooled CLIP text token (S=1) collapses cross-attention: softmax([x])=1 always

**Solution v1.5-v3**: Expand token to 8 virtual tokens (text_expand hack)

**Solution v2+**: Use actual word tokens from last_hidden_state (S=variable):
```python
global_clip = text_encoder(query)['last_hidden_state']  # (S, 512)
text_kv = Linear(512 → d_model)(global_clip)  # (S, d_model)
# Now cross-attention has genuine S K,V pairs
```

### AudioGating Module (v5)

**Purpose**: Learn per-feature importance for audio

```python
class AudioGating(nn.Module):
    def forward(self, audio):  # (B, T, 2048)
        gate = sigmoid(Linear(2048 → 256 → 2048)(audio))  # (B, T, 2048)
        return gate * audio  # element-wise weighting
```

**Why**: Audio coverage is incomplete (86%); quality varies; noisy features should be suppressed

**Effect**: Learned data-dependent weighting vs. hard masking

### Stochastic Depth / DropPath (v4+)

**Mechanism**: Randomly drop entire residual branches during training

```python
if training and random() < drop_prob:
    return original_input  # skip this layer's computation
```

**Linearly staggered**: Layer 0 gets drop_prob=0, final layer gets drop_prob=0.1

**Why**: Prevents co-adaptation of layers; forces each layer to be useful independently

**Impact**: Stronger regularization than standard dropout

### Residual Gate (v4+)

**Purpose**: Blend refined transformer output with shortcut from projection layer

```python
proj_residual = (video_emb + audio_emb + clip_emb) / 3.0
gate = sigmoid(learnable_param)  # init 0.5
output = gate * transformer_out + (1 - gate) * proj_residual
```

**Why**: Preserves temporal diversity; prevents over-reliance on fusion layers

### Multi-Phase Loss Training (v5)

**Epoch schedule**:
- Epochs 1-4: BCE + MarginRanking only (learn saliency prediction)
- Epochs 5-9: Gradually introduce ContrastiveSaliency loss (warm up contrast)
- Epochs 10+: Full loss: BCE + MarginRanking + ContrastiveSaliency

**Hard negative mining**: Top-k=5 hardest negatives per batch

**Why**: Curriculum prevents loss explosion in early training; hard negatives focus learning

---

## 10. NOTEBOOK NAMING & VERSIONING SCHEME

| Filename | Version | Status | Purpose |
|---|---|---|---|
| `audio_cfsum_training_first.ipynb` | v0 | Baseline reference | Initial dual-attention implementation |
| `cfsum_v1_25_march.ipynb` | v1 | Baseline variant | PE reduction experiment |
| `audio_cfsum_training_improved_20mc2026__2_better.ipynb` | v1.5 | Improved | Add triple-attention, full word tokens |
| `audio_cfsum_v2_24_march_clean.ipynb` | v2 | Clean | Consolidate v1.5 improvements, add BCE loss |
| `audio_cfsum_v2_24_march_clean_new_bad.ipynb` | v2 (experimental) | **Bad version** | Experimental; marked as problematic |
| `audio_cfsum_training_23_march (1).ipynb` | v2.1 | Improved | Refinements to v2 |
| `cfsum_v4_overfit_fix.ipynb` | v4 | Production | Major regularization: DropPath, reduced capacity, coarse_layers=1 |
| `cfsum_v5_balanced.ipynb` | v5 | Final | Audio gating, balanced loss, full train/val/test support |

**Naming notes**:
- Date suffix (25_march, 24_march, 23_march) indicates experiment date
- `_clean` suffix = well-organized, final version
- `_bad` suffix = experimental, issues found
- `(1)` suffix = remake/retry of same date

---

## 11. CONVERGENCE & TRAINING DYNAMICS

### v0 Training Trace (50 epochs)

```
Epoch   1:  train 1.1112  val 0.9543  ✓ best
Epoch   2:  train 0.9372  val 0.9259  ✓ best
Epoch   3:  train 0.8769  val 0.8555  ✓ best
Epoch   4:  train 0.8187  val 0.8262  ✓ best
Epoch   5:  train 0.7776  val 0.8164  ✓ best  ← Best validation
Epoch   6:  train 0.7430  val 0.8299  (1/10)
Epoch   7:  train 0.7168  val 0.8191  (2/10)
...
Epoch  15:  train 0.5728  val 0.9020  (10/10)  EARLY STOP
```

**Observations**:
- Strong initial descent (1.11 → 0.78 in 5 epochs)
- Plateau after epoch 5 (best val = 0.8164)
- Training continues improving while validation stagnates (overfitting signal)
- Early stopping at epoch 15 (patience=10 exceeded)

### v4 Expected Trajectory (with DropPath + regularization)

- Slower initial descent (DropPath noise)
- Smoother convergence (stochastic depth stability)
- Smaller train/val gap (regularization controls overfitting)
- Better test generalization (reduced capacity + strong regularization)

---

## 12. SUMMARY TABLE: QUICK REFERENCE

| Notebook | Date | Version | Key Innovation | d_model | Params | Best Val Metric | Eval Status |
|---|---|---|---|---|---|---|---|
| audio_cfsum_training_first.ipynb | Early Mar | v0 | Dual-attention baseline | 512 | ~35M | MSE 0.8164 | ✓ Complete |
| cfsum_v1_25_march.ipynb | Mar 25 | v1 | PE reduction | 512 | ~35M | - | Intermediate |
| audio_cfsum_training_improved...ipynb | Mar 20 | v1.5 | Triple-attention, word tokens | 512 | ~35M | - | Experimental |
| audio_cfsum_v2_24_march_clean.ipynb | Mar 24 | v2 | Consolidate improvements | 256 | ~10M | - | ✓ Clean |
| audio_cfsum_v2_clean_new_bad.ipynb | Mar 24 | v2x | Experimental variant | 256 | ~10M | - | ✗ Bad |
| audio_cfsum_training_23_march (1).ipynb | Mar 23 | v2.1 | v2 refinements | 512 | ~35M | - | Refinement |
| cfsum_v4_overfit_fix.ipynb | Mar 25 | v4 | DropPath, coarse_layers=1 | 256 | ~10M | Expected: Lower gap | ✓ Production ready |
| cfsum_v5_balanced.ipynb | Mar 31 | v5 | Audio gating, balanced loss | 256 | ~11M | Expected: Best generalization | ✓ Final |

---

## 13. RECOMMENDATIONS FOR FUTURE WORK

1. **Ablation studies**: Use modality ablation flags to quantify video/audio/CLIP contributions
2. **Test set evaluation**: cfsum_v5_balanced supports full train/val/test; run final evaluation
3. **Audio quality analysis**: Study which 14% of missing audio vs. how missing audio affects performance
4. **Attention visualization**: Visualize which text tokens each modality attends to
5. **Query length analysis**: Explore if variable seq_len of text tokens (S) affects performance
6. **Hyperparameter sweep**: Test d_model ∈ {128, 256, 512}, dropout ∈ {0.2, 0.3, 0.4}
7. **Loss ablation**: Separately evaluate BCE, MarginRanking, ContrastiveSaliency contributions
