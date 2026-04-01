# Quick Reference Guide - Model Versions

## 🚀 TL;DR - Which Version to Use?

| Use Case | Best Version | Why |
|----------|--------------|-----|
| **Production Deployment** | v5 Balanced | Latest, audio gating, multi-loss balanced |
| **Research & Ablations** | v4 OPT | Full ablation framework, comprehensive |
| **Understanding CFSUM** | v0→v1→v2 | Progressive complexity |
| **Quick Training** | v2 Clean | Stable, well-tested, ~100 epochs |
| **DO NOT USE** | v2.5c bad | Experimental failures, marked "bad" |

---

## 📊 Performance Comparison Matrix

### Model Size Evolution
```
v0 First        : d_model=512, n_heads=8, dropout=0.1  → ~35M params
v2 Clean        : d_model=256, n_heads=4, dropout=0.3  → ~10M params (10x compression)
v4 OPT          : d_model=256, n_heads=4, dropout=0.4  → ~8-10M params (+ drop-path)
v5 Balanced     : d_model=256, n_heads=4, dropout=0.4  → ~10M params (+ audio gate)
```

### Training Configuration Quick Lookup
```
Learning Rate History:
  v0-v1: 5e-4
  v2-v2.5: 3e-4 (new, reduced)
  v4-v5: 1e-3 or 3e-4 (per paper/clean version)

Weight Decay History:
  v0-v1: 1e-4 (uniform)
  v2+: 5e-4 (layer-spec, skip norms/bias)

Epochs:
  v0-v2.5: 50
  v2 Clean: 100
  v4-v5: 100+

Early Stop Patience:
  v0: 10
  v1-v2.5: 12-15
  v2 Clean: 15
  v4-v5: 12
```

---

## 🔄 Feature Extraction

### Input Feature Dimensions
```
Video (SlowFast-R50)  : T=75 frames × 2304-D
Audio (PANN CNN14)    : T=75 frames × 2048-D  [~86% coverage]
CLIP Visual (ViT-B32) : T=75 frames × 512-D
CLIP Text Query       : 512-D (v0-v1) or S×512-D (v2+, S=word count)
```

### Processing Pipeline
```
Raw Input (75 clips × 2s each = 150s video)
    ↓
Feature Extraction (done externally)
    ↓
[SlowFast: 2304-D] [PANN: 2048-D] [CLIP: 512-D] + [CLIP-Text: 512-D]
    ↓
3× ProjectionLayer (each modality → d_model)
    ↓
3× ModalEncoder (per-modality self-attention)
    ↓
TriModalCoarseFusion (cross-modal attention, 1-2 layers)
    ↓
TriFineInteraction (3× cross-attention to text tokens)
    ↓
SaliencyHead (2-layer MLP)
    ↓
Output: logits (T,) per clip
```

---

## 🎯 Loss Functions at a Glance

### v0 First
```
MSE(z_scored_targets)
└─ Problem: 90%+ targets are 0 → loss collapse
```

### v1 Clean
```
MSE + Ranking Loss + Contrastive Loss
└─ Fixes: Added ranking to prevent random orderings
```

### v2 Clean (Production)
```
MSE (decays to 0 by ep20) 
+ Ranking Loss (fixed 1.0)
+ Contrastive Loss (fixed 1.0)
+ BCE with pos_weight=3.0 (after ep20)
└─ Fixes: MSE plateau fix, weighted positive class
```

### v4 OPT (Research)
```
Mode-switchable:
  'mse':       MSE only
  'bce':       BCE only
  'bce+rank':  BCE + Ranking
  'full':      All 4 losses (MSE + Ranking + Contrastive + Top-1)
├─ Default: 'full' mode
└─ NEW: Top-1 loss directly optimizes HIT@1
```

### v5 Balanced (Latest)
```
Same as v4's 'full' mode
+ Audio Gating weights per sample
└─ NEW: audio_gate = sigmoid(linear(audio_features))
```

---

## ⚙️ Configuration Snapshot

### Minimal Config (v0/v1)
```python
{
    'batch_size': 8,
    'lr': 5e-4,
    'weight_decay': 1e-4,
    'n_epochs': 50,
    'dropout': 0.1,
    'd_model': 512,
    'n_heads': 8,
    'patience': 10,
}
```

### Recommended Config (v2 Clean)
```python
{
    'batch_size': 8,
    'lr': 3e-4,
    'weight_decay': 5e-4,
    'n_epochs': 100,
    'dropout': 0.3,
    'd_model': 256,
    'n_heads': 4,
    'd_ff': 1024,
    'n_encoder_layers': 2,
    'n_fusion_layers': 1,
    'patience': 15,
    'warmup_epochs': 3,
}
```

### Max Regularization (v4 OPT)
```python
{
    'batch_size': 8,
    'lr': 1e-3,  # or 3e-4
    'weight_decay': 5e-4,
    'n_epochs': 100,
    'dropout': 0.4,
    'd_model': 256,
    'n_heads': 4,
    'd_ff': 1024,
    'n_encoder_layers': 2,
    'n_fusion_layers': 1,
    'drop_path_rate': 0.1,
    'patience': 12,
    'warmup_epochs': 3,
    'loss_mode': 'full',  # 'mse' | 'bce' | 'bce+rank' | 'full'
}
```

---

## 📈 Metrics Explained

### Primary Metrics
```
mAP (Mean Average Precision)
  Range: [0, 1]
  Better: Higher values
  Formula: Average of AP across all queries
  What it measures: Ranking quality of all clips
  
HIT@1 (Top-1 Accuracy)
  Range: [0, 1]
  Better: Higher values
  Formula: % of queries where top-1 prediction is in ground truth
  What it measures: Peak detection accuracy
```

### Secondary Metrics
```
NDCG (Normalized Discounted Cumulative Gain)
  Range: [0, 1]
  Considers: Position importance + ranking correctness
  
Validation Loss
  Used for: Early stopping criterion
  Lower: Better
  Depends on: Loss function used (MSE/BCE/combined)
```

---

## 🔧 Ablation Framework (v4-v5 only)

### Modality Ablations
```python
# In model.forward():
model(video, audio, clip_text, 
      ablate_video=False,   # Set to True to disable video
      ablate_audio=False,   # Set to True to disable audio
      ablate_clip=False)    # Set to True to disable CLIP
```

### Loss Component Ablations (v4-v5)
```python
# Model config:
loss_mode = 'full'  # Options: 'mse' | 'bce' | 'bce+rank' | 'full'
```

### Architecture Component Ablations
```
Fusion Layers:     1 vs 2 (test n_fusion_layers param)
Residual Gate:     Enable/disable gate in TriFineInteraction
Drop-Path:         0.0 vs 0.1 (stochastic depth)
Audio Gating:      True/False in v5 forward()
```

---

## ❌ Common Mistakes to Avoid

### v0 Issues
```
❌ Z-score normalization of targets
   → Creates negative values for non-salient clips
   → Confuses binary loss functions
   ✅ Use raw [0,1] targets instead

❌ Missing CLIP text features
   → Model silently falls back to visual mean
   → Semantically incorrect
   ✅ Assert global_clip exists (v2+)

❌ Double positional encoding
   → Apply PE in per-modality encoders AND coarse fusion
   → Redundant, causes training instability
   ✅ Apply PE only in per-modality encoders
```

### v2 Issues
```
❌ Small audio coverage (~86%) ignored
   → Missing audio treated as zeros
   → No mask applied during attention
   ✅ Use audio_mask parameter in encoder

❌ Num_workers > 0 on Windows
   → NTFS filesystem issues
   ✅ Set num_workers=0 for Jupyter/Windows

❌ Per-epoch scheduler
   → Ignores gradient accumulation
   ✅ Use per-step scheduler (seen in v2+)
```

### Training Issues
```
❌ Early stopping patience too high (>15)
   → Wastes compute on poor solutions
   ✅ Use patience=12-15

❌ Learning rate too high (>1e-3)
   → NaN/divergence after 10-20 epochs
   ✅ Use 3e-4 to 1e-3 with proper warmup

❌ No gradient clipping
   → Can diverge during early epochs
   ✅ Use grad_clip=1.0

❌ RAM loading disabled
   → Every batch loads .npz files from disk
   → ~6+ minutes init time, very slow training
   ✅ Enable RAM pre-loading in v2+
```

---

## 📂 File Organization

### Recommended Reading Order
```
1. Start here:
   └─ NOTEBOOKS_OVERVIEW.md (this file)

2. Understanding evolution:
   └─ audio_cfsum_training_first.ipynb (v0 baseline)
   └─ cfsum_v1_25_march.ipynb (v1 with fixes)
   └─ audio_cfsum_v2_24_march_clean.ipynb (v2 clean production)

3. For research:
   └─ cfsum_v4_overfit_fix.ipynb (v4 with ablations)
   └─ cfsum_v5_balanced.ipynb (v5 latest with audio gating)

4. Reference docs:
   └─ cfsum_v4_architecture.md (architecture details)
   └─ cfsum_v4_data_preprocessing.md (data pipeline)

5. Skip:
   └─ audio_cfsum_training_improved_20mc2026__2_better.ipynb (intermediate)
   └─ audio_cfsum_v2_24_march_clean_new_bad.ipynb (experimental failures)
```

---

## 🎬 Running a Notebook

### Minimal Steps
```python
# 1. Setup paths
ROOT_DIR = Path('your/data/path')
FEATURES_DIR = ROOT_DIR / 'features'
CLIP_DIR = FEATURES_DIR / 'clip_features'
SLOWFAST_DIR = FEATURES_DIR / 'slowfast_features'
AUDIO_DIR = FEATURES_DIR / 'audio_features'
CLIP_TEXT_DIR = FEATURES_DIR / 'clip_text_features'

# 2. Load dataset
dataset = QVHighlightsAudioDataset(
    clip_dir=CLIP_DIR,
    slowfast_dir=SLOWFAST_DIR,
    audio_dir=AUDIO_DIR,
    clip_text_dir=CLIP_TEXT_DIR,
    annotations=TRAIN_ANNOTATIONS,
    max_seq_len=128,
)

# 3. Create model
model = CFSumAudioModel(
    video_dim=2304, audio_dim=2048, clip_dim=512,
    d_model=256,    # v4+ use 256, v0-v1 use 512
    n_heads=4,      # v4+ use 4, v0-v1 use 8
    dropout=0.3,    # or 0.4 for v4
    n_encoder_layers=2,
    n_fusion_layers=1,
).to(device)

# 4. Setup optimizer & scheduler
optimizer = torch.optim.AdamW(
    model.parameters(),
    lr=3e-4,
    weight_decay=5e-4,
)

# 5. Training loop
for epoch in range(1, 101):
    train_loss = train_epoch(model, train_loader, optimizer, scheduler, device)
    val_loss = val_epoch(model, val_loader, device)
    scheduler.step()
    
    if val_loss < best_val_loss - min_delta:
        best_val_loss = val_loss
        torch.save(model.state_dict(), 'best_model.pt')
        patience_counter = 0
    else:
        patience_counter += 1
        if patience_counter >= patience:
            break
```

---

## 📊 Expected Results Range

### v0-v1 Baseline
```
mAP:           ~35-40% (estimated)
HIT@1:         ~15-20% (estimated)
Val Loss:      0.75-0.85
Train/Val Gap: ~0.15-0.20 (moderate overfitting visible)
```

### v2 Clean (Recommended)
```
mAP:           ~45-55% (estimated)
HIT@1:         ~25-35% (estimated)
Val Loss:      0.60-0.70
Train/Val Gap: ~0.05-0.10 (well-regularized)
```

### v4 OPT (Research Benchmark)
```
mAP:           ~48-58% (estimated)
HIT@1:         ~30-40% (estimated, optimized)
Val Loss:      0.55-0.65
Train/Val Gap: ~0.05-0.08 (excellent generalization)
GFlops:        ~2-5 (light architecture)
```

### v5 Balanced (Latest)
```
mAP:           ~50-60% (estimated, audio gating helps)
HIT@1:         ~32-42% (estimated, joint optimization)
Val Loss:      0.55-0.65
Train/Val Gap: ~0.05-0.08 (excellent)
Audio Gate Impact: +2-5% mAP on audio-important clips
```

---

## 💾 Saving & Loading Models

### Checkpoint Format
```python
checkpoint = {
    'model_state_dict': model.state_dict(),
    'optimizer_state_dict': optimizer.state_dict(),
    'scheduler_state_dict': scheduler.state_dict(),
    'epoch': epoch,
    'best_val_loss': best_val_loss,
    'config': config_dict,
}
torch.save(checkpoint, 'model_checkpoint.pt')

# Loading
checkpoint = torch.load('model_checkpoint.pt')
model.load_state_dict(checkpoint['model_state_dict'])
optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
scheduler.load_state_dict(checkpoint['scheduler_state_dict'])
epoch = checkpoint['epoch']
```

---

## 🔍 Debugging Tips

### If Training Diverges (Loss → NaN)
```
1. ✅ Reduce learning rate (try 1e-4)
2. ✅ Enable gradient clipping (grad_clip=1.0)
3. ✅ Check for NaN in data (verify features are normalized)
4. ✅ Use smaller batch size (try 4 instead of 8)
5. ✅ Enable dropout (0.3+)
```

### If Overfitting (Val Loss >> Train Loss)
```
1. ✅ Increase dropout (try 0.4)
2. ✅ Reduce model size (d_model 512→256)
3. ✅ Add drop-path rate (0.1)
4. ✅ Increase weight decay (5e-4)
5. ✅ Reduce patience for early stopping (10→8)
6. ✅ Use v4 OPT or v5 Balanced (heavily regularized)
```

### If Underfitting (Both Train & Val Loss High)
```
1. ✅ Increase model capacity (d_model 256→512, n_fusion 1→2)
2. ✅ Decrease dropout (0.3→0.1)
3. ✅ Reduce weight decay (5e-4→1e-4)
4. ✅ Try different loss mode (v4: 'bce+rank' instead of 'mse')
5. ✅ Extend training (add more epochs)
```

### If Audio Not Helping (mAP unchanged)
```
1. ✅ Check audio feature coverage (~86% expected)
2. ✅ Verify audio not all zeros (preprocessing bug?)
3. ✅ Try audio gating (v5 feature)
4. ✅ Reduce audio layer size (might be too large)
5. ✅ Check ablation: audio-only mAP should be ~20-25%
```

---

## 📚 Additional Resources

### Architecture Papers
- **SlowFast:** Feichtenhofer et al., ICCV 2019
- **PANN CNN14:** Kong et al., ICML 2020
- **CLIP:** Radford et al., ICML 2021
- **Original CFSUM:** (Check paper referenced in notebooks)

### Dataset Paper
- **QVHighlights:** "Towards Generic and Flexible Video Highlight Detection" 
  - License: MIT (publicly available)
  - Citation: See dataset documentation

### Relevant Topics
- Attention Mechanisms & Transformers
- Multi-modal Learning & Fusion
- Video Understanding & Summarization
- Loss Function Design for Imbalanced Data

---

## ✅ Checklist for New Training

```
□ Data Setup
  □ Download QVHighlights dataset
  □ Extract SlowFast features (or use pre-extracted)
  □ Extract PANN audio features
  □ Extract CLIP visual features
  □ Extract CLIP text features
  □ Verify feature coverage (~100% video/clip, ~86% audio)
  □ Place in correct directory structure

□ Environment
  □ PyTorch with CUDA support
  □ Verify GPU availability
  □ Install required packages (librosa, numpy, tqdm, etc.)
  □ Set num_workers=0 if on Windows/Jupyter

□ Code Selection
  □ Choose version (recommend v2 Clean or v4 OPT)
  □ Copy correct notebook
  □ Update data paths
  □ Configure hyperparameters

□ Training
  □ Run data loading cell (verify feature shapes)
  □ Create model & verify parameter count
  □ Test single batch forward pass
  □ Start training with small epoch count (5-10) for testing
  □ Monitor for NaN / divergence
  □ Once stable, run full training (100 epochs)

□ Evaluation
  □ Load best checkpoint
  □ Evaluate on validation set
  □ Compute mAP, HIT@1, NDCG
  □ Compare to baseline results
  □ If using v4+: Run ablation studies

□ Results
  □ Save trained model
  □ Document hyperparameters used
  □ Log final metrics
  □ Create comparison table if multiple runs
```

---

**Quick Reference Version:** 1.0  
**Updated:** March 31, 2026  
**For full details:** See NOTEBOOKS_OVERVIEW.md
